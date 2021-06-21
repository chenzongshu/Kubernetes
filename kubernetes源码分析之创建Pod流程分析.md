# kubernetes源码分析: 创建Pod流程分析

敲下`kubectl run nginx --image=nginx --replicas=3` 命令后，K8s 中发生了哪些事情？

# kubectl

入口地址在 `cmd/kubectl/kubectl.go`，进过封装跳转，执行到 `NewKubectlCommand`

输出是一个cobar的command

```go
func NewKubectlCommand(in io.Reader, out, err io.Writer) *cobra.Command {
······
  // 返回MatchVersionFlags结构体，名字是matchVersionKubeConfigFlags
  // MatchVersionFlags
	kubeConfigFlags := genericclioptions.NewConfigFlags(true).WithDeprecatedPasswordFlag()
	kubeConfigFlags.AddFlags(flags)
	matchVersionKubeConfigFlags := cmdutil.NewMatchVersionFlags(kubeConfigFlags)
······  
  // 返回的f是一个genericclioptions.RESTClientGetter
  f := cmdutil.NewFactory(matchVersionKubeConfigFlags)
·····
  // 根据不同命令调不同函数执行
	groups := templates.CommandGroups{
		{
			Message: "Basic Commands (Beginner):",
			Commands: []*cobra.Command{
				create.NewCmdCreate(f, ioStreams),
				expose.NewCmdExposeService(f, ioStreams),
				run.NewCmdRun(f, ioStreams),
				set.NewCmdSet(f, ioStreams),
			},
		}, 
······    
}
```







