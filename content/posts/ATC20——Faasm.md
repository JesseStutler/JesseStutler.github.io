---
title: "ATC20——Faasm"
date: 2021-05-18T14:43:26+08:00 
description: "serverless论文总结系列#4"
tags: [serverless,论文总结系列]
featured_image: "/images/serverless.jpg"
images: ["/images/serverless.jpg"]
categories: serverless
comment : true
---

# ATC20——Faasm

这是一篇关于webassembly sandbox的文章

论文链接：https://www.usenix.org/conference/atc20/presentation/shillaker

## 为什么要提出WASM-sandbox（本文是Faaslet)？

大多数serverless平台使用的是容器承载function，但是对于容器来说，启动开销和过多的memory footprint仍然与serverless场景不太匹配（像边缘场景如果容器是overprovision的，性能会随着资源可用量的减少而下降；而且边缘如果是多租户的，long-running container也不合需求，如果资源不够用了需要频繁的驱逐），而且现有以容器为承载的方案（尽管有提出本地存储来减少访问数据开销的）会产生**冗余数据**，每个函数都有一份拷贝，而且需要重复的序列化和网络开销。对比docker来说，faaslet能够极大的减少冷启动延迟，减少开销，让一台机器承载更多的sandbox

## Fasslet

![截屏2021-05-10 下午7.21.46](https://tva1.sinaimg.cn/large/008i3skNgy1gqdjwbsburj30p009umyy.jpg)

1. function和其library，runtime都会编译为WASM；

2. cgroup做cpu周期隔离；
3. network namespace做Network隔离和提供virtual network interface；
4. faaslet以线程运行，共享进程资源；
5. 部分WASI+部分POSIX实现（图中Host interface）做system calls，因为WASI是基于compability-based security的，所以对于资源的访问是通过不可伪造的句柄来保存引用的

- 两层状态共享机制

<img src="/Users/chenzicong/Library/Application Support/typora-user-images/截屏2021-05-10 下午7.24.07.png" alt="截屏2021-05-10 下午7.24.07"  />

local状态共享就是多个faaslet（多个线程）共享父进程的同一块内存区域，global负责集群内状态的同步

- faaslet快照

预初始化faaslet并做成快照，可以减少冷启动时间和各种开销



