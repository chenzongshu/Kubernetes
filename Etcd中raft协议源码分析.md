以etcd或者docker中的raft协议为列子,来解析raft协议的实际落地


# server通信

## 概述

server之前的消息传递并不是简单的request-response模型，而是读写分离模型，

即每两个server之间会建立两条链路，对于每一个server来说，一条链路专门用来发送数据，另一条链路专门用来接收数据.

在代码实现中，通过streamWriter发送数据，通过streamReader接收数据。即通过streamReader接收数据接收到数据后会直接响应，在处理完数据后通过streamWriter将响应发送到对端

对于每个server来说，不管是leader、candicate还是follower，都会维持一个peers数组，每个peer对应集群中的一个server，负责处理server之间的一些数据交互。

当server需要向其他server发送数据时，只需要找到其他server对应的peer，然后向peer的streamWriter的msgc通道发送数据即可，streamWriter会监听msgc通道的数据并发送到对端server；

而streamReader会在一个goroutine中循环读取对端发送来的数据，一旦接收到数据，就发送到peer的p.propc或p.recvc通道，而peer会监听这两个通道的事件，写入到node的n.propc或n.recvc通道，node只需要监听这两个通道的数据并处理即可。这就是在etcd的raft实现中server间数据交互的流程。

## 启动监听

对于每个server，都会创建一个raftNode，并且启动一个goroutine，执行raftNode的serveRaft方法

```
func (rc *raftNode) serveRaft() {
......

	err = (&http.Server{Handler: rc.transport.Handler()}).Serve(ln)
}
```

这个方法主要是建立一个httpserver，监听其他server的连接，处理函数为rc.transport.Handler()

```
func (t *Transport) Handler() http.Handler {
	pipelineHandler := newPipelineHandler(t, t.Raft, t.ClusterID)
	streamHandler := newStreamHandler(t, t, t.Raft, t.ID, t.ClusterID)
	snapHandler := newSnapshotHandler(t, t.Raft, t.Snapshotter, t.ClusterID)
	mux := http.NewServeMux()
	mux.Handle(RaftPrefix, pipelineHandler)
	mux.Handle(RaftStreamPrefix+"/", streamHandler)
	mux.Handle(RaftSnapshotPrefix, snapHandler)
	mux.Handle(ProbingPrefix, probing.NewHandler())
	return mux
}
```

## 处理服务

上节提到的streamHandler，这个handler用于处理server之间的心跳、投票、附加日志等请求的发送，该handler的ServeHTTP代码核心为

```
func (h *streamHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
......
	p.attachOutgoingConn(conn)
	<-c.closeNotify()
```

一旦接收到对端的连接，则把该连接attach到自己encoder的writer中，这样自己encoder和对端decoder就能协同工作了，

对于每个节点，会主动去连接其他节点，连接成功后便通过自己的decoder循环读取该连接的数据，该节点通过该decoder读取其他节点发来的数据；

当某节点收到其他节点连接请求并连接成功后便把该连接attach到该节点的encoder，该节点通过该encoder向其他节点发送数据；

看其函数实现

```
func (p *peer) attachOutgoingConn(conn *outgoingConn) {
	var ok bool
	switch conn.t {
	case streamTypeMsgAppV2:
		ok = p.msgAppV2Writer.attach(conn)
	case streamTypeMessage:
		ok = p.writer.attach(conn)
	default:
		plog.Panicf("unhandled stream type %s", conn.t)
	}
	if !ok {
		conn.Close()
	}
}
```

其中`msgAppV2Writer`和`writer`都是一样的结构体`streamWriter`

```
func (cw *streamWriter) attach(conn *outgoingConn) bool {
	select {
	case cw.connc <- conn:
		return true
	case <-cw.done:
		return false
	}
}
```

最终将该连接写入到cw.connc通道，下面看下streamWriter监听该通道的goroutine：

