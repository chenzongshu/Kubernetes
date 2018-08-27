# 配置文件

`etcd`配置文件位于`/etc/etcd/etcd.conf`,该配置文件一共有5个section

名称 | 作用 
------- | ------- 
member | 本节点的配置，包括监听服务端口、心跳时间等
cluster|集群配置，包括集群状态、集群名称以及本节点广播地址
proxy|用于网络自动发现服务
security|安全配置
logging|日志功能组件

具体配置采集可以见另一个文章《Etcd集群配置和使用》

初次看到配置文件，都会有一个疑问，为什么在members已经设置了监听服务地址，为什么在cluster还要再次设置一次广播地址呢？

原因：**etcd主要的通信协议主要是http协议，对于http协议中所周知它是B/S结构，而非C/S结构，只能一端主动给另一端发消息而反过来则不可。所以对于集群来说，双方必须都要知道对方具体监听地址。**

# 服务监听

我们都知道，建立socket服务端一共有5个基本步骤（C语言）：
1、创建socket套接字
2、bind地址及端口
3、listen监听服务4
4、accept接收客户端连接
5、启动新线程为客户端服务。
正所谓万变不离其宗，到了etcd中（etcd使用默认golang http模块）也是这些步骤，只不过是被封装了一下（语法糖）

启动流程见《Etcd源码分析：启动篇》

## listener

当进入`embed/etcd.go`里面的`StartEtcd()`函数的时候

```
      //为peer创建listener，socket三部曲只到了第二个步骤
	if e.Peers, err = startPeerListeners(cfg); err != nil {
		return e, err
	}
      //为client创建listener，socket三部曲只到了第二个步骤
	if e.sctxs, err = startClientListeners(cfg); err != nil {
		return e, err
	}
```

在创建了listener之后，开始创建EtcdServer

```

      //创建EtcdServer并且创建raftNode并运行raftNode
	if e.Server, err = etcdserver.NewServer(srvcfg); err != nil {
		return e, err
	}

	// buffer channel so goroutines on closed connections won't wait forever
	e.errc = make(chan error, len(e.Peers)+len(e.Clients)+2*len(e.sctxs))

······
	e.Server.Start()

	if err = e.servePeers(); err != nil {
		return e, err
	}
	if err = e.serveClients(); err != nil {
		return e, err
	}
	if err = e.serveMetrics(); err != nil {
		return e, err
	}

	serving = true
```

Listener有两个分别为：peer listener和client listener，两者大同小异，这里拿peer listener做为分析对象。

```
func startPeerListeners(cfg *Config) (peers []*peerListener, err error) {
······
	peers = make([]*peerListener, len(cfg.LPUrls))
······
	for i, u := range cfg.LPUrls {   //循环遍历多个peer url
		if u.Scheme == "http" {
			if !cfg.PeerTLSInfo.Empty() {
				plog.Warningf("The scheme of peer url %s is HTTP while peer key/cert files are presented. Ignored peer key/cert files.", u.String())
			}
			if cfg.PeerTLSInfo.ClientCertAuth {
				plog.Warningf("The scheme of peer url %s is HTTP while client cert auth (--peer-client-cert-auth) is enabled. Ignored client cert auth for this url.", u.String())
			}
		}
        // 构造peerListener对象 监听2380 作为服务端模式
		peers[i] = &peerListener{close: func(context.Context) error { return nil }}
        
        //调用接口，创建listener对象，返回来之后，socket套接字已经完成listener监听流程
		peers[i].Listener, err = rafthttp.NewListener(u, &cfg.PeerTLSInfo)
		if err != nil {
			return nil, err
		}
		// once serve, overwrite with 'http.Server.Shutdown'
		peers[i].close = func(context.Context) error {
			return peers[i].Listener.Close()
		}
		plog.Info("listening for peers on ", u.String())
	}
	return peers, nil
}
```  
 
下面调用关系为
 
```
startPeerListeners()  [embed/etcd.go]  
-> rafthttp.NewListener()  [rafthttp/util.go]
    -> transport.NewTimeoutListener()  [pkg/transport/timeout_listener.go]
        -> newListener()  [pkg/transport/listener.go]
            -> net.Listen()  [golang net库函数]
```

## 服务监听

服务端socket需要调用Accept方法，我们来看一下serve方法。方法serve大致内容为：将每个服务放到gorouting中，也就是启动一个协程来监听服务。

先看看`servePeers()`

```
func (e *Etcd) servePeers() (err error) {
	ph := etcdhttp.NewPeerHandler(e.Server)
	var peerTLScfg *tls.Config
	if !e.cfg.PeerTLSInfo.Empty() {
		if peerTLScfg, err = e.cfg.PeerTLSInfo.ServerConfig(); err != nil {
			return err
		}
	}

	for _, p := range e.Peers {
		gs := v3rpc.Server(e.Server, peerTLScfg)
		m := cmux.New(p.Listener)
		go gs.Serve(m.Match(cmux.HTTP2()))
		srv := &http.Server{
			Handler:     grpcHandlerFunc(gs, ph),
			ReadTimeout: 5 * time.Minute,
			ErrorLog:    defaultLog.New(ioutil.Discard, "", 0), // do not log user error
		}
		go srv.Serve(m.Match(cmux.Any()))
		p.serve = func() error { return m.Serve() }
		p.close = func(ctx context.Context) error {
			// gracefully shutdown http.Server
			// close open listeners, idle connections
			// until context cancel or time-out
			stopServers(ctx, &servers{secure: peerTLScfg != nil, grpc: gs, http: srv})
			return nil
		}
	}

	// start peer servers in a goroutine
	for _, pl := range e.Peers {
		go func(l *peerListener) {
			e.errHandler(l.serve())
		}(pl)
	}
	return nil
}
```

1、生成http.hander 用于处理peer请求；
2、在for循环里面，起一些goroutine，调用`Server()`函数来接受Listener传入的连接。

我们来看看`NewPeerHandler()`

```
func newPeerHandler(cluster api.Cluster, raftHandler http.Handler, leaseHandler http.Handler) http.Handler {
	mh := &peerMembersHandler{
		cluster: cluster,
	}

 //将url和业务层handler注册到servemux中，也就是每一个url请求都会有其对应的handler进行处理
     //初始化一个Serve Multiplexer结构
      mux := http.NewServeMux()
	mux.HandleFunc("/", http.NotFound)
	mux.Handle(rafthttp.RaftPrefix, raftHandler)
	mux.Handle(rafthttp.RaftPrefix+"/", raftHandler) 
	mux.Handle(peerMembersPrefix, mh)  //处理请求/members handler是mh，即peerMembersHandler
	if leaseHandler != nil {
		mux.Handle(leasehttp.LeasePrefix, leaseHandler)
		mux.Handle(leasehttp.LeaseInternalPrefix, leaseHandler)
	}
	mux.HandleFunc(versionPath, versionHandler(cluster, serveVersion))
	return mux
}
```
应用层业务逻辑需要自己注册url和handler，这样才能保证每个http request都能够被处理。而每个handler都必须要实现对应接口ServeHTTP，例如peerMembersHandler，实现的ServeHTTP接口是用于返回集群成员列表

那么此处只是完成注册，那么在什么地方会调用此处handler？

答案是在`ServeHTTP()`里面



