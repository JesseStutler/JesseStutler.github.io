---
title: "K8s Iptables分析"
date: 2021-06-10T14:51:48+08:00 
description: "k8s iptables分析"
tags: [k8s]
featured_image: "/images/k8s-logo.png"
images: ["/images/k8s-logo.png"]
categories: k8s
comment : true
---

# K8s iptables分析

笔者最近因为需要看了下k8s的kube-proxy在默认iptables模式下对于iptables规则是如何管理的，以此作为记录并分享，后面会继续跟进kube-proxy并进行源码分析

这里笔者分享一套还不错的iptables学习路线：

https://www.zsythink.net/archives/category/%e8%bf%90%e7%bb%b4%e7%9b%b8%e5%85%b3/iptables

## iptables知识

### 查

-t [表名] 指定想查看的表的规则，默认为filter表

-L [链名] 指定想查看的某个表在某个链当中的规则，不写链列出表在所有链当中的规则

-v 列出详细规则

-n 不做域名解析，直接显示ip（通常-nvL同行）

--line-number 显示行数

e.g:

`iptables -t nat -L INPUT`

列出nat表在INPUT链中的所有规则

### 增

-I [链名] 插入规则到指定的链当中（**在链的首部插入**）

-I [链名] [序号] 插入规则到指定的链当中（**指定是链的第几条规则**）

-A [链名] 插入规则到指定的链当中（**在链的尾部插入**）

-s [源地址,[源地址]...] 或[源地址网段]

-d [目标地址]

> 用`!`在-s或-d前表示取反，非地址匹配

--sport 源端口

--dport 目标端口

--sport/dport [起始端口:结束端口] 表示匹配此范围内端口

> 指定源端口和目标端口为扩展匹配条件，-m用来指定扩展模块，如-m tcp -dport 22表示匹配ssh，若不指定-m（最好指定），默认与-p指定的协议相同

-m [模块] 使用模块扩展匹配条件

```shell
离散端口：
-m multiport --dports 22,53,80 
需要指定multiport模块用来离散匹配

范围ip：
-m iprange --src-range 192.168.0.211-192.168.0.220

加备注：
-m comment --comment [...]
```



-p [协议]

-i [流入网卡]

-o [流出网卡]

-j [target] target可以是动作，如DROP,ACCEPT,REJECT等 ；也可以是自定义链，将系统链和自定义链关联起来进行进一步的匹配



-N [自定义链名] 创建自定义链，将指定规则放到自定义链中方便管理，防止某条系统链过长比较难找，k8s很多都是自定义链

**在系统链当中定义规则，并指明-j 自定义链名即可将自定义链引用到系统链上，如果规则匹配，就会到自定义链上进行进一步的匹配，如果不指定-j光定义自定义链是不会产生效果的**

### 删

- -D [链名] [序号] 删除链的第几条规则（可用--line-number先查看链的序号）

或不指定序号，而是根据规则来匹配

e.g:

`iptables -D INPUT -s 192.168.0.211 -j ACCEPT`

删除INPUT中filter表的（不指定-t默认为filter）源地址为192.168.0.211，target为ACCEPT的规则



- -F [链名] 删除某条链的所有规则

e.g:

`iptables -t 表名 -F 链名`

删除指定表中某条链中的所有规则



- -X [自定义链名] 删除自定义需要满足以下规则：

1. 自定义链没有被任何默认链引用，即自定义链的引用计数为0。

2. 自定义链中没有任何规则，即自定义链为空。

### 改

- -P [链名] [target] 修改制定链的默认规则

> iptables黑名单即链的默认规则为ACCEPT，而定义DROP或REJECT规则，白名单即反过来

- -R [链名] [序号] 修改链中制定第几条规则（有坑，需要匹配写明所有原匹配规则，所以不如删了原来的规则再加一条）

- -E [原链名] [新链名] 将原自定义链名改为新链名

### 保存规则

- 第一种

  `service iptables save`

  只要没保存，就可以通过`service iptables restart`恢复

- 第二种

  `iptables-save > /etc/sysconfig/iptables`

  `iptables-restore < /etc/sysconfig/iptables`

  

### NAT

https://www.zsythink.net/archives/1764

#### SNAT

-j SNAT --to-source [ip] 将源ip进行转换

一般放在POSTROUTING链当中

也可以放在INPUT链当中

#### DNAT

-j DNAT --to-destination [ip:port] 将目标ip进行转换

一般放在PREROUTING链当中

也可以放在OUTPUT链当中

#### MASQUERADE

类似SNAT，只不过SNAT需要指定转换ip，MASQUERADE可以指定-o流出网卡，根据流出网卡的ip自动配置转换ip，应对转换ip变动的情况

#### REDIRECT

-j REDIRECT --to-ports 

端口重定向

REDIRECT规则只能定义在PREROUTING链或者OUTPUT链中

## Kubernetes iptables分析

### Iptables解析

