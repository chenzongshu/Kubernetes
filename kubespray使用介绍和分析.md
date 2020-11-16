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

# 集群安装

## 安装机

### 安装python和pip

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

### 配置免密

```
ssh-keygen
```

然后连续三次回车，生成ssh的公钥和私钥

然后配置ssh单向通道， 将公钥分发给其他机器

```
ssh-copy-id root@192.168.51.130
ssh-copy-id root@192.168.51.131
ssh-copy-id root@192.168.51.132
```

### kuberspray下载

去github下载最新代码 `https://github.com/kubernetes-sigs/kubespray`

### kuberspray安装

安装依赖

```
pip3 install -r requirements.txt
```

生成主机组

```
# 从模板拷贝一份，名字可以自己随意
cp -rfp inventory/sample inventory/mycluster

# 从inventory builder生成inventory文件
declare -a IPS=(192.168.51.130 192.168.51.131 192.168.51.132)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

```

查看集群安装参数，如果有需要， 修改

```
cat inventory/mycluster/group_vars/all/all.yml
cat inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml
```

修改阿里源的kubernetes镜像仓库

```
vim inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml
kube_image_repo: "registry.aliyuncs.com/google_containers"
```

如果需要安装日志，修改kubespray里面的ansible配置文件

```
cd ~/kubespray
echo 'log_path = /var/log/ansible.log' > ansible.cfg
```

安装Playbook

```
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml
```

### 创建具有操作权限的Dashboard账号

默认的Dashboard登陆token只有浏览权限，没有操作权限，需要创建一个admin账号

```
cd /tmp/

cat >k8s-admin.yaml<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dashboard-admin
subjects:
  - kind: ServiceAccount
    name: dashboard-admin
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl create -f k8s-admin.yaml

# 将获取token的语句存入脚本
cat >k8s-dashboard-info.sh<<EOF
#!/bin/bash
source ~/.bash_profile
kubectl cluster-info
echo -e '\n默认登陆Token:'
SecretName=\$(kubectl get secret -o wide --all-namespaces|grep namespace-controller-token|awk '{print \$2}')
kubectl describe secret \${SecretName} -n kube-system| grep ^token|awk '{print \$2}'
echo -e '\n具有操作权限的Token:'
SecretName=\$(kubectl get secret -o wide --all-namespaces|grep dashboard-admin|awk '{print \$2}')
kubectl describe secret \${SecretName} -n kube-system|grep ^token|awk '{print \$2}'
EOF

bash k8s-dashboard-info.sh
```

### 安装其他插件

如果需要安装其他插件如 nginx-ingress或者 metallb等， 可以修改`inventory/mycluster/group_vars/k8s-cluster/addons.yml` 文件中的对应组件的开关，

### 高可用部署

本身kubespray内置了一个local LB： nginx， 这个LB的做法很特别，不是给master做LB，是在每个worker节点上面部署一个nginx，然后修改kubelet的配置，让kubelet连接到APIServer的时候是连接到本地nginx，nginx再指向多个APIServer，这样就达到了高可用的目的



### 国内无法下载问题

kubectl、kubelet、kubeadm要去google下载， 国外访问不通，解决方法如下：

用nginx建一个简单的文件下载服务器，方法略，然后去github上kubernetes仓库下载对应版本二进制包，放到文件服务器下，然后修改 `roles/download/defaults/main.yml`， 修改成类似样子

```
kubelet_download_url: "http://192.168.106.95/kubelet"
kubectl_download_url: "http://192.168.106.95/kubectl"
kubeadm_download_url: "http://192.168.106.95/kubeadm"
```

`cluster-proportional-autoscaler-amd64`这个镜像阿里云上没有，必须另外找源，找到如下，然后重新打tag

```
docker pull mirrorgcrio/cluster-proportional-autoscaler-amd64:1.8.1
docker tag mirrorgcrio/cluster-proportional-autoscaler-amd64:1.8.1 registry.aliyuncs.com/google_containers/cluster-proportional-autoscaler-amd64:1.8.1
```

Ingress-nginx对应的组件阿里云上也没有，需要自己提前下载好

## 卸载

```
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root reset.yml
```

然后到每个节点执行

```
rm -rf /etc/kubernetes/
rm -rf /var/lib/kubelet
rm -rf /var/lib/etcd
rm -rf /usr/local/bin/kubectl
rm -rf /etc/systemd/system/calico-node.service
rm -rf /etc/systemd/system/kubelet.service
systemctl stop etcd.service
systemctl disable etcd.service
systemctl stop calico-node.service
systemctl disable calico-node.service
docker stop $(docker ps -q)
docker rm $(docker ps -a -q)
service docker restart
```

## 说明

### inventory

inventory有几个组

