---
title: "Serverless分析"
date: 2021-05-18T14:13:36+08:00 
description: "serverless现状分析（持续更新）"
tags: [serverless]
featured_image: "/images/serverless.jpg"
images: ["/images/serverless.jpg"]
categories: serverless
---

# Serverless分析

本文根据Berkeley rise lab的综述Cloud Programming Simplifified:

A Berkeley View on Serverless Computing并结合其他相关材料进行总结，探究serverless的研究点，本文会持续进行更新。



*简单的说，Serverless就是FaaS+BaaS*

### 特点

- 按使用量付费（无请求时无资源无分配无花费，有请求时按使用量，按时间计算付费），性能提高（高并发量），autoscale，强隔离性（多租户），可供有突发流量情况而又无服务器扩展需求的公司使用；

- **低请求量服务改造**：原先需要一直监听请求的应用，当无请求来时需要一直占用资源，而改造成serverless可以用function代替原先的应用，这样无请求来临时可以down to zero，有请求来时再invoke一个或多个function实例（而且这些function是可以并行的）并进行处理；（不仅是针对可以减少资源使用量，而且可以应对流量尖峰）

- 由外部服务触发比如S3（有object更新，比如新增图片），消息队列（事件驱动，收到事件），或者以API gateway的形式（可以是以Backend或以function的形式）等待HTTP request到来触发

- **一定是**stateless，无法保证写到memory或者local disk的数据（VM上）下次被invoked还能读到，需要借助外部存储服务来保存状态或数据
- 适合short-lived task

从serverful过渡到serverless就像从汇编语言过渡到高级语言一样，汇编语言计算一个c=a+b需要指定寄存器，存放，计算结果然后并存回，而serverful就像汇编语言一样需要先知道哪些资源是可用的，然后给资源加载code和环境，执行计算，再得到结果，这些原先需要平台使用者去知晓，但是serverless不需要programmer去知晓和管理资源，只需要**编写code，编写function，编写业务就够了**

## 现今Serverless的有限性

### 存储对于细粒度操作的局限性

因为function之间是相互隔离的，**所以需要借助外部存储服务(BaaS)来提供状态的支持**，这是serverless的特性所致。但是对于划分到function这么细粒度的操作来说，现在的外部存储服务要不是太贵（access或者storage）要不就是延迟太高，e.g:对象存储比如AWS S3等，access花费和延迟过高；key-value数据库存储费用高，扩容慢；内存存储如redis等没有容错性，不能自动扩缩。当然这要看应用的要求，但还是与serverless理想的存储方案相差不少。

### 缺少细粒度的消息沟通

背景：两个task合作，taskA需要taskB的output作为input，但是不知道何时output会过来，所以需要引入消息中间件，但是现有的消息中间件对于细粒度(task/function)操作的延迟和花费太高

可能的解决方案：自己设计消息通知机制比如长期运行一个汇集消息的server，能够以命名的方式直接定位到function实例从而获取到ouput等

### 标准沟通模式对于细粒度的性能太差

背景：broadcast，shuffle，aggregation都是分布式系统中重要的原语，但是如果划分粒度过细，比如拿聚合来说，VM实例中的function如果本地不做聚合而每次聚合都需要到远端聚合，那么这个消息数量会成倍增加，shuffle则更多

### 冷启动的局限性

1）启动function需要一定时间（分配和加载资源：分配VM，初始化container，将function的静态文件拷贝到container）

2）需要一些时间去下载函数执行环境（OS，库，语言的runtime比如JVM等）

- 函数的package依赖需要经过远端的download，local install，import过程，这个时间比较长，是否可以在本地machine上预先下载好所有语言涉及的包？（通过压缩的方式存储）这样直接去本地加载package，省去去远端下载package的时间。**所有container通过overlayfs或者bind mount共享已经安装好的package**

- SOCK：利用Zygote机制预import一些需要的package（这样的Zygote很多，需要预import什么package就fork出新的Zygote），这样从Zygote进程fork出的新子进程不需要进行同样的初始化操作，直接从内存读取即可（减少开辟新内存的消耗）

  > tips：fork出的子进程与父进程共享堆栈，fd，代码段，由于copy-on-write，只有子进程写时才会完全拷贝

  **含Zygote进程的container-->含从Zygote fork出的子进程的container**