```
func (cw *streamWriter) run() {
......
		case conn := <-cw.connc:
......
			switch conn.t {
			case streamTypeMsgAppV2:
				enc = newMsgAppV2Encoder(conn.Writer, cw.fs)
			case streamTypeMessage:
				enc = &messageEncoder{w: conn.Writer}
......
```

当监听到cw.connc通道有数据时，获取该数据，即与其他某个server的连接，然后获取conn.Writer封装成一个encoder，用来将要发送的数据发送出去。

在rc.transport.AddPeer方法中调用了startPeer方法，里面创建了streamReader，并开启了一个goroutine：

```
func (cr *streamReader) run() {
	t := cr.typ

	for {
		rc, err := cr.dial(t)
		if err != nil {
        ......
			}
		} else {
			cr.status.activate()
			err = cr.decodeLoop(rc, t)

.......
```

通过rc, err := cr.dial(t)与对端建立连接，在err := cr.decodeLoop(rc, t)中循环读取该连接的数据：

# 集群选举与日志

## 定时器

raft协议以定时器为基础来选举,有两种定时器,选举定时器和心跳定时器,选举定时器默认1000ms, 心跳定时器默认100ms.

实际在代码中, 只定义了一个定时器, **用一个定时器来管理2个定时逻辑**

在raft结构体中, 有如下参数和定时器有关

```
type raft struct {
......

	// Leader才会有的心跳计数器,注意是计算器而不是定时器
	heartbeatElapsed int

	heartbeatTimeout int
	electionTimeout  int

	randomizedElectionTimeout int //范围是[electiontimeout, 2 * electiontimeout)

    /* tick是超时定时器的callback, Follower、Candidate、PreCandidate的tick值为tickElection, Leader值为tickHeartbeat */
	tick func()
}
```
`tickElection()`和 `tickHeartbeat()`较简单可以自己分析


## raft集群节点

集群中每个节点, 定义为一个结构体

```
type raftNode struct {
	proposeC    <-chan string            // proposed messages (k,v)
	confChangeC <-chan raftpb.ConfChange // proposed cluster config changes
	commitC     chan<- *string           // entries committed to log (k,v)
	errorC      chan<- error             // errors from raft session

	id          int      // client ID for raft session
	peers       []string // raft peer URLs
	join        bool     // node is joining an existing cluster
	waldir      string   // path to WAL directory
	snapdir     string   // path to snapshot directory
	getSnapshot func() ([]byte, error)
	lastIndex   uint64 // index of log at start

	confState     raftpb.ConfState
	snapshotIndex uint64
	appliedIndex  uint64

	// raft backing for the commit/error channel
	node        raft.Node
	raftStorage *raft.MemoryStorage
	wal         *wal.WAL

	snapshotter      *snap.Snapshotter
	snapshotterReady chan *snap.Snapshotter // signals when snapshotter is ready

	snapCount uint64
	transport *rafthttp.Transport
	stopc     chan struct{} // signals proposal channel closed
	httpstopc chan struct{} // signals http server to shutdown
	httpdonec chan struct{} // signals http server shutdown complete
}
```

其运行函数为`(rc *raftNode) serveChannels()`

入口定义一个定时器, `ticker := time.NewTicker(100 * time.Millisecond)`, 每隔100ms触发一次, 每次触发该100ms定时事件,

```
	for {
		select {
		case <-ticker.C:
			rc.node.Tick()
```

raftNode会向node的n.tickc通道写入事件，这样相当于node有了一个每隔100ms触发一次的时钟信号

在node的一个routine中会监听r.msgs是否有消息要处理，有的话会将r.msgs封装成Ready写入到node的n.readyc通道，raftNode当监听到n.readyc通道有消息时会将消息写入到cw.msgc通道，streamWriter会把cw.msgc通道的消息发送到对应的server

当对应peer的streamReader监听到p.recvc通道有事件并且事件为投票响应消息时会将消息写入到node的n.recvc通道，node根据投票响应结果统计投赞成票的server个数，如果超过半数则变为leader.

