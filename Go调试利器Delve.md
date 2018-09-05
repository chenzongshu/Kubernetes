Go语音虽然可以用GDB调试，但是对goroutine的支持不好， 调试Go可以使用 Delve

项目的github地址为： https://github.com/derekparker/delve 

# 安装

## 方法1

先安装好`golang`和`git`

然后执行

```
go get github.com/derekparker/delve/cmd/dlv
```

## 方法2

如果你像我一样, 办公环境可能连不上网, 可以试试方法2

设置`GOPATH`, 把`delve`项目代码clone下了,然后编译

```
$ git clone https://github.com/derekparker/delve.git //再拷贝到目标环境的GOPATH下面
$ cd $GOPATH/src/github.com/derekparker/delve
$ make install
```

# 调试代码

首先,你要知道你生成的可执行文件在哪, 方法1的话是应该在 `$GOPATH/bin/`下面, 需要带路径执行, 也可以把该文件放入`/usr/bin`中

看代码例子

```
package main

import (
    "fmt"
    "sync"
    "time"
)

func dostuff(wg *sync.WaitGroup, i int) {
    fmt.Printf("goroutine id %d\n", i)
    time.Sleep(300 * time.Second)
    fmt.Printf("goroutine id %d\n", i)
    wg.Done()
}

func main() {
    var wg sync.WaitGroup
    workers := 10

    wg.Add(workers)
    for i := 0; i < workers; i++ {
        go dostuff(&wg, i)
    }
    wg.Wait()
}
```

启动调试

```
dlv debug test-debug.go
```

执行停留在`main`入口, 可以打断点

```
(dlv) break main.main
Breakpoint 1 set at 0x22d3 for main.main() ./test-debug.go:16
```

使用`continue`可以往下执行到断点所在, `next` 单步往下执行

按`回车`可以重复前一步命令

`print`可以打印变量或者表达式如: "print x < 5"

# 进阶调试

上面一节是运行写好的go文件, 我们也可以进入正在运行的程序, 和gdb类似

```
dlv attach 40994  // 40994为程序PID
```

# 启动命令列表

- `dlv attach`	- Attach到已经运行的进程.
- `dlv connect`	- 连接调试服务器.
- `dlv core`	- 调试core dump.
- `dlv debug`	- 编译并开始调试 main 包 
- `dlv exec`	- 执行一个可执行文件,并开始调试
- `dlv replay`	- Replays a rr trace.
- `dlv test`	- 编译并test一个程序
- `dlv trace`	- 编译并trace一个程序
- `dlv version`	- 查看版本

# 调试指令

见delve官方手册:

https://github.com/derekparker/delve/blob/master/Documentation/cli/README.md