3）有些应用对于代码需要做一些定制的初始化操作，需要花费一定时间（比如加载和初始化数据结构，库等）

4）如果需要频繁冷启动，namespace的频繁creation和cleanup需要性能损耗

#### 什么时候冷启动会发生？

1. 当function的code或者配置改变的时候，或者function第一次部署的时候
2. idle instance被shut down
3. instance到了最大age被shut down（即使仍然在运行）
4. 之前的instance都在忙于处理请求，需要横向扩展的时候

#### 什么时候需要考虑冷启动的影响？

也许像要访问存储服务的function本来就需要等待存取的latency，冷启动时间相对这段latency可有可无；也许实时数据流服务会频繁地invoke function，function会一直处理event很多次（可能200000次在到达最大age之前），那冷启动时间也可有可无。

但是对于请求量较少的function，可能一小时invoke一次，就有可能中间被shut down，需要每次都冷启动，那就需要考虑冷启动的开销，如果冷启动需要加载的依赖和库过大，就有可能需要很多的冷启动时间；对于需要快速回应的应用也需要考虑冷启动的影响

#### 解决冷启动方案

1. AWS使用VM就绪池，在function启动之前就准备接收function的调度，这样基本上只受scheduling latency的影响
2. 尽可能的减少依赖，尽可能地用加载较快的语言（像Java中的JVM加载较慢）

#### 降低冷启动时间的好处

1. 可以让idle instance更少一些，可用资源更多一些（这样就不用因为担心冷启动时间过长而一直等待后续的请求了）

## Serverless可以探索的点

### Abstraction

- Resouce requirements

  不要让serverless平台的使用者来指定要使用多少资源，这样违背于serverless的理念（不应该管理资源），而且会降低资源的利用率。**更好的做法**是让cloud provider来推断出需要使用多少资源，比如**静态代码分析，归档之前跑完的数据，动态编译等等**，总而言之就是要自动的推断出需要多少资源。

- Data dependencies

  现在的serverless平台无法知晓function之间存在什么数据依赖，甚至需要交换的数据的规模。**更好的做法**是暴露一个API让应用指明function的computation graph，以便更好地放置function的位置；而且可以引入coordinator来解决function之间顺序依赖的关系，和调控function的状态（function为有限状态机，收到消息发生状态改变）

### System

- Storage

  1. **Ephemeral storage**（暂时存储）

  既然serverless computing需要的是暂时的状态存储，当计算结束时这些状态就可以丢弃，那么可以用暂时存储的方案，比如用内存存储（以分布式内存存储的方式），利用RDMA来减少延迟，利用共享内存的方式来减少serverful computing中内存被VM实例独占，无用内存碎片过多的情况

  2. Durable storage（长期存储）

  大部分都是暂时存储的情况（我觉得serverless也是适用于暂时存储），但是如果针对于设计serverless数据库的话需要长期存储，可能需要多个存储服务的结合，以及像SSD这样提高硬盘的IOPS

- Coordination/signaling service

  **解决缺少细粒度的消息沟通和数据一致性问题**（因为多个function可能会放到一起，所以分布式系统中的一致性算法和leader election不适用）

- Minimize startup time

  **解决冷启动的局限性**

  1. 解决1）：提供更新的轻量级的隔离机制（e.g:Firecracker）

  2. 解决2）：利用unikernels，预配置硬件，静态分配数据结构，只包含应用所需的驱动和函数库等；或者动态地加载函数库（有点类似ddl）

  3. 解决3）：提前做初始化操作；当做完初始化操作call readiness API去通知function工作；利用warm pool存放已经加载好的拥有流行的系统和库的实例等

  4. **WebAssembly**，有一项调查显示其实50%的function运行时间不超过100ms，但是启动container和加载runtime时间却远远超过100ms以上。

     > VM或者container初始化需要设置system library以供系统调用使用，这会引入开销；且VM或container下的现有付费模型是根据内存使用量和使用时间来计算的，像细粒度的cpu周期使用量等没有涉及到



### Network

