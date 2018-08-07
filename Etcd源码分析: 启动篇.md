# 启动

启动从源码 `/etcdmain/main.go` 中的`main`函数开始

```
func Main() {
	checkSupportArch()   // 检查系统是否支持

	if len(os.Args) > 1 {  // 获取入参
		cmd := os.Args[1]  // 获取启动命令
		if covArgs := os.Getenv("ETCDCOV_ARGS"); len(covArgs) > 0 {
			args := strings.Split(os.Getenv("ETCDCOV_ARGS"), "\xe7\xcd")[1:]
			rootCmd.SetArgs(args)
			cmd = "grpc-proxy"
		}
		switch cmd {
		case "gateway", "grpc-proxy":
			if err := rootCmd.Execute(); err != nil {
				fmt.Fprint(os.Stderr, err)
				os.Exit(1)
			}
			return
		}
	}

	startEtcdOrProxyV2()
}
```
上面仔细去看，前面是根据命令行输入的第一个参数去启动不同的代码逻辑

后面的`startEtcdOrProxyV2()`则是启动etcd server的函数

## 命令行输入

我们来查看一下etcd命令的使用帮助:

```
$ ./etcd -help
Usage:

  etcd [flags]
    Start an etcd server.

  etcd --version //查看版本
    Show the version of etcd.

  etcd -h | --help //获取帮助
    Show the help information about etcd.

  etcd --config-file //设置配置文件
    Path to the server configuration file.

  etcd gateway //启动 L4 TCP网关代理
    Run the stateless pass-through etcd TCP connection forwarding proxy.

  etcd grpc-proxy //L7 grpc 代理
    Run the stateless etcd v3 gRPC L7 reverse proxy.
```

可以看到有三个启动命令

```
 ./etcd //启动etcd服务
 ./etcd gateway start //启动tcp网关代理
 ./etcd grpc-proxy start //启动grpc网关代理
```

其中,代码中的`rootCmd`,这个是一个叫corba的在github上的开源库,它的设计目的是提供一个编写/生成交互式命令程序的框架.

简单的说,它就是可以添加命令和子命令的插件库,查看 gateway.go的源码可以发现

```
var (
	rootCmd = &cobra.Command{                 (1)
		Use:        "etcd",
		Short:      "etcd server",
		SuggestFor: []string{"etcd"},
	}
)

func init() {
	rootCmd.AddCommand(newGatewayCommand())            (2)
}

// newGatewayCommand returns the cobra command for "gateway".
func newGatewayCommand() *cobra.Command {
	lpc := &cobra.Command{
		Use:   "gateway <subcommand>",
		Short: "gateway related command",
	}
	lpc.AddCommand(newGatewayStartCommand())           (3)

	return lpc
}

func newGatewayStartCommand() *cobra.Command {
	cmd := cobra.Command{
		Use:   "start",
		Short: "start the gateway",
		Run:   startGateway,
	}

	cmd.Flags().StringVar(&gatewayListenAddr, "listen-addr", "127.0.0.1:23790", "listen address")
	cmd.Flags().StringVar(&gatewayDNSCluster, "discovery-srv", "", "DNS domain used to bootstrap initial cluster")
	cmd.Flags().BoolVar(&gatewayInsecureDiscovery, "insecure-discovery", false, "accept insecure SRV records")
	cmd.Flags().StringVar(&gatewayCA, "trusted-ca-file", "", "path to the client server TLS CA file.")

	cmd.Flags().StringSliceVar(&gatewayEndpoints, "endpoints", []string{"127.0.0.1:2379"}, "comma separated etcd cluster endpoints")

	cmd.Flags().DurationVar(&getewayRetryDelay, "retry-delay", time.Minute, "duration of delay before retrying failed endpoints")

	return &cmd
}
```

(1) 首先生成了一个`etcd`的命令
(2) 添加了一个子命令 `gateway`
(3) 接着又添加了一个子命令`start`

组合起来,是不是就是我们看到的`etcd gateway start`,这个命令就会去执行`Run`字段指向的函数,而`Short`是注释

具体`cobra`用法可以查询相关文档

## server启动

上面有提到`main()`函数里面调用了`startEtcdOrProxyV2()`

查看其代码，发现里面做了2件事情

- 初始化配置文件
- 根据数据目录的情况，启动进程(`startEtcd()`)Etcd Server或者代理进程(`startProxy()`,代理进程是和其他etcd节点交互的)

`startEtcd()`又调用了`embed.StartEtcd(cfg)`,这个即启动EtcdServer的函数。

```
func StartEtcd(inCfg *Config) (e *Etcd, err error) {
	if err = inCfg.Validate(); err != nil {
		return nil, err
	}
	serving := false
	e = &Etcd{cfg: *inCfg, stopc: make(chan struct{})}
	cfg := &e.cfg

	······
	return e, nil
```

从函数的前面的代码可以看到，该函数入参是一个配置文件，返回值是一个`Etcd`的结构体指针，代表一个Etcd Server实例，但是不保证已经加入集群。

进来先进行了入参的检查，`inCfg.Validate()`，里面需要注意的是
`5*cfg.TickMs > cfg.ElectionMs` ：选举超时时间必须大于五倍于心跳超时时间。
`cfg.ElectionMs > maxElectionMs` ：选举超时时间必须小于5000ms

接着来看看`Etcd`的内容

```
type Etcd struct {
	Peers   []*peerListener
	Clients []net.Listener
	// a map of contexts for the servers that serves client requests.
	sctxs            map[string]*serveCtx
	metricsListeners []net.Listener

	Server *etcdserver.EtcdServer

	cfg   Config
	stopc chan struct{}
	errc  chan error

	closeOnce sync.Once
}
```

