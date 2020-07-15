# kubernetes权限控制

kubernetes的知识不在此重复细讲, 只讲点干货

- kubernetes本身的资源通过RBAC来控制( 当然还有其他方式, 这里省略)
- 外部访问kubernetes集群需要经过`认证` -> `授权` -> `准入控制` 三步, kubeconfig解决第一步, RBAC解决第二步
- kubernetes不带有用户体系, 因此多租户一直是kubernetes落地到产品中需要解决的问题
- kubectl访问kubernetes需要通过kubeconfig文件来知道集群, 用户, 命名空间等信息, 默认地址是`/root/.kube/config` , 也可以手动指定



# 环境

测试集群采用kubeadm安装, master节点上有所有证书, 证书地址是 `/etc/kubernetes/pki`



# 方案1：用户证书+kubeconfig+RBAC

通过根证书来签发用户证书， 再到集群里面配置kubeconfig来实现权限细化管理

首先根据根证书签发用户名的证书, 这里我用户名用了"czs"

```
echo {\"CN\":\"czs\", \"key\": \{\"algo\":\"rsa\", \"size\":2048\}} | cfssl gencert -ca /etc/kubernetes/pki/ca.crt -ca-key /etc/kubernetes/pki/ca.key - | cfssljson -bare czs
```

生成的下面三个文件如下:

```
 czs.csr  czs-key.pem  czs.pem
```

然后配置kubeconfig文件

```
# 设置Cluster信息
kubectl config set-cluster openstack --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true --server=https://192.168.152.100:6443

# 设置User信息
kubectl config set-credentials czs --client-certificate=/etc/kubernetes/pki/czs.pem --client-key=/etc/kubernetes/pki/czs-key.pem

# 设置context
kubectl config set-context default-context --cluster=openstack --user=czs

# 选用当前context
kubectl config use-context default-context
```

然后执行命令, 可以看到, 没权限, 这是因为我们没有创建role和绑定用户

```
[root@k8s-worker2 pki]# kubectl get po
Error from server (Forbidden): pods is forbidden: User "czs" cannot list resource "pods" in API group "" in the namespace "default"
```

然后在master节点上, 创建`Role`和`RoleBinding`, 执行下面的yaml, 创建了只能get命名空间default的role

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: czs
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

再执行上面的命令, 可以看到了

```
[root@k8s-worker2 pki]# kubectl get po
NAME                                                 READY   STATUS    RESTARTS   AGE
my-dashboard-kubernetes-dashboard-5c885b88dc-b2rnx   1/1     Running   160        2d18h
```

我们来试试几个没赋权的操作, 可以看到该User权限已经限制了

```
[root@k8s-worker2 pki]# kubectl delete po my-dashboard-kubernetes-dashboard-5c885b88dc-b2rnx
Error from server (Forbidden): pods "my-dashboard-kubernetes-dashboard-5c885b88dc-b2rnx" is forbidden: User "czs" cannot delete resource "pods" in API group "" in the namespace "default"

[root@k8s-worker2 pki]# kubectl get po -n test
Error from server (Forbidden): pods is forbidden: User "czs" cannot list resource "pods" in API group "" in the namespace "test"
```

注意:

上面我们在执行`kubectl config set-credentials`给kubeconfig设置User信息的时候, 如果使用apiserver的证书 

```
kubectl config set-credentials czs --client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
```

则会获得admin的权限

# 方案2: SA Token来设置kubeconfig权限

`serviceaccount`虽然本意是给运行在集群中服务使用APIServer用的, 但是管理用户不同的权限的时候也可以做一个很好的切入点



**本质上是我们把一个sa作为一个用户名来处理**, 首先我们创建一个 sa, 这里就用上面的 czs

```
[root@k8s-master home]# kubectl create sa czs
serviceaccount/czs created
```

