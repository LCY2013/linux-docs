## k8s

### docker

```dockerfile
docker 容器的实现机理，来自于Linux下的Cgroup和NameSpace
NameSpace : 
linux 创建线程来自于系统调用的clone()
eg: int pid = clone(main_function,stack_size,SIGCHLD,NULL);
这里第三个参数可以指定CLONE_NEWPID | SIGCHLD,所以这里面的PID可以是1，但是它在宿主机上的PID还是真实的，比如: 8846
NameSpace 分类: Mount,UTS,IPC,Network,User
容器其实就是一种特殊的进程

Cgroup : (Linux control group)
Cgroup 是Linux用来给进程设置资源限制的一个重要功能,限制资源包括(CPU,网络,磁盘,网络带宽等等)
mount -t cgroup
blkio: 为快设备设置I/O限制，一般用于磁盘设备。    
cpu,cpuacct  
freezer  
net_cls           
perf_event
cpu      
cpuset: 为进程分配单独的CPU核和对应的内存节点。       
hugetlb  
net_cls,net_prio  
pids
cpuacct  
devices      
memory: 为进程设置内存限制。   
net_prio          
systemd

docker 容器参数限定: docker run -it --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/sh


ls /sys/fs/cgroup/cpu
cgroup.clone_children  cpu.rt_period_us      notify_on_release
cgroup.procs           cpu.rt_runtime_us      tasks
cpu.cfs_period_us      cpu.shares
cpu.cfs_quota_us       cpu.stat
其中cpu.cfs_period_us 和 cpu.cfs_quota_us这两个参数组合可以用来限制进程在长度为cfs_period的一段时间内，只能被分配到总量为cfs_quota的CPU时间
如何使用这样的配置文件？
	1、进入/sys/fs/cgroup/cpu
	2、mkdir container
	3、ls container 这时会发现在container目录被创建完成后，Linux就会自动生成对应的资源限制文件
	4、执行一个脚本 while : ; do : ; done & , CPU 被打满,然后找到这个PID杀掉这个进程
	5、查看cpu.cfs_quota_us -1(即不存在任务限制) cpu.cfs_period_us 1000000 (即100ms 100000us)
  6、echo 20000 > cpu.cfs_quota_us 意思每100ms只能20ms的CPU执行
  7、将需要被限制的PID写入到container组的tasks中 echo PID > /sys/fs/cgroup/cpu/container/tasks 然后就会生效

docker run imageName
docker run -it busybox /bin/sh  -it就是告诉容器分配一个TTY，文本输入输出环境

容器化与虚拟化相比: 敏捷和高性能,但是隔离不测底
```

### docker swarm

```dockerfile
docker run -H "swarm 集群API地址" imageName
#docker 收购 fig 诞生容器编排"Container Orchestration"，这时docker compose 诞生了
```

### mesos 

```markdown
1、mesos + marathon 组合
```



### Linux 下clone系统调用

```c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

/* 定义一个给 clone 用的栈，栈大小1M */
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];

char* const container_args[] = {
    "/bin/bash",
    NULL
};

int container_main(void* arg)
{
    printf("Container - inside the container!\n");
    /* 直接执行一个shell，以便我们观察这个进程空间里的资源是否被隔离了 */
    execv(container_args[0], container_args); 
    printf("Something's wrong!\n");
    return 1;
}

int main()
{
    printf("Parent - start a container!\n");
    /* 调用clone函数，其中传出一个函数，还有一个栈空间的（为什么传尾指针，因为栈是反着的） */
    int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWNS | SIGCHLD, NULL);
    /* 等待子进程结束 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

编译上面的程序gcc -o ns ns.c

执行 文件 ./ns

查看 ls /tmp

声明要启用 Mount Namespace 之外，我们还可以告诉容器进程，有哪些目录需要重新挂载

```c
int container_main(void* arg)
{
  printf("Container - inside the container!\n");
  // 如果你的机器的根目录的挂载类型是 shared，那必须先重新挂载根目录
  // mount("", "/", NULL, MS_PRIVATE, "");
  mount("none", "/tmp", "tmpfs", 0, "");
  execv(container_args[0], container_args);
  printf("Something's wrong!\n");
  return 1;
}
```

mount(“none”, “/tmp”, “tmpfs”, 0, “”) 语句,就这样，我告诉了容器以 tmpfs（内存盘）格式，重新挂载了 /tmp 目录。

这次 /tmp 变成了一个空目录。

mount -l | grep tmpfs 容器里的 /tmp 目录是以 tmpfs 方式单独挂载的。

docker run -d ubuntu:latest sleep 3600

docker image inspect ubuntu:latest  查看ubuntu镜像的信息