解决标准沟通模式对于细粒度的性能太差问题，解决方案可同Abstraciton——data dependencies，让应用提供一个computation graph，以便serverless平台能够将一些function放置到同一VM实例中，**减少通过network发送的消息，尽量在本地先处理完**



解决方案：

1）eBPF Tail & Function Calls？

### Security

暂不考虑



### Architecture

- 硬件的异构性和性能提升陷入瓶颈

  背景：同样的架构但是不同时代的产品，虽然价格一样但是速度可能不同；硬件性能提升陷入瓶颈

  解决方法：

  1）使用语言特定的处理器提升处理速度

  2）使用DSA（Domain Specific Architectures），比如GPU加速图像处理，TPU加速factor处理等

## Serverless已经适配的应用（持续更新）

### Multimedia processing

- 比如有一个图片上传（比如用户上传头像）到AWS S3这样的object storage，触发function将图片处理成缩略图，然后重新存回到AWS S3或者网站自己的存储服务当中
- 或者有一个文件上传后触发function将文件进行压缩然后重新存回

### Database

- 当数据库更新的时候invoke一些function进行一些动作，比如添加或删除的时候
- 网站上有天气系统，当点击查看时触发function取得数据库的天气数据
- 支撑购物系统快速获取商品信息

### IoT sensor input messages

- 处理或过滤来自IoT device的MQTT消息，实时性要求高

### Stream processing at scale

- 连接消息源处理事件流，实时性要求高，serverless弹性扩缩能力和高并发能力天然适合

### Batch jobs or scheduled tasks（workflow，需要给定状态图）

- 一天内不会需要很多时间处理的batch job，可以在一天内的其他时间不需要一直running

### HTTP REST APIs and web applications（天然适合）

- 简单的REST操作适合serverless

## Serverless可以优化的应用（持续更新）

**优点**

1）function实例启动快 

2）花费少（按使用量计费）

3）可以invoke多个实例并行处理请求（**主要原因**）

**引入的问题**（如何将原来的应用迁移成细粒度的function）

1）如果仅仅是简单地将应用划分成不同的代码段，一个代码段一个function，那这些不同的function不能很好地用到warm start，每次都是cold start

2）可能会受到平台实例并发数量的限制

3）如果function instance是有顺序协作关系的（以链的形式），可能会发生死锁

4）并发性能是提高了，但是如何保证拥有和改造前应用相近的质量？（是否要斟酌牺牲一点？）

### video encoding(compute-heavy task)

背景：现有的编码方案需要花费数十分钟甚至数十小时去上传视频

已有的解决方案：ExCamera，以函数语义并行执行编码的慢的部分，串行快的部分

### MapReduce

背景：Map，Shuffle，Reduce皆可以移植到serverless computing，在mapreduce期间资源需求变化很大

已有的解决方案：只解决了Map-Only job，整套的MapReduce待解决

### 线性代数

背景：并行数量在计算期间变化很大，移植到serverless computing可以提高执行速度并且提高资源利用率

已有的解决方案：Numpywren

### Machine learning

背景：预处理，模型训练等不同阶段的资源需求变化大，与线性代数类似

已有的解决方案：Cirrus

### Database（不太好解决）

背景：使用cloud function来进行数据存储，但是需要存储服务的支持（Backend）；但是数据库连接又需要特定协议，从函数层面上不到网络层这一层；

## References收集

- https://blog.symphonia.io/posts/2017-11-14_learning-lambda-part-8 Learning Lambda — Part 8 cold starts

- https://blog.symphonia.io/posts/2020-06-30_analyzing_cold_start_latency_of_aws_lambda Analyzing Cold Start latency of AWS Lambda

- https://archive.fosdem.org/2020/schedule/event/containers_bpf/ BPF as a revolutionary technology for the container landscape

- https://github.com/cncf/wg-serverless/tree/master/whitepapers/serverless-overview CNCF Serverless Whitepaper v1.0

- chrome-extension://ikhdkkncnoglghljlkmcimlnlhkeamad/pdf-viewer/web/viewer.html?file=https%3A%2F%2Farxiv.org%2Fpdf%2F2010.07115.pdf

  SSVM——WebAssembly Virtual machine





