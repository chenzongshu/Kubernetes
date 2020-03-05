
# kubebuilder

1. 从github网页  https://github.com/kubernetes-sigs/kubebuilder 下载二进制包

2. 解压

```
tar -C /usr/local -zxvf kubebuilder_2.3.0_linux_amd64.tar.gz
mv /usr/local/kubebuilder_2.3.0_linux_amd64 /usr/local/kubebuilder
```

3. 修改环境变量

```
vim ~/.profile

export PATH=$PATH:/usr/local/kubebuilder/bin

source ~/.profile
```

4. 开启go module和对go mod配置proxy

```
vim ~/.profile

export GO111MODULE=on
export GOPROXY=https://goproxy.io

source ~/.profile
```