- **kube-node** : 能部署Pod节点的
- **kube-master** : 部署master组件的
- **etcd**: 部署Etcd的，注意，Etcd是以镜像方式，在kubernetes集群外单独部署的，和原生kubeadm不同
- **calico-rr** : 
- **bastion** : 配置堡垒机的（如果节点不能直达）

# 模板解析

## 主流程步骤

可以从github的安装文档可以看到， 安装集群执行的是，执行`cluster.yml`， 总共16个步骤

```
ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```

我们看看这个yaml

```
---
- name: Check ansible version
  import_playbook: ansible_version.yml

- hosts: all
  gather_facts: false
  tags: always
  tasks:
    - name: "Set up proxy environment"
      set_fact:
        proxy_env:
          http_proxy: "{{ http_proxy | default ('') }}"
          HTTP_PROXY: "{{ http_proxy | default ('') }}"
          https_proxy: "{{ https_proxy | default ('') }}"
          HTTPS_PROXY: "{{ https_proxy | default ('') }}"
          no_proxy: "{{ no_proxy | default ('') }}"
          NO_PROXY: "{{ no_proxy | default ('') }}"
      no_log: true

- hosts: bastion[0]
  gather_facts: False
  roles:
    - { role: kubespray-defaults }
    - { role: bastion-ssh-config, tags: ["localhost", "bastion"] }

- hosts: k8s-cluster:etcd
  strategy: linear
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  gather_facts: false
  roles:
    - { role: kubespray-defaults }
    - { role: bootstrap-os, tags: bootstrap-os}

- name: Gather facts
  tags: always
  import_playbook: facts.yml

- hosts: k8s-cluster:etcd
  gather_facts: False
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  roles:
    - { role: kubespray-defaults }
    - { role: kubernetes/preinstall, tags: preinstall }
    - { role: "container-engine", tags: "container-engine", when: deploy_container_engine|default(true) }
    - { role: download, tags: download, when: "not skip_downloads" }
  environment: "{{ proxy_env }}"

- hosts: etcd
  gather_facts: False
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  roles:
    - { role: kubespray-defaults }
    - role: etcd
      tags: etcd
      vars:
        etcd_cluster_setup: true
        etcd_events_cluster_setup: "{{ etcd_events_cluster_enabled }}"
      when: not etcd_kubeadm_enabled| default(false)

- hosts: k8s-cluster
  gather_facts: False
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  roles:
    - { role: kubespray-defaults }
    - role: etcd
      tags: etcd
      vars:
        etcd_cluster_setup: false
        etcd_events_cluster_setup: false
      when: not etcd_kubeadm_enabled| default(false)

- hosts: k8s-cluster
  gather_facts: False
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  roles:
    - { role: kubespray-defaults }
    - { role: kubernetes/node, tags: node }
  environment: "{{ proxy_env }}"

- hosts: kube-master
  gather_facts: False
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  roles:
    - { role: kubespray-defaults }
    - { role: kubernetes/master, tags: master }
    - { role: kubernetes/client, tags: client }
    - { role: kubernetes-apps/cluster_roles, tags: cluster-roles }

- hosts: k8s-cluster
  gather_facts: False
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  roles:
    - { role: kubespray-defaults }
    - { role: kubernetes/kubeadm, tags: kubeadm}
    - { role: network_plugin, tags: network }
    - { role: kubernetes/node-label, tags: node-label }

- hosts: calico-rr
  gather_facts: False
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  roles:
    - { role: kubespray-defaults }
    - { role: network_plugin/calico/rr, tags: ['network', 'calico_rr'] }

- hosts: kube-master[0]
  gather_facts: False
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  roles:
    - { role: kubespray-defaults }
    - { role: kubernetes-apps/rotate_tokens, tags: rotate_tokens, when: "secret_changed|default(false)" }
    - { role: win_nodes/kubernetes_patch, tags: ["master", "win_nodes"] }

- hosts: kube-master
  gather_facts: False
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  roles:
    - { role: kubespray-defaults }
    - { role: kubernetes-apps/external_cloud_controller, tags: external-cloud-controller }
    - { role: kubernetes-apps/network_plugin, tags: network }
    - { role: kubernetes-apps/policy_controller, tags: policy-controller }
    - { role: kubernetes-apps/ingress_controller, tags: ingress-controller }
    - { role: kubernetes-apps/external_provisioner, tags: external-provisioner }

- hosts: kube-master
  gather_facts: False
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  roles:
    - { role: kubespray-defaults }
    - { role: kubernetes-apps, tags: apps }
  environment: "{{ proxy_env }}"

- hosts: k8s-cluster
  gather_facts: False
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  roles:
    - { role: kubespray-defaults }
    - { role: kubernetes/preinstall, when: "dns_mode != 'none' and resolvconf_mode == 'host_resolvconf'", tags: resolvconf, dns_late: true }
```

