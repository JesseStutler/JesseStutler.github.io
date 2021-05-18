---
title: "NSDI17 ExCamera"
date: 2021-05-18T14:45:44+08:00 
description: "serverless论文总结系列#5"
tags: [serverless,论文总结系列]
featured_image: "/images/serverless.jpg"
images: ["/images/serverless.jpg"]
categories: serverless
comment : true
---

# NSDI17-ExCamera 

这是一篇将video encoding改造到serverless平台上的文章

论文链接：https://www.usenix.org/conference/nsdi17/technical-sessions/presentation/fouladi

## mu框架

<img src="/Users/chenzicong/Library/Application Support/typora-user-images/image-20210413105734381.png" alt="image-20210413105734381" style="zoom: 25%;" />

### 大致流程

AWS s3 invoke第一个Worker（function实例），然后Worker与Coordinator建立TLS连接并保持（Coordinator通过RPC call来调控Worker的状态，Worker就是一个有限状态机），当Coordinator收到来自Worker的message时，就会根据状态转换逻辑产生新的状态给Worker并发送下一个RPC请求。

Coorinator是dependency-aware的，他会根据Worker产生的output来指派可以处理这个output的worker，这样就可以顺序执行而不会产生死锁



## 出现原因

传统的视频encoding速度太慢，一些实时的视频处理平台需要快速的视频上传业务

> 背景知识：我们都知道视频由一帧帧的图片组成，对于将一段视频压缩成比特流来说，有些帧与帧之间，图片的某些部分是重复的，那压缩成比特流就不必重复，encoding过程就是花费cpu时间来寻找帧与帧之间的联系，从而尽可能的压缩输出的比特流大小；但是这样带来的问题就是帧与帧之间会存在依赖关系，从而不能将比特流从中间段进行解码，比如说直播的时候有不同的清晰度，想要切换成更高清的流。现在引入Stream Access Point来切分视频数据流

借助Stream Access Point技术（将视频数据流进行切分，各段数据流都是独立的，段与段之间的帧没有依赖关系，VP8/VP9使用的是
"key frame"概念），**可以将各段encoding过程改造成使用相同的function来处理**，各段压缩完成之后再进行简单的连接（串行），形成一个完整的视频数据流



## ExCamera encoding流程

1. （并行）使用*vpxenc*（谷歌优化的encoder）encode六个帧，都以key frame为开头（也就是使用Stream Access Point分割视频数据流）,这代表一个chunk

2. （并行）使用ExCamera设计的encoder将原先*vpxenc*生成的key frame替换成与前面部分的encoder产生的输出相关联的inter frame（因为key frame会影响压缩速率，所以ExCamera针对此进行了优化）。最后生成的chunk只有一个key frame为开头。

   >  这step2与step3之间还涉及到很多并行优化步骤，因为涉及到视频encode和decode背景，略过

3. （串行）将各个chunk顺序连接起来