# service访问超时

建了一个tomcat的RC, 然后给它建了一个service, 

```
[root@vm1 k8s]# kubectl get svc -o wide
NAME         CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE       SELECTOR
kubernetes   10.254.0.1       <none>        443/TCP    61d       <none>
webapp       10.254.164.136   <none>        8080/TCP   18m       app=webapp
```

在work node访问超时

```
[root@vm2 ~]# curl 10.254.164.136:8080
curl: (7) Failed connect to 10.254.164.136:8080; 连接超时
```

看service和endpoint正常, 查看iptables规则没有发现service ip, 于是怀疑是防火墙或者selinux作怪, 查看, 果然selinux打开状态

关闭之后查看iptables, 多了service的规则

```
[root@vm2 ~]# iptables -L -v -t nat
......
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  any    any     anywhere             10.254.0.1           /* default/kubernetes:https cluster IP */ tcp dpt:https
    0     0 KUBE-SVC-BL7FHTIPVYJBLWZN  tcp  --  any    any     anywhere             10.254.164.136       /* default/webapp: cluster IP */ tcp dpt:webcache
......
```

就能连通service了

```
[root@vm2 ~]# curl 10.254.164.136:8080
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8" />
        <title>Apache Tomcat/8.5.33</title>
        <link href="favicon.ico" rel="icon" type="image/x-icon" />
.......
```


