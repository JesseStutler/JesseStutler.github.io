---
title: "Raft"
date: 2021-03-09T16:30:52+08:00 
description: "raft论文的基本解读"
tags: [分布式,raft]
categories: 分布式
comment : true
---

# Raft

## 引言

Raft是分布式数据一致性算法，用于解决PAXOS多年来晦涩难懂且难以工程复现的问题，本文对Raft发表的原文论文进行了大致解读

## 基本算法内容

### Basics

![图 5](https://github.com/maemual/raft-zh_cn/raw/master/images/raft-%E5%9B%BE5.png)

#### Follower,Candidate,Leader

每个server分为三种状态（状态转换图见Leader election）：

- Follower：只接受RPC请求（就算收到来自client的请求也会重定向给leader）
- Candidate：参加竞选，可以发送RequestVote RPC，同样也可以接受请求
- Leader（only one）：**只有Leader可以处理来自client的请求**，可以发送AppendEntries RPC，可以是追加日志条目用，也可以是心跳检测用（定期检测其他server是否还活着，通过**无条目追加**的AppendEntries RPC来做到）

#### term——任期

Raft将时间随机划分，每一段称为**任期**（任期是单调递增的），任期都以一次选举开始，选举可以是选出leader也可以是未能选出leader（未能选出leader就直接进入下一任期）

**每台server发现自己的任期小于其他机器就需要update到最新**

#### entry——条目

条目就是指日志的条目，由client发来的`command+任期数（term number，用来检测不一致性）+index（条目索引）`构成

### Leader election

![图 4 ](https://github.com/maemual/raft-zh_cn/raw/master/images/raft-%E5%9B%BE4.png)

状态转换图分析：

1. Starts up:

   初始时，每台server都是Follower

2. Follower--->Candidate：

   当超出election timeout时间（长时间未收到有Leader发过来的RPC消息，说明当前cluster未选出leader，或者是未收到来自candidate的RequestVote RPC），Follower增加自己的当前任期数，并将自己转换为Candidate；参加竞选leader，给自己投票，然后并发地向其他server发送RequestVote RPC请求，需要他们给自己进行投票（**一般规则是先收到谁请求就投谁**）；重设election timeout

3. Candidate--->Candidate：

   - 选举发生投票分歧

   未能选出leader（发生投票分歧），比如有好几台Candidates票数一致的情况，或者大家都是Candidate（不可能给竞争对手投票是吧:P ），增加自己的当前任期数，并开始新一轮的选举。不过这样有可能造成一直产生投票分歧的情况，打破这种情况并选出leader的机制就是**election timeout，Candidates从时间段中随机给自己选一个election timeout时间，如果发生投票分歧，先超时的Candidate赢得选举**

   - Candidate收到Leader（已经暂时选出的）的RPC请求，发现其任期比自己旧，拒绝请求并保持Candidate状态

4. Candidate--->Leader：

   赢得竞选（获得**大多数**servers的投票）

5. Leader--->Follower:

   通过RPC的回复发现自己的任期已过期（有比自己更新的任期），退回到Follower

6. Candidate--->Follower：

   - 输掉选举（收到了来自己已选出的leader的RPC，**但要确定自己的任期至少和Leader的任期相同，参考第3步**）
   - 通过RPC的回复发现自己的任期已过期（有比自己更新的任期），退回到Follower



### Log Replication

**首先要说明的是，Leader只追加条目（entry）而不修改或删除entry**

Leader通过AppendEntries RPC来加条目复制到其他servers上，如果有server挂了他也会一直重复尝试发送。

- Follower如何确定自己Leader发过来的条目可以追加？

  Leader发送的条目会包括索引号和任期数，如果Follwer没有找到相同索引号和任期数的条目，就拒绝请求，找到了就说明这个条目之前的条目都是相同的

- 条目什么时候应该被apply到各个机器上？

  我们称之为**提交**（commited），当条目**已经被复制到大多数的servers**上（维护一个replicas来确定），这些条目（包括之前的未提交的条目，前leader剩下的未提交的条目）就都会被提交，每台server都会维护一个highest index表示最后的已经提交的条目，已表示这之前的条目都已提交

### Leader crash所导致的log不一致问题

这个就是关键要解决的，Leader崩溃（然后发生Leader exchange）可能会导致一系列各server上**log不一致**的问题（前leader可能还未完全复制给所有机器），**Follower可能会带有现Leader没有的log，或者更复杂的不一致问题**，那如何解决呢？

**覆盖**。

如果有不一致的log，Leader需要找到和Follower的日志中相同的**最后一个条目**（也就是索引号和任期数相同，说明之前的条目都相同），然后将后面不一样的条目都**覆盖成Leader的**（当然缺少的话就直接追加）。

> 解决方法：当新leader出现时，他会先进行一致性检查，他会维护一个nextIndex，表示下一个要发送给follower的条目的索引号，当follower拒绝追加请求时（发现不一致），leader就减小nextIndex的大小，**直到条目相同为止**，然后leader把这之后的条目全部覆盖掉Follower的日志



### Safety

当leader提交的时候万一发生Follower崩溃的情况，而Follower复原之后又当上新leader，可能会出现覆盖之前已经提交的entry的情况，继而造成不同server最后执行了不同command的情况

> 解决方法：加上election restriction，对于candidate**必须要涵盖之前已经所有已经提交的entry**，也就是说就算candidate获得了大多数票数的情况下，必须以涵盖所有已提交entry为前提，否则不能赢得选举

- 那如何保证candidate涵盖了所有已经提交的条目呢？

  candidate的日志必须是最新的（up-to-date）。怎样规定最新呢？两个server的日志如果任期更新者就是最新的，或者任期相同而拥有更长索引者就是最新的。如果candidate发送RequestVote RPC，**而其log不比voter新，voter就要拒绝投票给candidate**。

还有的情况是，旧leader未能提交entry，而新leader也无法确定这个entry是否已提交

> leader只为当前任期是否应该提交维护一个replicas，也就是replicas如果是大多数的话就提交



### Follower and candidate crashes

- 当Follower和candidate崩溃时，Leader会一直无限期的重发RPC直到它们重启并成功收到为止

- 保证幂等性：如果server在响应RPC前崩溃，要保证恢复后RPC的响应结果是一样的



### Timing and availability

> 广播时间 << 选举超时时间 << 平均崩溃时间

系统的时间要遵循上述不等式，第一个不等式避免Follower收不到心跳消息而转而变成Candidate状态进入新的选举，第二个不等式避免一直无法选出Leader

广播时间和平均崩溃时间由现实决定，**选举超时时间一般在10ms到500ms之间**



## 优化

### Cluster membership changes

如果集群需要更改配置，比如替换掉原来的机器或者加入新机器，先将集群停掉，更改完配置之后再上线会让集群有段时间不可用，但是如果直接将旧配置改为新配置有可能会造成集群同时出现两个leader的情况。Raft用一种两阶段算法，引入了一个过渡配置——共同一致，集群同时可以有旧配置和新配置的存在，但需要进行一定的限制，具体参考：https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md#6-%E9%9B%86%E7%BE%A4%E6%88%90%E5%91%98%E5%8F%98%E5%8C%96

### Log compaction

避免存储日志过多（提交了之后的日志条目会占用过多磁盘空间，其实只要保留一点metadata用来标识和回滚就够了）

> 解决方法：snapshot（快照），**由每台server自己**将已经提交的条目信息用metadata的形式记录下来形成snapshot，然后就可以把已经提交的日志条目丢弃了，这样就可以减少日志占用的空间，而且方便一致性检查，就是我们前文提到的当leader crash时出现的一系列问题，可以快速检索metadata获取最后提交的条目信息

这里引入一个InstallSnapshot RPC，当出现server已经提交而Follower还未提交的情况（比如刚刚加入集群的机器或者比较慢的未收到entry的机器），这时候Follower的snapshot是远远落后的

### client interaction

client是随机请求集群中的server，不一定是leader，如果是Follower收到了请求他会重定向给Leader（AppendEntries RPC中包含了Leader的地址）

当Leader已经执行了client的请求发来的command，但是响应client前崩溃，client可能会重新提交这个command的请求，造成两次执行同样的command，我们需要给每个command维护一个serial number，表示command序列，当已经执行过了这个serial number的command，收到同样的请求就不再执行

## References

https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md（论文中文翻译和原文）

https://raft.github.io/（raft概述和动画演示，以及论文原文下载）