---
title: "K8s Controller概析"
date: 2021-06-21T21:45:58+08:00 
description: "对于k8s-controller通用结构和通用流程的概析"
tags: [k8s]
featured_image: "/images/k8s-logo.png"
images: ["/images/k8s-logo.png"]
categories: k8s
comment : true
---

# k8s-controller

本篇文章介绍k8s-controller的大致结构和大致流程

![arch](https://github.com/YaoZengzeng/KubernetesResearch/raw/master/pic/kubecontroller/arch.jpeg)



此图展现了Controller的整体通用结构，上部Client-go部分是所有controller通用代码的抽象包（包括各种资源的controller，kube-proxy等，**这部分大家都差不多**），而下部custom controller是具体controller的逻辑代码



## 1）Reflector list&watch

controller自己有对监控的资源的缓存，与从api server中查到的数据（最终来自etcd）一致，**List用来获取资源的全量数据（每经过resyncPeriod时间，就使用List获取最新版本的API对象，强制更新缓存，保证缓存的有效性），Watch用来对资源进行监听，以增量的方式接收资源对象的变更**。Reflector会保存一个lastSyncResourceVersion，表示上次同步的版本，用来进行容错，出错时就从上次同步的版本开始。Reflector的工作就是将从api server HTTP GET到的数据实例化成对象，可以把Reflector视为informer的前端



Reflector工作流程：

1. 从api server获取一次全量对象，更新ResourceVersion，插入一个Sync类型的Delta到Delta FIFO中
2. 开一个go routine每resyncPeriod获取全量对象，做法同1
3. Watch ResourceVersion以后的对象变化情况，一旦有对象变化，根据变化类型（是新增，更新还是删除）产生相应类型的Delta插入到DeltaFIFO中

## 2) Reflector Add Object

Reflector作为生产者，informer作为消费者，Reflector将资源的变更事件放到队列当中，由Informer处理增量事件（如add，update，delete等）

增量事件表示（Delta）：

```go
type Delta struct {
	Type	DeltaType	// 增量事件类型，如Added, Updated, Deleted, Sync
	Object	interface{} //资源实例全部信息
}

```



## 3）Informer Pop object & 4) Informer Add Object & 5) Indexer Store Object

informer从delta队列中取得资源对象实例，**更新本地缓存**，Indexer实际上就是一个拥有读写锁的且拥有索引机制快速查找的Map，用来缓存监控资源对象数据



````go
type threadSafeMap struct {
	lock	sync.RWMutex
	items	map[string]interface{}
	
	indexers	Indexers
	indices	Indices
}
````



## 6) Informer Dispatch Event Handler Functions

Informer根据增量事件的类型交给初始化informer时注册的handler处理事件

不过通常来说，handler并不会真的实现自己的处理逻辑，因为这会阻塞Informer对于事件的分发。handler中的逻辑往往比较简单，就是将一个key放到队列中（图中work queue），可以通过多个go routine创建多个Worker，**Worker不断地通过ProcessNextWorkItem从队列中取得key，然后从缓存中取得“期望状态”，与从集群中取得的“实际状态”比较，使“实际状态”趋向于“期望状态”，这就是调谐**。也就是说，**Controller具体的处理逻辑是由Worker来完成的（ProcessNextWorkItem中的syncHandler）**



## 参考

https://www.cnblogs.com/YaoDD/p/11391344.html

深入浅出Shared-informer https://blog.csdn.net/weixin_42663840/article/details/81699303