1、范围：本地 ；

​      操作：检查ansible版本

2、范围：所有

​      操作：设置`proxy_env`的这一组fact，后面task使用

3、范围：`bastion[0]`

​      操作：引用2个role，`kubespray-defaults`和`bastion-ssh-config`，如果没有堡垒机，这步会跳过

4、范围：`k8s-cluster` 和 `etcd`

​      操作：引用2个role，`kubespray-defaults` ，`bootstrap-os`

5、导入了 `facts.yml` 的playbook

6、范围：`k8s-cluster` 和 `etcd`

​     操作：引用4个role，`kubespray-defaults` ，`kubernetes/preinstall`，`container-engine`，`download`

7、范围： `etcd` 

​     操作：引用2个role，`kubespray-defaults` ，`etcd`，附带参数为：

> ​      etcd_cluster_setup: true
>
> ​      etcd_events_cluster_setup: "{{ etcd_events_cluster_enabled }}"
>
>  when: not etcd_kubeadm_enabled| default(false)

8、范围： `k8s-cluster`

​     操作：引用2个role，`kubespray-defaults` ，`etcd`，附带参数为：

> ​        etcd_cluster_setup: false
>
> ​        etcd_events_cluster_setup: false
>
> ​      when: not etcd_kubeadm_enabled| default(false)

第7，8步都是部署Etcd，只是部署参数和范围不同， 主要在下面2个参数

- etcd_cluster_setup：
- etcd_events_cluster_setup：

9、范围： `k8s-cluster`

​     操作：引用2个role，`kubespray-defaults` ，`kubernetes/node`

10、范围： `k8s-master`

​     操作：引用4个role，`kubespray-defaults` ，`kubernetes/master`，`kubernetes/client`，`kubernetes-apps/cluster_roles`

11、范围： `k8s-cluster`

​     操作：引用4个role，`kubespray-defaults` ， `kubernetes/kubeadm`， `network_plugin`， `kubernetes/node-label`

12、范围： `calico-rr`， 一般情况下这步不会执行

​     操作：引用2个role，`kubespray-defaults` ，`network_plugin/calico/rr`

13、范围： `kube-master[0]`

​     操作：引用3个role，`kubespray-defaults` ， `kubernetes-apps/rotate_tokens`， `win_nodes/kubernetes_patch`， 

14、范围： `kube-master`

​     操作：引用5个role，`kubespray-defaults` ， `kubernetes-apps/external_cloud_controller`， `kubernetes-apps/network_plugin`， `kubernetes-apps/policy_controller`， `kubernetes-apps/ingress_controller`， `kubernetes-apps/external_provisioner`

15、范围： `kube-master`

​     操作：引用2个role，`kubespray-defaults` ，  `kubernetes-apps`

16、范围： `k8s-cluster`

​     操作：引用2个role，`kubespray-defaults` ，`kubernetes/preinstall`

## Role含义解析

### kubespray-defaults

主要做一些变量设置的工作

### bootstrap-os

- 从`/etc/os-release`或者OS类型，执行不同的yaml文件，CentOS执行操作如下：
  - 禁用掉fastestmirror的插件，加快yum速度
  - 如果定义了http proxy就加到`/etc/yum.conf`里面
  - 安装libselinux-python方便ansible操作selinux
- 建了一个`~/.ansible/tmp`文件夹
- 更新hostname
- 如果开启了`rbd_provisioner_enabled`则安装ceph-commmon
- 确保`bash_completion.d`文件夹存在

### kubernetes/preinstall

做了12步操作

- 关闭swap
- 做了很多配置项参数检测，过滤了不合法的配置项
- 检测是否有credentials相关文件
- 设置fact，后面使用
- 安装kubernetes格式建对应文件夹
- `0061-systemd-resolved.yml` 到 `0062-networkmanager.yml`都是设置域名解析，需要在特定条件才执行，
- 安装了一些repo，ipvs需要的软件，还有ansible需要的包
- 做系统参数的配置，比如selinux，ip forward等
- 配置`/etc/hosts`文件

### container-engine

以docker为例，检测内核版本，设置docker-ce仓库，安装并启动docker，设置系统变量

### download

下载二进制和镜像

### etcd

- 生成etcd证书
- 通过脚本建立etcd集群

### kubernetes/node

设置kubelet

### kubernetes/master

- 拷贝kubectl并设置权限和命令补全

- 检查kubeadm各项

- 生成kubeadm的配置文件

- 创建集群，保存token

### kubernetes/client

构建kubeconfig文件

### kubernetes-apps/cluster_roles

配置集群的ClusterRole和ClusterRoleBinding ，设置webhook （？）

### kubernetes/kubeadm

更新kubeadm token，节点join到集群

