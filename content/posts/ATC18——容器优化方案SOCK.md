---
title: "ATC18——容器优化方案SOCK"
date: 2021-05-18T14:41:22+08:00 
description: "serverless论文总结系列#3"
tags: [serverless,论文总结系列]
featured_image: "/images/serverless.jpg"
images: ["/images/serverless.jpg"]
categories: serverless
comment : true
---

# ATC18——容器优化方案SOCK 

这是一篇优化容器冷启动的文章

论文链接：https://www.usenix.org/conference/atc18/presentation/oakes

## 解构container（docker瓶颈）

- Bind mount可能比AUFS（或overlay）性能更好
- 频繁的container创建和删除（涉及到频繁的namespace的创建和删除，可能会有性能瓶颈，比如network namespace，并发的creation和cleanup越多延迟越高），是不是可以把一些不必要的namespace隔离给剔除或者进行一些优化（disable创建或删除时不必要的影响性能的功能）
- 频繁的创建和删除cgroup不如reuse cgroup，比如维护一个初始化好的cgroup池
- 当host上挂载的越多，mount namespace拷贝的速度就越慢，简单的做法可以考虑使用chroot



## SOCK优化方案

- Lean containers:

  - 用bind mount代替overlay，分四层：系统层（base），package层（read-only，用来package caching），code层（lambda代码），scratch层（就是container layer，可写层）
  - 用cgroup pool来分配在container创建时分配cgroup，container删除时重新回到池中
  - 将mount namespace和network namespace省去，其瓶颈在docker瓶颈中已提到

- Zygote机制：

  Zygote container就是一些已经预import需要的package的容器，内含Zygote进程，这样从这个进程fork出的新进程（子进程）并创建出的新容器不需要做重复性的初始化工作，直接从内存读相同的内容就好了，也就是：

  **含Zygote进程的container-->含从Zygote fork出的子进程的container**

- 三级缓存：

  - handler cache：

    将idle instance pause，不消耗cpu但是消耗内存，之后再有request过来unpause是比新创建一个container快的（warm start）

  - install cache：

    lean containers中的package层，read-only且被所有container共享

  - import cache:

    就是Zygote机制，但是命中和驱逐机制需要定制。命中可能与传统cache不同，存在多命中的情况（handler需要的包可能既在tree cache中的子节点也可能是父节点），这时需要找到最合适的entry（也就是Zygote进程）；驱逐因为Zygote进程会都有相同的包而存在共享内存的情况所以比较复杂

  

