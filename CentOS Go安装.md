# 下载

下载go安装包, 如果没梯子可以到如下网站下载: https://gomirrors.org/

# 安装

tar -C /usr/local -zxvf go1.13.linux-amd64.tar.gz

# 设置环境变量

`vim /etc/profile` 增加

```
export PATH=$PATH:/usr/local/go/bin 
export GOROOT=/usr/local/go 
export GOPATH=$HOME/go 
```

然后重载配置文件`source /etc/profile`

创建go的PATH目录 `mkdir /root/go`

# 验证

查看go的环境变量和输入命令`go version`查看go版本



# 设置代理

```
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.io,direct
```



