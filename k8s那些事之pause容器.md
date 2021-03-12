# 引子

为什么要谈下pause容器？安全的同学有次来找我，问为什么docker inspect看不到容器的IP，因为他们的程序想通过`docker.sock`去拿容器的信息，但是发现没有Pod IP，下面来剖析下Kubernetes中一个特殊的存在：pause容器

# pause容器是啥子

登录到Kubernetes的worker节点，使用`docker ps`命令可以看到一堆pause容器，没接触过Kubernetes的同学可能一脸懵，我没启动这个容器啊，kubectl命令也看不到这个容器啊，pause容器是撒子嘛？

在前面的《k8s那些事之万法之源Pod》中有提到已经提到了pause容器，让我们回顾下记忆点

- pod类似于进程组
- pause容器生命周期就是Pod生命周期
- pause容器本质是资源占位的，永远是第一个启动，pod中其他容器是加入pause的namespace来实现资源共享

下面我们来唠一唠为什么这么设计

# 设计原理

先说结论，pause容器作用就是

- 做根进程，管理container生命周期，管理僵尸进程
- 作为基础资源占位符来挂载namespaces

## 做根进程

docker是的单进程模型，在docker容器的里面，你可以看到docker启动的cmd进程就是其内部pid为1的根进程，相当于docker容器的生命周期和这个根进程是一致的

而Kubernetes的pod不同于docker的单进程模型，前面文章有说是类似 进程组的模型，k8s通过pause容器来管理这些僵尸进程

```
[root@centos-bb95bf8df-w8b4d /]# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   1012     4 ?        Ss   Feb26   0:00 /pause
root         6  0.0  0.0  11680  1340 ?        Ss   Feb26   1:04 /bin/bash -c while true; do date>>/log/test.log; sleep 10;done
......
```

具体原理我们在下面pause容器源码那一节来分析

## 挂载namespace

Docker创出来的namespace都位于 /proc/{PID}/ns 下面，我们来看看pause容器和pod里面的容器的ns

```bash
[root@node3 ~]# docker ps|grep centos
230fa1f5ccfe        77dd9217eeb9                                           "/bin/sh -c 'sh -c '…"   13 days ago         Up 13 days                              k8s_filebeat_centos-bb95bf8df-w8b4d_test_96a0b488-3d35-485d-a514-cad7623be99b_0
f80c95e1342b        28fe18be341b                                           "/bin/bash -c 'while…"   13 days ago         Up 13 days                              k8s_centos_centos-bb95bf8df-w8b4d_test_96a0b488-3d35-485d-a514-cad7623be99b_0
c5049f7ed0ff        k8s.gcr.io/pause:3.1                                   "/pause"                 13 days ago         Up 13 days                              k8s_POD_centos-bb95bf8df-w8b4d_test_96a0b488-3d35-485d-a514-cad7623be99b_0
```

然后使用`docker inspect --format '{{.State.Pid}}' 230fa1f5ccfe`命令看到每个容器的PID

最后查看进程的ns， 可以看到，每个容器的ns都是一样的