## 投票条件

`/raft/raft.go`的`Step()`函数为投票的处理函数, 来看看收到投票请求的时候, 对应的代码

```
	case pb.MsgVote, pb.MsgPreVote:
		if r.isLearner {
			return nil
		}

		if (r.Vote == None || m.Term > r.Term || r.Vote == m.From) && r.raftLog.isUpToDate(m.Index, m.LogTerm) {
			r.send(pb.Message{To: m.From, Term: m.Term, Type: voteRespMsgType(m.Type)})
			if m.Type == pb.MsgVote {
				r.electionElapsed = 0
				r.Vote = m.From
			}
```

看投赞成票的条件`r.raftLog.isUpToDate(m.Index, m.LogTerm)`

```
func (l *raftLog) isUpToDate(lasti, term uint64) bool {
	return term > l.lastTerm() || (term == l.lastTerm() && lasti >= l.lastIndex())
}
```

`m.Index`为candicate的最新日志的索引位置，即参数中的`lasti`，`m.LogTerm`为candicate最新日志的任期号，即参数中的`term`。

赞成条件为candicate的最新日志的任期号比follower的最新日志的任期号大`term > l.lastTerm()`，或者在双方最新日志任期号相同的情况下，candicate最新日志的索引位置要比follower的最新日志索引位置大，即比follower的日志更新 `term == l.lastTerm() && lasti >= l.lastIndex()`

然后回应`myVoteRespType`消息, candicate在`stepCandicate()`函数中判断投票数是否超过半数来决定是否升级为leader.

## 日志追加

**只有leader节点可以写日志**

在candicate升级为leader之后,会定期发送心跳消息到其他节点

并且在每次收到follower的心跳回复后，会根据follower与leader自己的日志对比将没发送的日志发送给follower

在`(r *raft) sendHeartbeat()`可以看到心跳消息如下:

```
pb.Message{
    To:      to,
    Type:    pb.MsgHeartbeat,
    Commit:  commit,   //leader的日志当前提交的索引
    Context: ctx,
}
```

在`stepCandidate()`和`stepFollower()`中都会响应`pb.MsgHeartbeat`消息,调用`handleHeartbeat()` (其中Candidate会先变成Follower, 因为心跳消息代表集群已经有leader了)

```
func (r *raft) handleHeartbeat(m pb.Message) {
	r.raftLog.commitTo(m.Commit)
	r.send(pb.Message{To: m.From, Type: pb.MsgHeartbeatResp, Context: m.Context})
}
```

follower会根据leader的心跳请求中的日志提交位置信息，将自己的日志提交索引设置到对应的位置，并发送心跳响应给leader

当leader在函数`stepLeader()`收到follower的心跳响应`pb.MsgHeartbeatResp`后，会比较该follower与leader日志的匹配位置`pr.Match`与leader日志的最新位置，如果两个位置不相等，说明还有日志需要发送给该follower，最终使得该follower的日志追上leader的日志。

```
if pr.Match < r.raftLog.lastIndex() {
        r.sendAppend(m.From)
    }
```

`sendAppend()`

```
//消息的发送目的server的id
m.To = to

term, errt := r.raftLog.term(pr.Next - 1)
ents, erre := r.raftLog.entries(pr.Next, r.maxMsgSize)
...

//消息类型：附加日志消息
m.Type = pb.MsgApp
//上一次发送给该follower的日志索引
m.Index = pr.Next - 1
//上一次发送给该follower的日志的任期号
m.LogTerm = term
//要发送的日志条目
m.Entries = ents
//leader的日志提交位置
m.Commit = r.raftLog.committed
if n := len(m.Entries); n != 0 {
    switch pr.State {
    // optimistically increase the next when in ProgressStateReplicate
    case ProgressStateReplicate:
        last := m.Entries[n-1].Index
        pr.optimisticUpdate(last)
        pr.ins.add(last)
    case ProgressStateProbe:
        pr.pause()
    default:
        r.logger.Panicf("%x is sending append in unhandled state %s", r.id, pr.State)
    }
}
r.send(m)

```

