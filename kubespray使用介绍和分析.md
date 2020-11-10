# kubespray介绍

kubespray是kubernetes-SIG 基于ansible提供的kubernetes集群安装工具

官方文档： https://kubespray.io/

Github地址： https://github.com/kubernetes-sigs/kubespray

基于2.14版本， 有如下要求

- pip3、Ansible v2.9 、python-netaddr、Jinja 2.11

- 机器配置IPv4 forward

- ssh免密配置

- 关闭firewalld和Selinux

- 最好用root运行

## 安装机

kubespray本身可以放在集群之外， 需要提前安装上面提到的对应软件

安装python3，去https://www.python.org/ftp/python/ 下载对应安装包

```
yum install -y gcc zlib zlib-devel openssl #为了编译安装python3

tar xf Python-3.9.0.tgz
cd Python-3.9.0/ && ./configure
make
make install
```

这种方式python3和pip3一起安装好

如果后面安装太慢可以配置国内pipy源

# 单master集群安装

```
pip3 install -r requirements.txt
```

