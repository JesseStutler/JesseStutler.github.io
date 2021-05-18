---
title: "ATC18——高性能workflowSAND"
date: 2021-05-18T14:28:02+08:00 
description: "serverless论文总结系列#1"
tags: [serverless,论文总结系列]
featured_image: "/images/serverless.jpg"
images: ["/images/serverless.jpg"]
categories: serverless
comment : true
---

# ATC18——高性能workflow SAND

这是一篇关于优化serverless workflow的文章

论文下载链接：https://www.usenix.org/conference/atc18/presentation/akkus

## Sandbox

- 同属于同一个workflow的function属于一个**application**，**一个application一个container**，而不是一个function一个container

  > 但是有可能会引入资源竞争问题？

- 当有request到来时，==通过fork function实例来快速水平扩展==（**实验证明，fork是最快的，比直接执行二进制文件创建进程都要快**），而不是频繁的冷启动一个不同的container。而且，同一个function的不同实例（也就是进程）可以共享内存，库只要加载一次就够了，相比另起一个container的内存占用量，内存占用量少很多。function执行完之后可以回收资源，等有请求来了再fork新实例，相比为了解决负载尖峰而一直保持container idle占用资源，可以避免资源一直被占用。

## Message Bus

**SAND使用了一个机制：同一台host中的function通过local message bus来沟通，不同host中的function通过global message bus来沟通，而且global message bus可以保存local message bus中的消息作为备份（用来容错）**

> tips：local/global message bus都为不同的funciton维护有不同的队列（或者topic），global像kafka这种实现有partition做容错，host agent订阅global message bus，本地function订阅local message bus。而且message bus不直接传递data，而是传递数据的引用（比如local可以通过in-memory的key-value存储，来快速获取数据，global可以通过分布式存储来获取数据），不仅存取数据快，这样检查状态和回滚也方便。

鉴于现有的serverless平台中的workflow沟通机制，即使两个function在同一台机子当中，也是要通过外部消息队列服务来存取的，这引入了极大的延迟，local message bus能够削减这段延迟时间

## Host Agent

Host agent是每台机子上的代理，他负责local message bus和global message bus的合作（比如备份，细节里会细说）；为自己机子上的函数从global message bus存取消息（订阅topic）；孵化容器和fork function

## 细节

### 如何做备份

当function产生message到local message bus当中自己的队列时，会产生**一份拷贝给host agent的队列**，然后host agent将这份拷贝消息放到global message bus的这个function的队列（或者topic）当中，==作为备份并且打上标签表示完成状态==，host agent会追踪要接收这个消息的下一个function的完成进度，顺利完成会将状态转为finished，处理失败会将状态转为failed并交给另一个机子上的function处理

### workflow流程

假设有两个function完成workflow，一台机子（两个partiton)

Step1：user request发送给function1，global message bus将消息放到partition1

Step2: host agent（host agent负责global的订阅）将消息从partition1取出并放到local message bus的function1的队列

Step3.1: function1（function负责local的订阅）将消息从local的自己队列中取出，fork新实例并处理，然后产生下一个消息给funtion2，将消息放到local message bus的function2队列

Step3.2：同时，会有一份3.1中产生的消息的==拷贝==放到local message bus的host agent的队列，host agent将这个消息放到global的partition2中作为==备份==，并打上标签表示状态‘processing'

Step4.1: function2从local自己队列中取出消息并处理，因为他是workflow的最后一步，所以完成后直接产生新消息给local中的host agent队列，host agent将这个消息放到global中

Step4.2：host agent追踪到funciton2顺利完成，将标签3.2中的消息改为finished表示完成

Step5：回复user表示完成