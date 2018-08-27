# 存储数据结构

Etcd存储在集群搭建和使用篇有简介，总结起来有如下特点：

- 采用kv型数据存储，一般情况下比关系型数据库快。
- 支持动态存储(内存)以及静态存储(磁盘)。
- 分布式存储，可集成为多节点集群。
- 存储方式，采用类似目录结构。
  1、只有叶子节点才能真正存储数据，相当于文件。
  2、叶子节点的父节点一定是目录，目录不能存储数据。

叶子节点数据结构位于 `/store/store.go`

```
type store struct {
	Root           *node
	WatcherHub     *watcherHub
	CurrentIndex   uint64
	Stats          *Stats
	CurrentVersion int
	ttlKeyHeap     *ttlKeyHeap  // need to recovery manually
	worldLock      sync.RWMutex // stop the world lock
	clock          clockwork.Clock
	readonlySet    types.Set
}
```
其父节点数据结构位于`/store/node.go`

```
type node struct {
	Path string

	CreatedIndex  uint64
	ModifiedIndex uint64

	Parent *node `json:"-"` // should not encode this field! avoid circular dependency.

	ExpireTime time.Time
	Value      string           // for key-value pair
	Children   map[string]*node // for directory

	// A reference to the store this node is attached to.
	store *store
}
```

其中`Path`即为key

## WAL

### 内存中的数据格式

```
 +-------------------------------+
 | F |  Pad  |                   |
 +-----------+  Length           |
 |                               |
 +-------------------------------+
 |             type              |
 +-------------------------------+
 |              CRC              |
 +-------------------------------+
 |             Data              |
 +-------------------------------+
```

字段名称 | 含义 | 占用大小  
------- | ------- | -------  
文本1 | 文本2 | 文本3  
F|是否存在补齐数据    0:表示没有补齐字段   1:表示存在补齐字段|1bit
Pad|表示补齐长度。在F为1时有效|7bit
Length|表示数据有效负载长度,不包括F、Pad自身长度、补齐字段。|56bit
Type|类型|int64，8字节，有符号
CRC|校验|uint32，4字节，无符号
Data|私有数据
 
### WAL文件数据格式

当我们持久化到文件系统中，数据格式并不是上面介绍，而是grpc格式。

WAL文件以小端序方式存储

启动一个全新etcd，默认会在目录：/var/lib/etcd/default.etcd/member/wal/中生成一个.wal文件。

### WAL的定义和创建

定义在`wal.go`中, WAL日志文件遵循一定的命名规则，由`walName()`实现，格式为"序号--raft日志索引.wal"。

```
// 根据seq和index产生wal文件名
func walName(seq, index uint64) string {
	return fmt.Sprintf("%016x-%016x.wal", seq, index)
}
```
WAL对外暴露的创建接口就是`Create()`函数

```
// Create creates a WAL ready for appending records. The given metadata is
// recorded at the head of each WAL file, and can be retrieved(检索) with ReadAll.
func Create(dirpath string, metadata []byte) (*WAL, error) {
	if Exist(dirpath) {
		return nil, os.ErrExist
	}

	// 先在.tmp临时文件上做修改，修改完之后可以直接执行rename,这样起到了原子修改文件的效果
	tmpdirpath := filepath.Clean(dirpath) + ".tmp"
	if fileutil.Exist(tmpdirpath) {
		if err := os.RemoveAll(tmpdirpath); err != nil {
			return nil, err
		}
	}
	if err := fileutil.CreateDirAll(tmpdirpath); err != nil {
		return nil, err
	}

	// dir/filename  ,filename从walName获取   seq-index.wal
	p := filepath.Join(tmpdirpath, walName(0, 0))

	// 对文件上互斥锁
	f, err := fileutil.LockFile(p, os.O_WRONLY|os.O_CREATE, fileutil.PrivateFileMode)
	if err != nil {
		return nil, err
	}

	// 定位到文件末尾
	if _, err = f.Seek(0, io.SeekEnd); err != nil {
		return nil, err
	}

	// 预分配文件，大小为SegmentSizeBytes（64MB）
	if err = fileutil.Preallocate(f.File, SegmentSizeBytes, true); err != nil {
		return nil, err
	}

	// 新建WAL结构
	w := &WAL{
		dir:      dirpath,
		metadata: metadata,// metadata 可为nil
	}

	// 在这个wal文件上创建一个encoder
	w.encoder, err = newFileEncoder(f.File, 0)
	if err != nil {
		return nil, err
	}

	// 把这个上了互斥锁的文件加入到locks数组中
	w.locks = append(w.locks, f)
	if err = w.saveCrc(0); err != nil {
		return nil, err
	}

	// 将metadataType类型的record记录在wal的header处
	if err = w.encoder.encode(&walpb.Record{Type: metadataType, Data: metadata}); err != nil {
		return nil, err
	}

	// 保存空的snapshot
	if err = w.SaveSnapshot(walpb.Snapshot{}); err != nil {
		return nil, err
	}

	// 重命名，之前以.tmp结尾的文件，初始化完成之后重命名，类似原子操作
	if w, err = w.renameWal(tmpdirpath); err != nil {
		return nil, err
	}

	// directory was renamed; sync parent dir to persist rename
	pdir, perr := fileutil.OpenDir(filepath.Dir(w.dir))
	if perr != nil {
		return nil, perr
	}

    // 将上述的所有文件操作进行同步
	if perr = fileutil.Fsync(pdir); perr != nil {
		return nil, perr
	}

	// 关闭目录
	if perr = pdir.Close(); err != nil {
		return nil, perr
	}

	return w, nil
}
```

