# 简介

[cobra](https://github.com/spf13/cobra)是一个golang的命令行程序库， 像kubernetes，docker，etcd都使用了该库， 同时也是一个应用， 可以生成应用框架。



## 基本概念

Cobra有三个基本概念：

- commands：代表行为
- arguments：代表数值
- flags：flags代表对行为的改变, 有两种flag
  - `Persistent Flags`: 该标志可用于为其分配的命令以及该命令的所有子命令。使用`rootCmd.PersistentFlags().BoolVarP()`命令增加参数
  - `Local Flags`: 该标志只能用于分配给它的命令。使用类似`rootCmd.Flags().BoolVarP()`命令增加参数

基本模型如下：

```
APPNAME COMMAND ARG --FLAG
```

例如：

```go
# server是commands，port是flag
hugo server --port=1313

# clone是commands，URL是arguments，brae是flags
git clone URL --bare
```

# 安装

```
go get -u github.com/spf13/cobra/cobra@v1.0.0
```

- 官方文档命令在高版本go下会报错， 这里必须指定版本

- 如果出现包下载不了的情况， 可以进入gopath的src目录， 创建`golang.org/x/`, 然后去github上面git colne对应的库， 比如

```go
git clone https://github.com/golang/sys.git
git clone https://github.com/golang/text.git
```

接下来，在golang代码中引用Cobra：

```go
import "github.com/spf13/cobra"
```



# Cobra工具生成框架

## 生成框架

使用命令生成框架

```
 /root/go/bin/cobra init appcenter --pkg-name appcenter
```

如果没有用到配置文件, 可以在初始化的时候关闭 Viper 的选项以减少编译后文件的体积，如下：

```go
cobra init demo --pkg-name=demo --viper=false
```

查看目录和文件

```
appcenter
├── cmd
│   └── root.go
├── LICENSE
└── main.go
```

`main.go`非常简单, 通常只干一件事，即初始化Cobra并执行command

```
package main

import "appcenter/cmd"

func main() {
	cmd.Execute()
}

```

cmd/root.go

```go
package cmd

import (
	"fmt"
	"github.com/spf13/cobra"
	"os"

	homedir "github.com/mitchellh/go-homedir"
	"github.com/spf13/viper"
)

var cfgFile string

var rootCmd = &cobra.Command{
	Use:   "appcenter",
    //命令简介
	Short: "A brief description of your application", 
    //命令详细介绍
	Long: `A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
    // 如果有对参数校验, 可以加到这个位置, 这里就校验了apptype只能为bin或者container
	Args: func(cmd *cobra.Command, args []string) error {
		if (apptype != "bin" && apptype != "container") {
			//fmt.Println("apptype format error")
			return fmt.Errorf("apptype format error")
		}
		return nil
	},    
	// Uncomment the following line if your bare application
	// has an action associated with it:
	//	Run: func(cmd *cobra.Command, args []string) { },
}

func Execute() {
	if err := rootCmd.Execute(); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
}

# 每当执行该命令时，先执行init函数
func init() {
	cobra.OnInitialize(initConfig)

	rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.appcenter.yaml)")

	rootCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
}

// initConfig 读取配置文件和环境变量
func initConfig() {
	if cfgFile != "" {
	    // 使用 flag 标志中传递的配置文件
		viper.SetConfigFile(cfgFile)
	} else {
        // 获取 Home 目录
		home, err := homedir.Dir()
		if err != nil {
			fmt.Println(err)
			os.Exit(1)
		}
        // 在 Home 目录下面查找名为 ".appcenter" 的配置文件
		viper.AddConfigPath(home)
		viper.SetConfigName(".appcenter")
	}
    // 读取匹配的环境变量
	viper.AutomaticEnv() // read in environment variables that match
    // 如果有配置文件，则读取它
	if err := viper.ReadInConfig(); err == nil {
		fmt.Println("Using config file:", viper.ConfigFileUsed())
	}
}
```

- `Use`指定使用信息，即命令怎么被调用，格式为`name arg1 [arg2]`。`name`为命令名，后面的`arg1`为必填参数，`arg3`为可选参数，参数可以多个。

- `Short/Long`都是指定命令的帮助信息，只是前者简短，后者详尽而已。

- `Run`是实际执行操作的函数。

  > Run相关还有一系列函数, 执行顺序如下 PersistentPreRun > PreRun > Run > PostRun > PersistentPostRun

- 整体执行顺序是

  `root.go.init ` -> `main.go`的`cmd.Execute()`之前部分 -> `root.go.initConfig` -> `RUN:func`



## 增加flag

如果我们想增加flag, 比如`appcenter --name czs`这样的, 

```go
// 先定义一个变量接受参数
var name string

//再到init中增加name参数
func init() {
	cobra.OnInitialize(initConfig)

	rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.appcenter.yaml)")

	rootCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
	rootCmd.Flags().StringVarP(&name, "name", "n", "", "app template's name")

	rootCmd.MarkFlagRequired("name") //设置这个参数是必填项

}
```







## 添加子命令

在项目根目录下面创建一个名为 `upload` 的命令, 可用cobra的命令 `cobra add upload`, 可以看到`cmd`目录下面多了一个`upload.go`的文件， 其和`root.go`比较类似, 不过`init()`里面是`rootCmd.AddCommand(uploadCmd)`



如果我们需要为upload添加一个子命令, 比如`appcenter upload test`, 可以和上面一样的操作, 只是修改init函数

```go
// cmd/test.go里面
func init() {
	uploadCmd.AddCommand(testCmd)
}
```



