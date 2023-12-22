# 节点列表来加标签和污点

```bash
#!/bin/bash

nodelist=(node1 node2)

for node in ${nodelist[@]}
do
   kubectl label node $node test=test
   kubectl taint node $node test=test:NoSchedule
   echo " the node $node label "
done
```



# 节点组设置不可调度

```bash
#!/bin/bash

## label the node unschedulable ##
for nodes in `kubectl get nodes -l alibabacloud.com/nodepool-id=npxxxxxx|awk 'NR == 1 {next} {print $1}'`
do
   kubectl cordon $nodes
   echo " the node $nodes cordon "
done
```



# 删除指定namespace里面某种状态的pod

其中Init可以改成自己需要指定的值

```bash
##!/bin/bash
nss=(ns1 ns2)
for ns in ${nss[@]}
do
  echo "**************** namespace [$ns]  *********************"

  for po in `kubectl -n $ns get po|grep Init|awk 'NR == 1 {next} {print $1}'`
  do
    echo $po
    kubectl -n $ns delete po $po
  done
done
```



# 获取指定节点上某个APP的Pod

```bash
#!/bin/bash
nodelist=(node1 node2 node3 ....)

appid=xxxxx

## label the node unschedulable ##
for nodes in ${nodelist[@]}
do
    echo "-------  node $nodes -----"
    kubectl describe node $nodes|grep $appid
    echo "*************************"
    echo "                         "
done
```



# 获取指定节点组里面指定namespace的Pod

```bash
#!/bin/bash

for nodes in `kubectl get nodes -l alibabacloud.com/nodepool-id=xxx1|awk 'NR == 1 {next} {print $1}'`
do
  kubectl describe node $nodes|grep -E 'ns1|ns2|ns3'
done

for nodes in `kubectl get nodes -l alibabacloud.com/nodepool-id=xxx2|awk 'NR == 1 {next} {print $1}'`
do
  kubectl describe node $nodes|grep -E 'ns1|ns2|ns3'
done
```



# 获取指定APP的指定信息

```bash
#!/bin/bash
javas=(app1 app2 app3)

## label the node unschedulable ##
for java in ${javas[@]}
do
    kubectl -n xxxx get deploy $java --output=custom-columns="NAME:.metadata.name,NUMS:.spec.replicas,CPU:.spec.template.spec.containers[*].resources.limits.cpu,MEM:.spec.template.spec.containers[*].resources.limits.memory"|awk 'NR > 1'
done
```



# 获取没有配置PDB的APPlist

```bash
##!/bin/bash
nss=(ns1 ns2 ns3)
for ns in ${nss[@]}
do
  echo "**************** namespace [$ns]  *********************"

  for deploy in `kubectl -n $ns get deploy|grep -v "0/"|grep -v "\-daemon"|grep -v "\-cron"|grep -v "\-v"|awk 'NR == 1 {next} {print $1}'`
  do
    result=`kubectl -n $ns get pdb $deploy|grep -v Error`
    if [[ $result == "" ]]; then
      echo $deploy
    fi
  done
done
```



# 获取指定APP里面从指定时间之后启动的Pod

```bash
#!/bin/bash
javas=(app1 app2 app3....)

## label the node unschedulable ##
for java in ${javas[@]}
do
    echo $java
    kubectl -n xxx get po $java -o yaml|grep startedAt|grep "2022-11-08"
done
```



# 给所有节点安装某个二进制程序

```bash
#!/bin/bash

for nodes in `kubectl get nodes |awk 'NR == 1 {next} {print $1}'`
do
   kubectl node-shell $nodes -- sh -c 'curl -sL http://xxxx:8080/downloadfile?filename=xxxxx.sh| bash'
   echo " the node $nodes installed "
done
```