其中, `SaveSnapshot()`是做walpb.Snapshot持久化的,  里面的内容略过, 不过里面有一行代码`if err := w.encoder.encode(rec)`表示一条Record需要先把序列化后才能持久化，这个是通过`encode()`函数完成的（encoder.go）

一个Record被序列化之后（这里为JOSN格式），会以一个Frame的格式持久化。Frame首先是一个长度字段（`encodeFrameSize()`完成,在`encoder.go`文件），64bit，其中MSB表示这个Frame是否有padding字节，接下来才是真正的序列化后的数据

### WAL存储

当raft模块收到一个proposal时就会调用Save方法完成（定义在wal.go）持久化

```
func (w *WAL) Save(st raftpb.HardState, ents []raftpb.Entry) error {
	w.mu.Lock()  // 上锁
	defer w.mu.Unlock()

	// short cut（捷径）, do not call sync
	// IsEmptyHardState returns true if the given HardState is empty.
	if raft.IsEmptyHardState(st) && len(ents) == 0 {
		return nil
	}

	// 是否需要同步刷新磁盘
	mustSync := raft.MustSync(st, w.state, len(ents))

	// TODO(xiangli): no more reference operator
	// 保存所有日志项
	for i := range ents {
		if err := w.saveEntry(&ents[i]); err != nil {
			return err
		}
	}

	// 持久化HardState, HardState表示服务器当前状态，定义在raft.pb.go，主要包含Term、Vote、Commit
	if err := w.saveState(&st); err != nil {
		return err
	}

	// 获取最后一个LockedFile的大小（已经使用的）
	curOff, err := w.tail().Seek(0, io.SeekCurrent)
	if err != nil {
		return err
	}
	// 如果小于64MB
	if curOff < SegmentSizeBytes {
		if mustSync {
			// 如果需要sync，就执行sync
			return w.sync()
		}
		return nil
	}

	// 否则执行切割（也就是说明，WAL文件是可以超过64MB的）
	return w.cut()
}
```

## snapshot

snapshot比wal大小要小5倍左右，只有`CRC`和`Data`两个字段

etcd中对raft snapshot的定义如下（在文件raft.pb.go）：
```
type Snapshot struct {
	Data             []byte           `protobuf:"bytes,1,opt,name=data" json:"data,omitempty"`
	Metadata         SnapshotMetadata `protobuf:"bytes,2,opt,name=metadata" json:"metadata"`
	XXX_unrecognized []byte           `json:"-"`
}
```

Metadata则是snaoshot的元信息

```
// snapshot的元数据
type SnapshotMetadata struct {
      // 最后一次的配置状态
	ConfState        ConfState `protobuf:"bytes,1,opt,name=conf_state,json=confState" json:"conf_state"` 
       // 被快照取代的最后的条目在日志中的索引值(appliedIndex)
	Index            uint64    `protobuf:"varint,2,opt,name=index" json:"index"`
       // 该条目的任期号                     
	Term             uint64    `protobuf:"varint,3,opt,name=term" json:"term"`                           
	XXX_unrecognized []byte    `json:"-"`
}
```

snapshot持久化使用`func (s *Snapshotter) SaveSnap(snapshot raftpb.Snapshot)`

过程比较简单, 略去, 里面可以看到snapshot文件的命名规则

```
// 将raft snapshot序列化后持久化到磁盘
func (s *Snapshotter) save(snapshot *raftpb.Snapshot) error {
	// 产生snapshot的时间
	start := time.Now()
	// snapshot的文件名Term-Index.snap
	fname := fmt.Sprintf("%016x-%016x%s", snapshot.Metadata.Term, snapshot.Metadata.Index, snapSuffix)
```

## 动态存储



```
+--------------------+          +--------------------+
| etcdserver/raft.go |          |   raft/storge.go   |
|                    +<-------->+                    |
|   startNode()      |          | NewMemoryStorge()  |
+---------^----------+          +--------------------+
          |
          |
          |
+---------v----------+          +--------------------+
|     rafe/node.go   |          |     rafe/node.go   |
|                    +<-------->+                    |
|   StartNode()      |          |     newRaft()      |
+---------^----------+          +--------------------+
          |
          |
          |
+---------v----------+
|     raft/node.go   |
|                    |
|        run()       |
+--------------------+
```
首先调用NewMemoryStorage进行初始化，然后在newRaft()中生成raftLog对象并且调用InitialState()进行状态初始化，最后在node中run方法接收数据。