创建`Role`和`RoleBinding`,  和方法1的区别就是指定是sa

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sarolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: czs
```

然后配置kubeconfig信息, 和上面的步骤就有一步不一样, 在设置User的时候

先在master节点上获取刚刚我们设置的sa对应的token值

```
[root@k8s-master home]# kubectl get secrets
NAME                                            TYPE                                  DATA   AGE
czs-token-nlklg                                 kubernetes.io/service-account-token   3      17m
default-token-d5khf                             kubernetes.io/service-account-token   3      4d6h
kubernetes-dashboard-csrf                       Opaque                                0      2d21h
kubernetes-dashboard-key-holder                 Opaque                                0      2d21h
my-dashboard-kubernetes-dashboard-certs         Opaque                                0      2d21h
my-dashboard-kubernetes-dashboard-token-4b5qd   kubernetes.io/service-account-token   3      2d21h
sh.helm.release.v1.my-dashboard.v1              helm.sh/release.v1                    1      2d21h
[root@k8s-master home]# 
[root@k8s-master home]# kubectl describe secrets czs-token-nlklg
Name:         czs-token-nlklg
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: czs
              kubernetes.io/service-account.uid: e4a909ae-e4a3-4499-8fa7-cc02293bd1a7

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjhiUXNBeU9iM3BVTmxpU0tRNVhtcjRYVlkxa1hTRC12UGxJMmxtLWw2QkEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImN6cy10b2tlbi1ubGtsZyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJjenMiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlNGE5MDlhZS1lNGEzLTQ0OTktOGZhNy1jYzAyMjkzYmQxYTciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpjenMifQ.Ys0RdwhET9VDlgXH5rA8wJgkDXX7xgW6hFmJxmxdXTbcCs_1A_S6q5fMJCN43itA6N3AKpoJxbodFy1jOGyLYWYISonNZysbZ68gxwolq50yvyALs-0Ujx5s2VckRAtqPEQNOMI7hR-LD-Lp90XFyxJF9ZUD6SIByCeUACKggM6DZmwtHpZRKCTX5hab8BJ9A2F6ZWhz6JLhKHwpj5YvmowEMc-zRH-_pcjes-N2uU14yUTEWvVng1mk6vj2gsbewBYIGl-K7pBJClctSmrl8p5N672u_Hsu9OTyWsm9pAMBfsVaVCpEGoa7LFVQZ8Aq5_EZZvqqJFXqSQET7sQ8ZA
[root@k8s-master home]# 
```

然后把token值设置到User里面

```
[root@k8s-worker2 .kube]# kubectl config set-credentials czs --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IjhiUXNBeU9iM3BVTmxpU0tRNVhtcjRYVlkxa1hTRC12UGxJMmxtLWw2QkEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImN6cy10b2tlbi1ubGtsZyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJjenMiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlNGE5MDlhZS1lNGEzLTQ0OTktOGZhNy1jYzAyMjkzYmQxYTciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpjenMifQ.Ys0RdwhET9VDlgXH5rA8wJgkDXX7xgW6hFmJxmxdXTbcCs_1A_S6q5fMJCN43itA6N3AKpoJxbodFy1jOGyLYWYISonNZysbZ68gxwolq50yvyALs-0Ujx5s2VckRAtqPEQNOMI7hR-LD-Lp90XFyxJF9ZUD6SIByCeUACKggM6DZmwtHpZRKCTX5hab8BJ9A2F6ZWhz6JLhKHwpj5YvmowEMc-zRH-_pcjes-N2uU14yUTEWvVng1mk6vj2gsbewBYIGl-K7pBJClctSmrl8p5N672u_Hsu9OTyWsm9pAMBfsVaVCpEGoa7LFVQZ8Aq5_EZZvqqJFXqSQET7sQ8ZA

```

其他步骤和上面一样, 然后测试可以发现kubectl使用User的权限已经被限制了

```
[root@k8s-worker2 .kube]# kubectl get po -n test
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:czs" cannot list resource "pods" in API group "" in the namespace "test"
```









