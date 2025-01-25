---
title: 【Docker】容器的本质
date: 2024-04-30 19:54:06
categories: Docker
tags: Docker

---

一个正在运行的 Docker 容器，其实就是一个启用了多个 Linux Namespace 的应用进程，而这个进程能够使用的资源量，则受 Cgroups 配置的限制。

下面通过一个小实验证明这句话

<!-- more --> 

### 验证

在 Linux 中，Cgroups 给用户暴露出来的操作接口是文件系统，即它以文件和目录的方式组织在操作系统的 /sys/fs/cgroup 路径下。

```
mount -t cgroup 
```

进入到 /sys/fs/cgroup/cpu 目录下，并创建一个目录，操作系统会在新创建的 container 目录下，自动生成该子系统对应的资源限制文件。

```
root@ubuntu:/sys/fs/cgroup/cpu$ mkdir container
root@ubuntu:/sys/fs/cgroup/cpu$ ls container/
cgroup.clone_children cpu.cfs_period_us cpu.rt_period_us  cpu.shares notify_on_release
cgroup.procs      cpu.cfs_quota_us  cpu.rt_runtime_us cpu.stat  tasks
```

执行一个脚本，打满CPU

```
while : ; do : ; done &
```

修改container目录中的CPU资源限制并把刚刚创建的进程加入到任务组中

```
echo 20000 > /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us

echo PID > /sys/fs/cgroup/cpu/container/tasks 
```

这时候再通过TOP去查看CPU使用就会发现它的使用率从100%降到了20%

[![img](https://raw.githubusercontent.com/Alexhuihui/photo/main/20240430101156.png)](https://raw.githubusercontent.com/Alexhuihui/photo/main/20240430101156.png)

### 缺点

docker使用namespace进行隔离但是隔离的不彻底，它本质上还是一个跑在宿主机上的进程。

docker使用cgroups进行使用资源的限制，但是Cgroups 对资源的限制能力也有很多不完善的地方。