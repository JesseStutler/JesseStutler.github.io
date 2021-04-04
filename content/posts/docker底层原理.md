---
title: "Docker底层原理"
date: 2021-03-07T11:12:39+08:00 
description: "对于docker底层原理的分析"
tags: [docker]
featured_image: "/images/docker-logo.jpg"
images: ["/images/docker-logo.jpg"]
categories: docker
comment : true
---



# docker底层原理

## namespace——资源隔离

namespace 是 Linux 内核用来隔离内核资源的方式，同一个namespace中的进程之间可以互相感知，不同namespace之间的进程是相互独立的，**docker本身就是一个进程，通过namespace来实现隔离，从而模拟独立运行环境**，在/proc/$$/ns下能查看当前进程下的所有link文件，每个link文件对应不同的namespace，**如果不同的进程间有相同的namespace的inode号，则他们是共享namespace的**，否则他们属于不同的的namespace

![img](https://tva1.sinaimg.cn/large/008eGmZEgy1gob6e80y1pj30i904dglp.jpg)

- **通过clone()函数在创建子进程的同时给子进程创建新的namespace（传入CLONE_*宏定义）**

- UTS namespace：

  提供**主机名和域名**的隔离，使容器能够通过服务名访问

- IPC namespace（进程间通信）：

  实现**信号量、消息队列和共享内存等资源**的隔离

- PID namespace：

  对进程内的PID重新标号，从1开始，每个PID namespace都有自己的计数程序，宿主机的PID namespace相当于创建进程的PID namespace的parent pid namespace，能看到子节点（child pid namespace)中的内容，但子节点不能看到父节点当中的内容，这样父节点能在外部管理容器内的进程

  如果pid namespace中的某个进程的父进程被杀死，该进程成为孤儿进程，则**会被当前pid namespace的init进程（pid为1，如/bin/bash）收养，成为其子进程**

- mount namespace:

  通过隔离文件系统挂载点来隔离文件系统（当创建新的mount namespace时，会将所有挂载点复制给子进程，但在这之后，子进程对自己namespace内文件系统进行的操作不会影响到父进程namespace）

  **可以通过共享挂载机制传播挂载（主从挂载、共享挂载等）**

- network namespace:

  提供网络资源的隔离，包括网络设备、协议栈、路由表、防火墙等等