- Cluster-IP实际上就是做了个cluster-ip到对应pod的endpoint的DNAT转换

  - 非本机访问pod：

    PREROUTING --> KUBE-SERVICE --> KUBE-SVC-XXX --> KUBE-SEP-XXX

  - 本机访问pod：

    OUTPUT --> KUBE-SERVICE --> KUBE-SVC-XXX --> KUBE-SEP-XXX

  到具体某个pod的查看链过程：

  ```shell
  iptables -t nat -nvL OUPUT（或者PREROUTING)
  iptables -t nat -nvL KUBE-SERVICES
  #SVC规则全在KUBE-SERVICES链当中
  #以Load Balancer服务为例，含三条iptables链：KUBE-MARK-MASQ,KUBE-SVC-P24HJGZOUZD6OJJ7(svc hash值),KUBE-FW-P24HJGZOUZD6OJJ7
  #查看cluster-ip链规则，-d指定了cluster-ip，其中定义了各pod链，命中率按概率算
  iptables -t nat -nvL KUBE-SVC-P24HJGZOUZD6OJJ7
  #查看具体某个pod链规则，其中包含DNAT转换，转换到具体某个endpoint
  iptables -t nat -nvL KUBE-SEP-L4YOSO3ZOS4HVO5N
  
   0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/echo-load-balance: */ tcp to:10.244.2.162:8080
  ```

  出node的查看链过程：

  ```shell
  iptables -t nat -nvL KUBE-MARK-MASQ
  #上面提到的KUBE-MARK-MASQ链，是在OUTPUT过程中给出node的流量打上标签的，用于在后面POSTROUTING做SNAT转换
  iptables -t nat -nvL POSTROUTING
  iptables -t nat -nvL KUBE-POSTROUTING
  #KUBE-POSTROUTING中规则如下：
  Chain KUBE-POSTROUTING (1 references)
   pkts bytes target     prot opt in     out     source               destination
     17  1020 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ mark match 0x4000/0x4000
  
  #表示匹配打上了0x4000标签的流量，即KUBE-MARK-MASQ中打上的，做SNAT动态转换，将pod ip转换成node ip发往其他node
  ```

  DNAT和SNAT转换过程：

  ```shell
  #比如在一台ip为10.0.0.40的机子上，访问cluster ip为10.111.107.12,其中一个pod的endpoint为10.244.2.162:8080，10.244.0.0为flannel.1的ip（因为由flannel.1发出）
  10.0.0.40:xxx ——> 10.111.107.12
                |
                |	DNAT
                V
  10.0.0.40:xxx ——> 10.244.2.162:8080
                |
                | SNAT
                V
  10.244.0.0:xxx ——> 10.244.2.162:8080
  ```

  

  

- NodePort

  - 非本机访问：

    PREROUTING --> KUBE-SERVICE --> KUBE-NODEPORTS --> KUBE-SVC-XXX --> KUBE-SEP-XXX

  - 本机访问：

    OUTPUT --> KUBE-SERVICE --> KUBE-NODEPORTS --> KUBE-SVC-XXX --> KUBE-SEP-XXX

  NodePort在KUBE-SERVICES链中最后一条规则，如果--dst-type为LOCAL，即node ip，则匹配

  ```s
  126K   16M KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
  ```

  查看KUBE-NODEPORTS链：

  ```shell
  iptables -t nat -nvL KUBE-NODEPORTS
  #第一条用来打标签，第二条用来转到cluster-ip链规则来做DNAT转换
  0     0 KUBE-MARK-MASQ  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/echo-load-balance: */ tcp dpt:32591
      0     0 KUBE-SVC-P24HJGZOUZD6OJJ7  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/echo-load-balance: */ tcp dpt:32591
     
  ```

  DNAT和SNAT转换过程：

  ```shell
  #比如在一台ip为10.0.0.40的机子上，访问NodePort,端口为32591,其中一个pod的endpoint为10.244.2.162:8080，10.244.0.0为flannel.1的ip（因为由flannel.1发出）
  10.0.0.40:xxx ——> 10.0.0.40:32591
                |
                |	DNAT
                V
  10.0.0.40:xxx ——> 10.244.2.162:8080
                |
                | SNAT
                V
  10.244.0.0:xxx ——> 10.244.2.162:8080
  ```

  

  ![img](https://res.cloudinary.com/stackrox/v1578619751/kube-networking-5_zswvwp.svg)

- Load Balancer：

  Load Balancer会有KUBE-FW-XXX链在KUBE-SERVICES链中定义，用来匹配Load Balancer external ip，如果命中，会有三条规则，第二条导向Cluster-IP：

  ```shell
  0     0 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/echo-load-balance: loadbalancer IP */
      0     0 KUBE-SVC-P24HJGZOUZD6OJJ7  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/echo-load-balance: loadbalancer IP */
      0     0 KUBE-MARK-DROP  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/echo-load-balance: loadbalancer IP */
  ```

  KUBE-MARK-DROP也和KUBE-MARK-MASQ一样，是用来打标签的，用来应对服务可能没endpoint的情况，打完标签后进入INPUT链或者OUTPUT链的规则当中，匹配KUBE-FIREWALL链，将打上了标签的流量DROP掉

### 参考

https://juejin.cn/post/6844904098605563912

https://yuerblog.cc/2019/12/09/k8s-%E6%89%8B%E6%8A%8A%E6%89%8B%E5%88%86%E6%9E%90service%E7%94%9F%E6%88%90%E7%9A%84iptables%E8%A7%84%E5%88%99/

https://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/03/27/Kubernetes-kube-proxy.html

https://www.stackrox.com/post/2020/01/kubernetes-networking-demystified/