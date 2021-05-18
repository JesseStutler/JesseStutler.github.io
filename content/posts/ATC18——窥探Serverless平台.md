---
title: "ATC18——窥探Serverless平台"
date: 2021-05-18T14:31:18+08:00 
description: "serverless论文总结系列#2"
tags: [serverless,论文总结系列]
featured_image: "/images/serverless.jpg"
images: ["/images/serverless.jpg"]
categories: serverless
comment : true
---

# ATC18——窥探serverless平台

这是一篇利用逆向工程测试Serverless平台的文章

论文链接：https://www.usenix.org/conference/atc18/presentation/wang-liang

## QA

Q:隔离的减少会导致I/O，networking，coldstart等表现的下降？

A:是的。如果VM中有多个instance实例会造成资源争夺的现象（见衡量指标中的I/O & network throughput）



Q:同一VM是运行多个function instance吗？

A：是。AWS可以通过I/O测试发现多个function instance共享/proc中的文件



Q:不同租户的function实例可以放到同一VM里吗？

A：可以但并没有平台采用（安全隔离性会有问题:side channel attack）



Q:idle instance（暂无请求的实例，但是不会收用户费用）是要退出并收回资源还是再利用（先放到池里）处理后续的请求？

https://aws.amazon.com/cn/blogs/compute/container-reuse-in-lambda/

A:这个问题值得考量。一方面，idle instance会一直占用VM的资源；而另一方面，如果有突发的请求又可以减少instance的冷启动时间。所以折中来说，AWS采用的是将一个函数的一半的instance每300s停掉并回收资源，剩下的instance运行直到一个最大idle time为止。



Q:多个request会被同个instance接收吗？

A：会。Google针对负载过多的话会开新实例



Q:function update是开新实例吗，还是在旧实例的基础上改？

A：开新实例，负载会从旧实例慢慢过渡到新实例，但是有一个时间差

## 衡量指标

- Cold-start latency（& warm start latency）

  这里代表的是function的冷启动时间（AWS使用了VM池在function启动之前就准备接收function的调度，这样基本上只受scheduling latency的影响）

  warm start指function在执行完之后，暂时“冻住”，为不久后再有请求而“解冻”并处理

- Function instance lifetime

  即使instance仍然在运行，但是到达一个lifetime也会被terminated，租户如果想用一个function维护in-memory state的话肯定想让这个instance运行地更久一点

  AWS instance lifetime中位数为6.2小时

- Maximum idle time before shut down

- I/O & network throughput

  当VM中的function实例越多，每个function的I/O和network吞吐量会越小，而且会受到function分配到的内存的影响，function占用的内存越大，吞吐量越高。所以，存在一个**VM多instance的资源争用问题**

- CPU usage（AWS是根据code的预配置memory量来分配cpu周期，memory量越多CPU周期越多，这样冷启动的时间也会减少越多，而且公平）

- Memory usage



## 可以利用点

### 优化调度

- AWS尝试将function调度视为一个装箱问题(bin-packing problem)，尽可能的将新生成的function实例装入已有的VM实例当中，以提高==内存利用率==。**调度与function code无关**。但是，这也会引入**instance的资源争用问题**

- 如果function update的话可能会造成新一轮请求仍然被旧实例（可能是旧的函数的新实例，也可能是未被shut down的旧函数的旧实例）处理，怎么优化调度器？
- 既然冷启动时间用VM就绪池的方法可以减少VM启动的这一部分latency，如何再减少scheduling latency？

### 优化冷启动时间

- 减少scheduling latency
- 使用library caching减少函数库的加载时间，只加载需要的库（Unikernel）

### 优化隔离性

- 更多的创造instance新实例而不是使用旧实例（符合serverless理念）
- Short-lived instance
- small memory-footprint functons