![img](https://images2018.cnblogs.com/blog/1259802/201804/1259802-20180410165500455-232801094.jpg)



**默认的bridge模式**：

每个容器有独立的network namespace，宿主机通过docker0网桥（虚拟网桥）来连接不同的network namespace，容器通过veth pair（虚拟以太网端口对，它们组成了一个数据的通道，数据从一个设备进入，就会从另一个设备出来）连接docker0网桥，设备的一端放在新创建的容器中，并命名为eth0。另一端放在主机中，以veth65f9这样类似的名字命名。

如果容器想主动和外界通信，或者外界想访问容器内的服务（访问宿主机的端口），实际上这是通过iptables来管理的（进行了转发和NAT转换等操作）

**host模式：**

容器和宿主机共享network namespace，但其他namespace与宿主机隔离，容器用的是宿主机的ip与外界通信，性能较好但易产生端口冲突

**container模式：**

新创建的容器若指定container模式，则和已经存在的容器共享一个network namespace，与此容器共享ip和协议栈



- user namespace

  提供安全隔离，比如用户id，用户组，权限等，在子进程的user namespace中拥有新的用户和用户组，在父进程中的普通用户可能却成为子进程中namespace的超级用户，结构与pid namespace类似（树状结构），**子user namespace中的用户和用户组需要与父user namespace中的用户和用户组相对应（做映射）**，这样这个user namespace才能与其他user namespace中的进程通信，甚至访问共享的文件（即对应到其他user namespace的用户和用户组并拥有相应的权限，如果没有相应的权限就不能在其他user namespace执行某些操作）



## cgroups——资源限制

cgroups 是Linux内核提供的一种**可以限制单个进程或者多个进程所使用资源**的机制，可以对 cpu，内存等资源实现精细化的控制（使用上限，使用范围等等），cgroup通过伪文件系统的形式进行控制

**涉及概念：**

- task（任务）：表示系统的一个进程或线程

- subsystem(子系统)：每个子系统就是一个资源控制器，有cpu、memory、io等等，/sys/fs/cgroup/下的目录就代表每个子系统

- hierarchy（层级）：层级由一系列的cgroup以一个树状形式组成，由一个或多个子系统限制层级的资源使用量

- **cgroup**（控制组）：核心概念，由cgroup组成层级，task放在cgroup中，从而控制进程的使用资源量，docker的实现方法就是在每个子系统中为每个容器创建cgroup

  

  ps:笔者自己ipad画的图，有点简陋抱歉:P，以后文章的图有时间都会用Graffle好好画一下

![image-20201104214438717](/Users/chenzicong/Library/Application Support/typora-user-images/image-20201104214438717.png)

## docker 架构总览



![image-20201209104859776](https://tva1.sinaimg.cn/large/008eGmZEgy1gob68s57bpj311c0hm7ax.jpg)



### runC

是对于OCI标准的一个参考实现，是一个可以用于创建和运行容器的CLI(command-line interface)工具。runc直接与容器所依赖的cgroup/linux kernel等进行交互，负责为容器配置cgroup/namespace等启动容器所需的环境，创建启动容器的相关进程。runC基本上就是一个命令行小工具，它可以不用通过Docker引擎，直接就可以创建容器。这是一个独立的二进制文件，使用OCI容器就可以运行它。

### containerd

containerd 是一个守护进程，它可以使用runC管理容器，并使用gRPC暴露容器的其他功能。docker engine面向client，containerd暴露出针对容器的增删改查的接口，Docker engine通过gRPC调用这些接口完成对于容器的操作，containerd最后会通过runc来实际运行容器。

### containerd-shim

containerd-shim称之为垫片，它使用runC命令行工具完成容器的启动、停止以及容器运行状态的监控。containerd-shim进程由containerd进程拉起，并持续存在到容器实例进程退出为止（和容器进程同生命周期）。这种设计的优点是，只要是符合OCI规范的容器，都可以通过containerd-shim来进行调用



## 联合文件系统



![Layers of a container based on the Ubuntu image](https://docs.docker.com/storage/storagedriver/images/container-layers.jpg)

**如果容器基于同一镜像构建，所有容器共享底部的镜像层，镜像层是read-only的，不可被修改，容器只是在镜像层之上创建了一个读写层，所有容器的修改都是在读写层当中修改，并不会影响到底部的镜像层，当容器删除了，读写层也就跟着删除了（除非commit做成了新的镜像）**

**就算容器不是基于同一镜像构建，如果不同镜像中有相同的层（比如FROM是相同的），容器也会读取同一镜像层，这就是联合文件系统的精髓所在，副本只保存一份**



![Containers sharing same image](https://docs.docker.com/storage/storagedriver/images/sharing-layers.jpg)



## Copy on write（COW）机制



**当容器需要读取文件的时候**

从最上层镜像开始查找，往下找，找到文件后读取并放入内存，若已经在内存中了，直接使用。(即，同一台机器上运行的docker容器共享运行时相同的文件)。

**当容器需要添加文件的时候**

直接在最上面的容器层可写层添加文件，不会影响镜像层。

**当容器需要修改文件的时候**

从上往下层寻找文件，找到后，**复制到容器可写层**，然后，对容器来说，可以看到的是容器层的这个文件，看不到镜像层里的文件。容器在容器层修改这个文件（也就是覆盖）。

**如果是一个经常需要写的应用，最好使用volume而不是都写在容器层里，这样不会使容器层变得很大**



**当容器需要删除文件的时候**

从上往下层寻找文件，找到后在容器中记录删除。即，**并不会真正的删除文件，而是软删除**。这将导致镜像体积只会增加，不会减少。**所以要写Dockerfile时如果要删除镜像中的文件，最好在同一层删除，否则只是软删除**（因为如果真正的删除就会导致基于这个镜像构架内的其他容器无法再读取这个文件了）





## iptables默认规则（只列举了部分）

- nat表：
  - 如果数据源地址是docker0网段地址，且发往除docker 0外端口（即发往主机外），则做SNAT转换，修改为主机网卡ip地址
- filter表：
  - docker0发出的包可以中转给docker 0本身，即容器之间可以互相通信
  - docker0发出的包可以中转给其他宿主机上的其他网卡



## 参考资料

- 《Docker容器与容器云 第2版》浙江大学SEL实验室·著
- https://www.jianshu.com/p/517e757d6d17
- https://www.cnblogs.com/sparkdev/tag/docker/default.html?page=1