```bash
[root@node3 ~]# ls -l /proc/6054/ns/
总用量 0
lrwxrwxrwx. 1 1000 1000 0 3月  11 20:52 ipc -> ipc:[4026532299]
lrwxrwxrwx. 1 1000 1000 0 3月  11 20:52 mnt -> mnt:[4026532536]
lrwxrwxrwx. 1 1000 1000 0 3月  11 20:52 net -> net:[4026532467]
lrwxrwxrwx. 1 1000 1000 0 3月  11 20:52 pid -> pid:[4026532465]
lrwxrwxrwx. 1 1000 1000 0 3月  11 20:52 user -> user:[4026531837]
lrwxrwxrwx. 1 1000 1000 0 3月  11 20:52 uts -> uts:[4026532537]
[root@node3 ~]#
[root@node3 ~]# ls -l /proc/5934/ns/
总用量 0
lrwxrwxrwx. 1 root root 0 2月  26 01:45 ipc -> ipc:[4026532299]
lrwxrwxrwx. 1 root root 0 3月  11 20:53 mnt -> mnt:[4026532218]
lrwxrwxrwx. 1 root root 0 2月  26 01:45 net -> net:[4026532467]
lrwxrwxrwx. 1 root root 0 2月  26 01:45 pid -> pid:[4026532465]
lrwxrwxrwx. 1 root root 0 3月  11 20:53 user -> user:[4026531837]
lrwxrwxrwx. 1 root root 0 3月  11 20:53 uts -> uts:[4026532220]
[root@node3 ~]#
[root@node3 ~]# ls -l /proc/6016/ns/
总用量 0
lrwxrwxrwx. 1 root root 0 3月   8 04:41 ipc -> ipc:[4026532299]
lrwxrwxrwx. 1 root root 0 3月   8 04:41 mnt -> mnt:[4026532534]
lrwxrwxrwx. 1 root root 0 3月   8 04:41 net -> net:[4026532467]
lrwxrwxrwx. 1 root root 0 3月   8 04:41 pid -> pid:[4026532465]
lrwxrwxrwx. 1 root root 0 3月  11 20:53 user -> user:[4026531837]
lrwxrwxrwx. 1 root root 0 3月   8 04:41 uts -> uts:[4026532535]
```



使用ip netns看pod的ip地址，同上面一样，可以获取容器的PID之后，

> 默认情况下，当 docker 实例被创建出来后，使用 `ip netns`  命令无法看到容器实例对应的 network namespace。这是因为 `ip netns` 命令是从 `/var/run/netns` 文件夹中读取内容的。

```
[root@node3 ~]# ln -s /proc/5934/ns/net /var/run/netns/c5049f7ed0ff
[root@node3 ~]#
[root@node3 ~]# ip netns
c5049f7ed0ff (id: 3)
[root@node3 ~]#
[root@node3 ~]# ip netns exec c5049f7ed0ff ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if35: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 3e:8d:d0:07:df:99 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.233.92.56/32 scope global eth0
       valid_lft forever preferred_lft forever
```

可以看到这个ns的ip就是pod的ip

# pause源码

pause容器内容特别简单，就是一个c语言编写的死循环

```c
#define STRINGIFY(x) #x
#define VERSION_STRING(x) STRINGIFY(x)

#ifndef VERSION
#define VERSION HEAD
#endif

static void sigdown(int signo) {
  psignal(signo, "Shutting down, got signal");
  exit(0);
}

static void sigreap(int signo) {
  while (waitpid(-1, NULL, WNOHANG) > 0)
    ;
}

int main(int argc, char **argv) {
  int i;
  for (i = 1; i < argc; ++i) {
    if (!strcasecmp(argv[i], "-v")) {
      printf("pause.c %s\n", VERSION_STRING(VERSION));
      return 0;
    }
  }

  if (getpid() != 1)
    /* Not an error because pause sees use outside of infra containers. */
    fprintf(stderr, "Warning: pause should be the first process\n");

  if (sigaction(SIGINT, &(struct sigaction){.sa_handler = sigdown}, NULL) < 0)
    return 1;
  if (sigaction(SIGTERM, &(struct sigaction){.sa_handler = sigdown}, NULL) < 0)
    return 2;
  if (sigaction(SIGCHLD, &(struct sigaction){.sa_handler = sigreap,
                                             .sa_flags = SA_NOCLDSTOP},
                NULL) < 0)
    return 3;

  for (;;)
    pause();
  fprintf(stderr, "Error: infinite loop terminated\n");
  return 42;
}
```

- 核心是一个死循环 `for (;;)`表示它处于休眠状态，等待信号量将其唤醒

- 收到`SIGCHLD`信号的时候调用`sigreap`函数里面的`waitpid`去收割一波僵尸进程