pr.Next表示要发给该follower的下一条日志的索引，pr.Next－1表示上一次发给该follower的日志，因此r.raftLog.term(pr.Next - 1)表示上次发给该follower的日志的任期号，r.raftLog.entries(pr.Next, r.maxMsgSize)表示从从leader日志中取出要发给该follower的日志条目

follower响应如下:

```
case pb.MsgApp:
    r.electionElapsed = 0
    r.lead = m.From
    r.handleAppendEntries(m)
```

handleAppendEntries方法如下：

```
func (r *raft) handleAppendEntries(m pb.Message) {
    // m.Index表示leader发送给follower的上一条日志的索引位置，
    // 如果当前follower在该位置的日志已经提交过了(有可能该leader是刚选举产生的，没有follower的日志信息，所以设置m.Index=0)，
    // 则把follower当前提日志交的索引位置告诉leader，让leader从该follower提交位置的下一条位置的日志开始发送给follower
    if m.Index < r.raftLog.committed {
        r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.committed})
        return
    }
    //将日志追加到follower的日志中，可能存在冲突，因此需要先找到冲突的位置，然后用leader发送来的日志中从冲突位置开始覆盖follower的日志
    if mlastIndex, ok := r.raftLog.maybeAppend(m.Index, m.LogTerm, m.Commit, m.Entries...); ok {
        //mlastIndex为follower已经追加好的最新日志的位置，追加成功后要把该信息告诉leader，以便leader会把该位置之后的日志再发送给该follower
        r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: mlastIndex})
    } else {
        r.logger.Debugf("%x [logterm: %d, index: %d] rejected msgApp [logterm: %d, index: %d] from %x",
        r.id, r.raftLog.zeroTermOnErrCompacted(r.raftLog.term(m.Index)), m.Index, m.LogTerm, m.Index, m.From)
        // 如果leader与follower的日志还没有匹配上，那么把follower的最新日志的索引位置告诉leader，
        // 以便leader下一次从该follower的最新日志位置之后开始尝试发送附加日志，直到leader与follower的日志匹配上了就能追加日志成功了
        r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: m.Index, Reject: true, RejectHint: r.raftLog.lastIndex()})
    }
}
```

r.raftLog.maybeAppend方法是将leader发送过来的日志追加到follower本地的方法，但如果leader发送过来的日志与follower的日志无法匹配(follower在index位置的日志的任期号不是logTerm)，则要把拒绝追加日志信息发送给leader，包含follower的最新日志位置，以便leader下一次从该位置尝试匹配，如下

```
func (l *raftLog) maybeAppend(index, logTerm, committed uint64, ents ...pb.Entry) (lastnewi uint64, ok bool) {
    //index，logTerm为leader上次发送给该follower的日志索引和日志的term，committed是可以提交的日志索引，ents为发过来的日志条目
    //只有follower能够匹配eader上次发送的日志索引和term，才能正常响应
    if l.matchTerm(index, logTerm) {
        //最新的日志索引
        lastnewi = index + uint64(len(ents))
        //获取冲突的日志索引，有些情况下leader发过来的日志不能直接追加，索引需要找到最新匹配的位置，从该位置之后的日志全部被leader覆盖
        ci := l.findConflict(ents)
        switch {
        case ci == 0:
        case ci <= l.committed:
            l.logger.Panicf("entry %d conflict with committed entry [committed(%d)]", ci, l.committed)
        default:
            offset := index + 1
            //取出从冲突的位置开始的日志，覆盖自己的日志，即出现冲突时以leader的日志为准
            l.append(ents[ci-offset:]...)
        }
        //如果leader的已提交的日志索引大于leader复制给当前follower的最新日志的索引，说明follower落后了，对于这次复制来的日志全都直接提交，否则提交leader已经提交的日志索引的日志
         l.commitTo(min(committed, lastnewi))
        return lastnewi, true
    }
    return 0, false
}
```