> 
- Peers和Clients都是监听器，允许多个goroutine同时连接，Peers用于内部member间通信，Clients用于外部通信
- cfg是配置参数
- Server为Etcd Server的核心配置

后面会通过`rafthttp.NewListener()`把这两个Listener初始化创建

下面初始化了`ServerConfig`结构体，这个是new一个 EtcdServer所需要的配置

```
    type ServerConfig struct {
        // etcdserver 名称，对应flag "name“
        Name           string  
        
		// etcd 用于服务发现，无需知道具体etcd节点ip即可访问etcd 服务，对应flag  "discovery"
        DiscoveryURL   string  
        
		// 供服务发现url的代理地址， 对应flag "discovery-proxy"
        DiscoveryProxy string  
        
		// 由ip+port组成，默认DefaultListenClientURLs = "http://localhost:2379"; 
        // 实际情况使用https schema，用以外部etcd client访问，对应flag "listen-client-urls"
        ClientURLs     types.URLs 
        
		// 由ip+port组成，默认DefaultListenPeerURLs   = "http://localhost:2380"; 
        // 实际生产环境使用http schema, 供etcd member 通信，对应flag "peer-client-urls"
        PeerURLs       types.URLs 
        
		// 数据目录地址，为全路径，对应flag "data-dir"
        DataDir        string   
        
		// DedicatedWALDir config will make the etcd to write the WAL to the WALDir
        // rather than the dataDir/member/wal.
        DedicatedWALDir     string
        
		// 默认是10000次事件做一次快照:DefaultSnapCount = 100000
        // 可以作为调优参数进行参考，对应flag "snapshot-count", 
        SnapCount           uint64  
        
		// 默认是5，这是v2的参数，v3内只有一个db文件，
        // DefaultMaxSnapshots = 5，对应flag "max-snapshots"
        MaxSnapFiles        uint  
        
		// 默认是5，DefaultMaxWALs      = 5，表示最大存储wal文件的个数，
        // 对应flag "max-wals"，保留的文件可以作为etcd-dump-logs工具进行debug使用。
        MaxWALFiles         uint  
        
		// peerUrl 与 etcd name对应的map,由方法cfg.PeerURLsMapAndToken("etcd")生成。
        InitialPeerURLsMap  types.URLsMap 
        
		// etcd 集群token, 对应flang "initial-cluster-token"
        InitialClusterToken string 
        
		// 确定是否为新建集群，对应flag "initial-cluster-state",
        // 由方法func (cfg Config) IsNewCluster() bool { return cfg.ClusterState == ClusterStateFlagNew }确定；
        NewCluster          bool 
        
		// 对应flag "force-new-cluster",默认为false,若为true，
        // 在生产环境内，一般用于含v2数据的集群恢复，
        // 效果为以现有数据或者空数据新建一个单节点的etcd集群，
        // 如果存在数据，则会清楚数据内的元数据信息，并重建只包含该etcd的元数据信息。
        ForceNewCluster     bool 
        
		// member间通信使用的证书信息，若peerURL为https时使用，对应flag "peer-ca-file","peer-cert-file", "peer-key-file"
        PeerTLSInfo         transport.TLSInfo
        
		// raft node 发送心跳信息的超时时间。 "heartbeat-interval" 
        TickMs           uint 
        
		// raft node 发起选举的超时时间，最大为5000ms maxElectionMs = 50000, 
        // 对应flag "election-timeout", 
        // 选举时间与心跳时间在最佳实践内建议是10倍关系。
        ElectionTicks    int 
        
		// etcd server启动的超时时间，默认为1s, 
        // 由方法func (c *ServerConfig) bootstrapTimeout() time.Duration确定；
        BootstrapTimeout time.Duration 

        // 默认为0，单位为小时，主要为了方便用户快速查询，
        // 定时对key进行合并处理，对应flag "auto-compaction-retention",
        // 由方法func NewPeriodic(h int, rg RevGetter, c Compactable) *Periodic确定，
        // 具体compact的实现方法为：
        //func (s *kvServer) Compact(ctx context.Context, r *pb.CompactionRequest) (*pb.CompactionResponse, error)
        AutoCompactionRetention int  
        
		// etcd后端数据文件的大小，默认为2GB，最大为8GB, v3的参数，
        // 对应flag  "quota-backend-bytes" ，具体定义：etcd\etcdserver\quota.go   
        QuotaBackendBytes       int64 

        StrictReconfigCheck bool

        // ClientCertAuthEnabled is true when cert has been signed by the client CA.
        ClientCertAuthEnabled bool

        AuthToken string
	}
```


继续往下分析，发现`StartEtcd()`函数里面最关键的调用如下

```
	······

	if e.Server, err = etcdserver.NewServer(srvcfg); err != nil {
		return e, err
	}

    ······

	e.Server.Start()

	······
```

这两个函数的具体实现都位于 `/etcdserver/server.go` 中

首先看看`NewServer()`函数，代码很长，省略，其核心思路就是先判断是不是有`WAL`文件存在，然后根据两个条件：1、是否有WAL；2、是否新建集群； 来走不同流程，最后`startNode()`,`startNode()`里面就是创建WAL文件和调用raft协议的函数`raft.StartNode(c, peers)`来启动一个新的节点


`EtcdServer.start()`函数做了简单初始化之后，调用了`EtcdServer.run()`，该函数为EtcdServer运行的主函数

主要逻辑有：

初始化了`raftReadyHandler`,里面有2个函数回调，然后调用`raftNode.start()`函数来在一个新的goroutine启动一个raft节点

然后是for循环，等待channel响应

