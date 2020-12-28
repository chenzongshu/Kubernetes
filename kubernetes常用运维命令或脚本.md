# 一次save节点上所有镜像

```
[root@k8s-m1 images]# cat image-save.sh 
#!/bin/bash

docker images > images.txt
awk '{print $1}' images.txt > images_cut.txt
sed -i '1d' images_cut.txt
while read LINE
do
docker save $LINE > ${LINE//\//_}.tar
echo ok
done < images_cut.txt
rm -f *.txt
echo finish
```

# 一次load所有镜像

```
[root@k8s-m1 images]# cat image-load.sh 
#!/bin/bash

ls | grep $1 | while read line
do
   echo "the docker images tar file is : ${line}"
   docker load -i ${line}
done

# 执行时候带参数，读取对应格式的镜像
load-load.sh tar
```

# 一次所有镜像打tag并上传到某个仓库

```
#!/bin/bash

readonly new_repo=registry.cn-shenzhen.aliyuncs.com/kubespray_pub

for image in $(docker images --format '{{.Repository}}:{{.Tag}}'); do
	name=${image##*/}
	new_img=${new_repo}/${name}
	echo "Processing ${image} -> ${new_img}"
	docker tag ${image} ${new_img}
	docker push ${new_img}
done
```

# 批量删除镜像

```
# 按条件筛选之后删除镜像
docker rmi `docker images | grep xxxxx | awk '{print $3}'`
docker rmi `docker images --format '{{.Repository}}:{{.Tag}}'| grep xxxxx`

# 直接删除所有镜像
docker rmi `docker images -q`
```

# 清理非 Running 的 pod

```
kubectl get pod -o wide --all-namespaces | awk '{if($4!="Running"){cmd="kubectl -n "$1" delete pod "$2; system(cmd)}}'
```

# 进入容器 netns

（该命令就是本质就是使用nsenter）粘贴脚本到命令行:

```bash
function e() {
    set -eu
    ns=${2-"default"}
    pod=`kubectl -n $ns describe pod $1 | grep -A10 "^Containers:" | grep -Eo 'docker://.*$' | head -n 1 | sed 's/docker:\/\/\(.*\)$/\1/'`
    pid=`docker inspect -f {{.State.Pid}} $pod`
    echo "entering pod netns for $ns/$1"
    cmd="nsenter -n --target $pid"
    echo $cmd
    $cmd
}
```

进入在当前节点上运行的某个 pod 的 netns:

```bash
# 进入 kube-system 命名空间下名为 metrics-server-6cf9685556-rclw5 的 pod 所在的 netns
e metrics-server-6cf9685556-rclw5 kube-system
```

进入 pod 的 netns 后就使用节点上的工具在该 netns 中做操作，比如用 `ip a` 查询网卡和ip、用 `ip route` 查询路由、用 tcpdump 抓容器内的包等。

# 输出各节点占用的资源

## 表格输出各节点占用的 podCIDR

```bash
kubectl get no -o=custom-columns=INTERNAL-IP:.metadata.name,EXTERNAL-IP:.status.addresses[1].address,CIDR:.spec.podCIDR
```

示例输出:

```
INTERNAL-IP     EXTERNAL-IP       CIDR
10.100.12.194   152.136.146.157   10.101.64.64/27
10.100.16.11    10.100.16.11      10.101.66.224/27
10.100.16.24    10.100.16.24      10.101.64.32/27
10.100.16.26    10.100.16.26      10.101.65.0/27
10.100.16.37    10.100.16.37      10.101.64.0/27
```

## 表格输出各节点总可用资源 (Allocatable)

```bash
kubectl get no -o=custom-columns="NODE:.metadata.name,ALLOCATABLE CPU:.status.allocatable.cpu,ALLOCATABLE MEMORY:.status.allocatable.memory"
```

示例输出：

```
NODE       ALLOCATABLE CPU   ALLOCATABLE MEMORY
10.0.0.2   3920m             7051692Ki
10.0.0.3   3920m             7051816Ki
```

## 输出各节点已分配资源的情况

所有种类的资源已分配情况概览：

```bash
kubectl get nodes --no-headers | awk '{print $1}' | xargs -I {} sh -c "echo {} ; kubectl describe node {} | grep Allocated -A 5 | grep -ve Event -ve Allocated -ve percent -ve --;"
```

示例输出:

```
10.0.0.2
  Resource           Requests          Limits
  cpu                3040m (77%)       19800m (505%)
  memory             4843402752 (67%)  15054901888 (208%)
10.0.0.3
  Resource           Requests   Limits
  cpu                300m (7%)  1 (25%)
  memory             250M (3%)  2G (27%)
```

表格输出 cpu 已分配情况:

```bash
kubectl get nodes --no-headers | awk '{print $1}' | xargs -I {} sh -c 'echo -n "{}\t" ; kubectl describe node {} | grep Allocated -A 5 | grep -ve Event -ve Allocated -ve percent -ve -- | grep cpu | awk '\''{print $2$3}'\'';'
```

示例输出：

```bash
10.0.0.2	3040m(77%)
10.0.0.3	300m(7%)
```

# 线程数排名统计

```bash
printf "    NUM  PID\t\tCOMMAND\n" && ps -eLf | awk '{$1=null;$3=null;$4=null;$5=null;$6=null;$7=null;$8=null;$9=null;print}' | sort |uniq -c |sort -rn | head -10
```

示例输出:

```bash
    NUM  PID		COMMAND
    594  14418        java -server -Dspring.profiles.active=production -Xms2048M -Xmx2048M -Xss256k -Dinfo.app.version=33 -XX:+UseG1GC -XX:+UseStringDeduplication -XX:+PrintGCDateStamps -verbosegc -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=10011 -Xloggc:/home/log/gather-server/gather-server-gac.log -Ddefault.client.encoding=UTF-8 -Dfile.encoding=UTF-8 -Dlogging.path=/home/log/gather-server -Dserver.tomcat.accesslog.directory=/home/log/gather-server -jar /home/app/gather-server.jar --server.port=8080 --management.port=10086
    449  7088        java -server -Dspring.profiles.active=production -Dspring.cloud.config.token=nLfe-bQ0CcGnNZ_Q4Pt9KTizgRghZrGUVVqaDZYHU3R-Y_-U6k7jkm8RrBn7LPJD -Xms4256M -Xmx4256M -Xss256k -XX:+PrintFlagsFinal -XX:+UseG1GC -XX:+UseStringDeduplication -XX:MaxGCPauseMillis=200 -XX:MetaspaceSize=128M -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=10011 -verbosegc -XX:+PrintGCDateStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=15 -XX:GCLogFileSize=50M -XX:AutoBoxCacheMax=520 -Xloggc:/home/log/oauth-server/oauth-server-gac.log -Dinfo.app.version=12 -Ddefault.client.encoding=UTF-8 -Dfile.encoding=UTF-8 -Dlogging.config=classpath:log4j2-spring-prod.xml -Dlogging.path=/home/log/oauth-server -Dserver.tomcat.accesslog.directory=/home/log/oauth-server -jar /home/app/oauth-server.jar --server.port=8080 --management.port=10086 --zuul.server.netty.threads.worker=14 --zuul.server.netty.socket.epoll=true
.......
```

- 第一列表示线程数
- 第二列表示进程 PID
- 第三列是进程启动命令

# Dig命令

使用 `dnsutils` 镜像，或者自己做一个