leader收到响应又会怎么处理呢?

raft的`stepLeader()`方法

```
case pb.MsgAppResp:
    pr.RecentActive = true

    if m.Reject {
        //如果处于ProgressStateReplicate状态，则将pr.Next降低为pr.Match + 1，以便去和follower的日志进行匹配
        if pr.maybeDecrTo(m.Index, m.RejectHint) {
            r.logger.Debugf("%x decreased progress of %x to [%s]", r.id, m.From, pr)
            //如果处于ProgressStateReplicate状态，则转变为ProgressStateProbe状态，去探测follower的匹配位置
            if pr.State == ProgressStateReplicate {
                pr.becomeProbe()
            }
            r.sendAppend(m.From)
        }
    } else {
        oldPaused := pr.IsPaused()
        //m.Index为follower的最新日志索引位置，根据该位置更新pr.Match和pr.Next, pr.Match=m.Index, pr.Next=m.Index+1
        if pr.maybeUpdate(m.Index) {
            switch {
            //一旦追加日志成功，则从ProgressStateProbe状态转变为ProgressStateReplicate状态，加快日志追加过程
            case pr.State == ProgressStateProbe:
                pr.becomeReplicate()
            case pr.State == ProgressStateSnapshot && pr.needSnapshotAbort:
                pr.becomeProbe()
            case pr.State == ProgressStateReplicate:
                //pr.ins用于限制发送消息的速率，当发送时将日志索引写入到pr.ins，pr.ins有数量限制，当发送消息收到回复后再把pr.ins中该发送成功日志的索引在pr.ins中移除掉
                pr.ins.freeTo(m.Index)
            }

            //收到follower的日志追加成功响应后判断是否能commit一部分日志
            if r.maybeCommit() {
                //向其他follower发送commit日志消息
                r.bcastAppend()
            } else if oldPaused {
                // update() reset the wait state on this node. If we had delayed sending
                // an update before, send it now.
                r.sendAppend(m.From)
            }
            // Transfer leadership is in progress.
            if m.From == r.leadTransferee && pr.Match == r.raftLog.lastIndex() {
                r.sendTimeoutNow(m.From)
            }
        }
    }
```

当leader收到附加日志拒绝消息时，说明p.Next太大了，而follower在p.Next-1位置的日志没有与follower匹配上，需要将p.Next降低为p.Match+1。然后转变为ProgressStateProbe状态，去探测follower与leader的日志匹配位置

当leader收到附加日志成功消息时，则要更新pr.Match和pr.Next，m.Index是follower的最新日志位置，要设置pr.Match=m.Index, pr.Next=m.Index+1。每次附加日志成功，就尝试提交下可以提交的日志(r.maybeCommit())，如果日志复制到了过半数server，说明可以提交了，便向其他follower发送日志提交请求(r.bcastAppend())

```
func (r *raft) maybeCommit() bool {
   mis := make(uint64Slice, 0, len(r.prs))
  for id := range r.prs {
      mis = append(mis, r.prs[id].Match)
  }
  //mis中保存着复制到每个server节点的日志索引，这里进行从大到小排序
  sort.Sort(sort.Reverse(mis))
  //如果节点数量为5，r.quorum()-1＝2，则在5个节点中日志第三新的节点的最新日志索引就是复制到过半数节点的日志索引，这个位置的日志可以提交啦
  mci := mis[r.quorum()-1]
  //尝试提交日志
  return r.raftLog.maybeCommit(mci, r.Term)
}

```

# 添加配置

raftNode监听`proposeC`这个channel, Leader节点的`stepLeader()`方法中, 会监听`MsgProp消息`, 该集群变更作为日志追加到本地`r.appendEntry(m.Entries...)`, 然后向其他follower发送附加日志`rpc：r.bcastAppend()`, 向`n.confc`通道写入集群变更消息, 过程比较类似就不详细分析了.
