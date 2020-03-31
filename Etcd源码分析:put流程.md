# put操作

put操作是etcd v3 client支持的命令，和v2的set用法差不多

但是需要注意的是，如果你在一个3节点的etcd集群中，A节点切换为v3 client版本，然后put进了一对key-value，在B节点，还是v2的client，这个时候你get不到数据的，如果在B节点切换到了v3 client，这个时候才可以get到数据

简单的说，**v2和v3 client，插入数据到同一个etcd集群中，数据不能互通** 

# put流程分析

## client端

`put`命令接受的入口，在`/etcdctl/ctlv3/command/put_command.go`中的`NewPutCommand()`函数中，采用了一个`corba`结构体接受命令参数，实际Run执行的命令是`putCommandFunc()`

```
func putCommandFunc(cmd *cobra.Command, args []string) {
	key, value, opts := getPutOp(cmd, args)  //解析参数

	ctx, cancel := commandCtx(cmd)
	resp, err := mustClientFromCmd(cmd).Put(ctx, key, value, opts...)
	cancel()
	if err != nil {
		ExitWithError(ExitError, err)
	}
	display.Put(*resp) //打印返回的结果
}
```

该函数的核心就是`mustClientFromCmd(cmd).Put(ctx, key, value, opts...)`

`mustClientFromCmd()`函数返回的是一个`clientv3.Client`结构体指针

相当于上面调用了`clientv3.Client.Put()`

在 `/clientV3/client.go` 中，可以看到`Client` 结构体的定义， 

```
type Client struct {
	Cluster
	KV
	Lease
```

里面内嵌一个`KV`的interface，由下面的代码具体实现接口

```
func (kv *kv) Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error) {
	r, err := kv.Do(ctx, OpPut(key, val, opts...))
	return r.put, toErr(ctx, err)
}
```

继续往下，是到了`func (c *kVClient) Put(......)`，该函数里面，通过grpc的调用发到了server端 `grpc.Invoke(ctx, "/etcdserverpb.KV/Put", in, out, c.cc, opts...)`


## server端

在`/etcdserver/etcdserverpb/rpc.pb.go`里面，可以看到上面定义的ServiceName和MethodName，可以找到对应的方法`_KV_Put_Handler`

```
var _KV_serviceDesc = grpc.ServiceDesc{
	ServiceName: "etcdserverpb.KV",
	HandlerType: (*KVServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "Range",
			Handler:    _KV_Range_Handler,
		},
		{
			MethodName: "Put",
			Handler:    _KV_Put_Handler,
		},
```

往下追踪 `srv.(KVServer).Put(ctx, in)` -> `(s *EtcdServer) Put()` -> ··· ··· -> `(s *EtcdServer) processInternalRaftRequestOnce(...)`

在该函数里面有一句关键调用 `s.r.Propose(cctx, data)`

`s`是`EtcdServer`, `r`是其里面的成员变量`raftNode`, 这就是进入raft协议相关的节奏了

`(n *node) Propose()` -> `step()`, 该函数代码较短，来看看

```
func (n *node) step(ctx context.Context, m pb.Message) error {
	ch := n.recvc
	if m.Type == pb.MsgProp {
		ch = n.propc
	}

	select {
	case ch <- m:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	case <-n.done:
		return ErrStopped
	}
}
```

这段代码主要就是根据消息类型来把传进来的`pb.Message`赋值给channel `n.recvc`或者`n.propc`，上面的`Propose()`定义了`pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Data: data}}})`，所以就是赋值给了`n.propc`


## raft协议里的流程

在一个raft集群启动完成以后, `(n *node) run()` 函数就是其运行的主函数, 里面是一个死循环, 循环中会根据channel来响应各种事件, 从而跳转状态

上节有说到channel `propc`里面被塞入了数据, `run()` 函数里面就会有对应的处理,代码如下:

```
		case m := <-propc:
			m.From = r.id
			r.Step(m)
```

对应的`(r *raft) Step(m pb.Message)`函数也是raft协议中的核心函数, 负责状态机的跳转, 里面主要有2个逻辑

- 根据传进来的term和本身的term的大小, 决定要做的动作, 具体的逻辑原理可见raft协议原理;
- 根据消息类型m.Type做不同的处理

注意其最后一段

```
	default:
		r.step(r, m)
```

`step()`是一个类似函数借口的东西,根据节点的类型不同而调用不同的函数,比如leader节点该函数就是`raft.stepLeader()`

在node run的死循环中，看看开头

```
             // readyc 和 advance 只有一个是有效值
		if advancec != nil {
			readyc = nil
		} else {
			rd = newReady(r, prevSoftSt, prevHardSt)
                   
                   // 如果raft.msgs中队列大小不为0 也会返回true 表示有数据发出
			if rd.containsUpdates() {
				readyc = n.readyc
			} else {
				readyc = nil
			}
		}
```

如果`advancec`是nil，说明刚commit了，可以创建`ready` channel来继续去把ready commit的commit了。

(未完待续)
