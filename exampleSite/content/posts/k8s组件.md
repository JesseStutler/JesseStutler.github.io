---
title: k8s组件
description: k8s组件系列
date: date
featured_image: /images/k8s-logo.png
tags: [k8s]
categories: k8s
---

# k8s组件

## pod

- 为什么需要pod?

  主要目的是由多个进程组成的一个应用程序，多个进程不能聚集在一个容器中运行**（容器的设计目的就是只运行一个进程，如果容器中运行多个不相关的进程，比如需要包含一种进程崩溃后能够重启的机制，同时将进程的活动记录记录到相同的标准输出中，我们很难确定每个进程分别记录了什么），我们用pod来封装容器，将其作为k8s的基本单位**，既可以做到一个进程单独运行于一个容器当中，容器之间相互隔离，保持了容器的特性，又能同时运行一些密切相关的进程，为他们提供相同的环境。

pod中的容器共享network namespace，容器中运行的进程之间能够通过端口来相互通信（同一个pod中的容器拥有相同的loopback网路接口，可以通过发往localhost与其他容器中的进程相互通信）

- 如何决定多个容器是否要放入同一个pod中？
  - 它们需要一起运行还是可以在不同主机上运行
  - 它们代表的是一个整体还是相互独立的组件
  - 它们必须一起扩缩容还是可以分别进行
  



### liveness probe & readiness probe

- liveness probe——存活探针（**在pod running时检测**）

  通过 TCP、HTTP 或者命令行方式对应用就绪进行检测。对于 HTTP 类型探针，Kubernetes 会定时访问该地址，如果该地址的返回码不在 200 到 400 之间，则认为该容器不健康，会杀死该容器重建新的容器。

  

- readiness probe——就绪探针（**在pod就绪前检测**）

  对于启动缓慢的应用，为了避免在应用启动完成之前将流量导入。Kubernetes 支持业务容器提供一个 readiness 探针，对于 HTTP 类型探针，Kubernetes 会定时访问该地址，如果该地址的返回码不在 200 到 400 之间，则认为该容器无法对外提供服务，不会把请求调度到该容器。

  

### 容器重启策略

- Always ： 容器失效时，kubelet 自动重启该容器（**就算成功执行完容器也会重启**）
- OnFailure ： 容器终止运行且退出码不为0时重启
- Never ： 不论状态为何， kubelet 都不重启该容器

### 节点亲和性

- nodeAffinity:

  约束pod可以调度到哪些节点，是对nodeSelector的一种加强。有以下两种字段：

  - requiredDuringSchedulingIgnoredDuringExecution（硬限制）：

    表示node**必须**满足**pod指定条件**(matchExpression）才能将pod调度到这些节点上，否则**不进行调度**（与nodeSelector相同）

  - preferredDuringSchedulingIgnoredDuringExecution（软限制）：

    **偏好**将pod调度到满足pod指定条件的node上，如果节点不满足也没关系，**还是可以将pod调度到其他节点上**

  **IgnoredDuringExecution**表示纵使节点标签发生改变，pod仍可以在节点上继续运行

### pod间亲和性

**如果是大集群不建议使用pod亲和，否则需要涉及到大量的处理**

- podAffinity和podAntiAffinity（pod亲和和反亲和）:

  **既需要node匹配，又需要pod匹配**，他的规则是：如果 X 节点上已经运行了一个或多个 满足规则 Y 的pod，则这个 pod 应该（或者在非亲和的情况下不应该）运行在 X 节点，X和Y都是标签（X是node的标签，可以是k8s内置标签也可以是自定义标签，表示一个**拓扑域**，Y是pod标签）

  字段与nodeAffinity相同，需要在pod的Spec中写明



### 污点和容忍度

污点其实就是节点的**反亲和性**，用处在于某些pod需要调度到特定节点上，而其他pod不能调度到这些节点上（就需要给这些pod加上容忍度），或者是某些节点挂掉了，需要驱逐某些pod，或者加上容忍度，容忍在指定时间内节点可以恢复，否则就要被驱逐

使用 kubectl taint nodes node1 key*value:NoSchedule命令给node加上taint，用法类似kubectl label，要说明的是NoSchedule，有以下几种动作：

- NoSchedule，如果node打上了taint，且pod没有指明tolerations，则pod不会被调度到该节点上
- NoExecute，如果node打上了taint，则在该节点上运行的pod会被驱逐（一般是pod已经在该node上运行了，然后node挂了，由k8s自动给该node打上taint，然后驱逐node上运行的pod），pod也可以指明tolerations不被驱逐，或者在tolerations当中指明容忍时间，在超出容忍时间之后仍会被驱逐
- PreferNoSchedule，如果没有合适的node，则可以被调度该node上（软限制）

节点可能会出现某种问题，如not-ready或者unreachable，那么**k8s会自动给node打上以下taint：**

如果pod未指定对以下两种taint的容忍度，则k8s自动会给pod加上对以下taint的tolerations并指定容忍时间为300s，超过该时间taint未清除则pod仍会被驱逐

![image-20201106201619457](https://tva1.sinaimg.cn/large/008eGmZEgy1goa6tpwjs7j314i05sgms.jpg)

### init container

init container是pod中先于应用容器创建的容器，用于为应用容器进行一些初始化配置（配置卷，配置环境变量等等），或者安装一些实用工具等等，**推荐image使用busybox，然后再运行一些命令**

**init container可以有多个，顺序执行，前一个成功执行后才能执行后一个（如果init容器执行失败，则k8s会重启该pod），有点类似job，全部成功执行完成之后（注意是之后），才会创建应用容器**

init container不能使用readinessProbe和livenessProbe，**因为他们在应用容器创建之前完成，pod还未就绪**

### 容器资源限制

除自定义资源限制外，一般限制CPU和memory

两种限制：

- requests:

  **节点上pod的requests总和不能超过节点可用资源的100%**，k8s的调度器调度也是根据requests来调度，如果pod的requests**小于**（节点可使用资源总量-其他pod申请量），即使其他pod的**实际使用资源量**未达到他们所申请量，这个pod也不可以调度到这个节点上。当未进行limits限制时，pod的实际使用资源量是可以**超过**申请量的，比如有pod暂时不需要CPU时，而有计算密集型pod需要全力使用CPU时，便可以超过申请量全部占用CPU。

- limits（底层实际上是cgrop进行限制）:

  **节点上pod的limits总和可以超过节点可用资源的100%，我们称之为超卖**，CPU进行limits限制时，进程分不到比限制量更多的CPU，而内存与CPU不同，因为内存一旦分配给进程，想要回收就需要进程主动释放，所以如果有恶意pod或者故障pod，或者是贪婪的进程就有可能吃掉节点上所有的内存，需要进行限制，**进程如果申请超过limits的量**或者节点使用总量超过100%就有可能会造成OOM(out of memory)，进程就会根据策略被杀死（linux OOM_reaper subsystem）

#### pod QoS

三种等级（优先级从低到高）：

- BestEffort
- Burstable
- Guranteed

当资源不够时，比如节点内存不够用时，就需要杀死pod中的进程，根据优先级**从低到高来杀死**

那怎么来决定pod的QoS class呢，粗略的规则就是，BestEffort未声明requests和limits；Guranteed必须声明requests和limits，且两者相等；其他情况皆为Burstable（比如只有requests或者limits，requests<limits等），细致的规则可以参考：

**单容器pod**：

![img](https://tva1.sinaimg.cn/large/008eGmZEly1gnu8y3ll3pj32nb0u0qv5.jpg)

**多容器pod:**

![img](https://tva1.sinaimg.cn/large/008eGmZEly1gnu8yai41bj32m20u0qv5.jpg)

**当资源不够时，不同等级的情况下先杀死QoS等级低的Pod中的进程，相同等级的pod根据进程已使用资源量和可用资源量的比较来杀死进程**



### LimitRange和ResourceQuota

LimitRange是用来限制某个命名空间**单个pod**的可用资源，API server中存在LimitRange admission control插件，当发起Pod Post请求时就需要进行检查或者添加字段，比如pod层面可以限定pod中所有容器的最小requests总和或最大limits总量，容器层面可以限定默认的requests和默认的limits，还可以限定PVC最大申请量等。不同命名空间可以有不同的LimitRange，但是LimitRange无法限制命名空间中所有pod的资源总量，这需要ResourceQuota



ResourceQuota是用来限制某个命名空间**所有pod**的可用资源，限定命名空间中所有pod requests总和或者limits总和，但是要**注意的是**，这样pod必须限定resources字段，不然ResourceQuota无法确定已用的requests和limits量，或者明确给命名空间声明一个LimitRange



### HPA(horizontal pod autoscaler)

Autoscaler根据指定资源（通常是**CPU或者QPS（每秒查询率）**等）的metrics来**自动水平扩缩**pod（deployment或statefulset等资源控制的pod），**其通过根据从metric server采集到的pod的metrics来进行计算，从而改变deployment等资源的replicas**，达到自动扩缩pod的目的

不过要注意的是，这个要让指定资源扩缩到的目标值是需要**显式**的让管理员指定的，这个只能根据经验来确定

#### pod数量计算方式

- 单metric：如果是单资源计算的话，根据pod的`[实际使用资源量之和/目标值]`再向上取整
- 多metrics：多资源计算的话，取各单metric的计算出的数量的Max值



## ReplicationController

rc负责创建和管理pod的多个副本，其有三个部件：

- label selector（标签选择器）：

  用来标识pod，rc**只负责管理创建rc时指定标签的pod**（**只管自己家的孩子**），如果在这之后pod更改了标签，就脱离了rc的管理范围（pod相当于手动创建，无人管理，变成了孤儿），rc认为现有pod数与replicas不符合（即少了一个pod），会根据template创建新的pod

- rplica count（副本个数）：

  指定pod副本数

- pod template（pod模板）：

  用来创建新的pod，之后可以更改pod模板，只是创建出来的pod和之前的不一样了，rc并不关心pod长什么样，它只关心pod的副本数是不是符合预期

**如果pod数多了rc就会删除多余的pod（使符合replicas数），少了就会创建新的pod**

可以用kubectl scale改变副本数量来水平扩缩pod

## ReplicaSet

ReplicationController的替代品，具有更强的标签筛选功能，工作机理与rc相同，**以后用rs即可**

可基于标签名匹配，多标签匹配，单标签多值匹配，甚至匹配缺少某个标签的pod等（增加operator为in，notin，exists等等）

## DaemonSet

如果不指定selector，就会在**每个节点上管理一个pod（这些pod常用作来日志收集和资源监控）**，指定了对于node标签的selector，就只会在特定节点上每个节点管理一个pod

## Job & CronJob

**job是一个批处理任务**（可顺序可并行），用来安排一个或多个运行完后退出的pod

注意：

- 如果需要多次完成，可以指定completions字段，表示需要完成几次任务，job跑完一个pod后会再创建一个pod，以此类推（顺序执行），也可以指定parallelism数量并行执行

- **job中pod的restartPolicy必须为onFailure或者Never**，如果是onFailure，则容器异常退出后，pod仍会在原node上，而重启容器；如果是Never，则pod failed时，job会新建一个pod

- **job存在运行时限，如果超过了activeDeadlineSeconds这个时限，则job会视为失败**

**一般创建cronjob来创建job**，从而来创建pod，cronjob机制与linux crontab相同，会定时执行job

## Service & EndPoints

将相同功能的pod组（通过标签筛选）暴露一个固定的IP给外部或者内部集群访问（通过端口转发），**这样pod即使销毁或者转移，固定IP也不会变动**，当有client请求service时，service会随机选择一个pod进行转发

参考资料：https://www.qikqiak.com/post/visually-explained-k8s-service/

- 服务发现：

  - service如果早于pod创建，k8s会将为service维护的一些环境变量配置到容器当中
  - service也能通过kube dns通过域名解析来找到对应的ip（直接访问服务的FQDN，所有service对应的pod的容器当中，/etc/resolv.conf都有域名配置）

- Endpoints：

  Endpoints是**一对对的 ip：端口 对**（容器对外提供服务的那个端口），创建服务时是根据标签筛选器获得对应pod的ip：端口对，然后存储在Endpoints当中，当客户端连接service时，服务代理选择Endpoints当中的一个ip端口对，进行请求转发，也就是说，**服务有了Endpoints，才能将请求转发到相应的pod，没有标签筛选器筛选出对应的pod，服务只是个空服务而已**，如果更换或者移除了选择器，Endpoints也会跟着变动或者停止更新

- 将服务暴露给客户端：

  - Nodeport:将service的type设置为nodeport，k8s将会为每个节点上开一个指定端口，到该端口的流量都会重定向到此服务，然后再由服务重定向到pod

    Client --> 节点的指定端口 -->cluster服务--> pod

    既可以访问服务的集群ip来访问，也可以访问节点指定端口来访问

    ![image-20200917195909101](/Users/chenzicong/Library/Application Support/typora-user-images/image-20200917195910768.png)

    
  
  - LoadBalancer：将service的type指定为LoadBalancer，LoadBalancer需要有独有的公网ip，专门由负责负载均衡的机子来做重定向， 机制与nodeport一致（**只不过多做了一次数据流重定向**）
  
    Client --> LoadBalancer --> 节点的指定端口-->cluster服务--> pod
    
    ![image-20200917200648198](/Users/chenzicong/Library/Application Support/typora-user-images/image-20200917200648198.png)
  
  
  
  ​		总结：Load Balancer --> Node Port --> cluster service 层层往下
  
  - Ingress：
  
    通过外加的ingress controller来进行反向代理和负载均衡，当客户端向ingress发送http请求，根据主机名和路径分发到对应的服务（**这样既可以只用一个公网ip就可以为多服务做负载均衡和对外暴露，又可以避免nodeport开放过多端口可能造成的安全问题，因为运行k8s需要关闭防火墙**），通过与该服务相关联的Endpoint来查看ip，然后将客户端的请求转发给其中一个pod
  
    Client --> ingress controller --> pod
  
    Ingress controller实现方案： nginx，traefix，HAproxy等等
  
    ingress参考资料：https://www.cnblogs.com/linuxk/p/9706720.html
  
    ingress controller通常通过daemonset实现，在每个node上跑一个pod（如nginx），它会监听ingress的变化，一旦有新的ingress就根据模板生成配置然后写入pod中，然后reload。ingress就是转发规则，要发往哪些service，指定什么URL来转发到相应的service。
  
- headless服务：

  将服务的clusterIP字段设置为none，k8s便不会为其分配集群ip，用dns查找服务会返回所有pod ip，客户端会直接访问pod，而不是经过服务代理
  
- External Name:

  通过external name指定集群外部服务（DNS名），将其虚拟为内部服务（直接通过cluster ip访问即可）

## volume

**使用卷来让容器之间共享文件或者持久化文件**

### volume的种类

- emptyDir（**一个pod中的多个容器共享**）:

  emptyDir卷的声明周期与pod的生命周期相同，卷的声明是在创建pod时声明的，**pod删除卷也跟着删除**，卷从一个空目录开始，挂载到多个容器的某个文件夹后（文件夹在不同容器的文件系统的位置可以不一样），多个容器之间便可以往里面存放文件，以便共享

- gitRepo:

  gitRepo实际上是一个emptyDir卷，与emptyDir同理，只不过创建卷后卷中会加入从git远程repo中clone下来的文件，比如web服务容器挂载了这样一个卷，网页开发者往git远程仓库里push新的网页版本，sidecar容器（通常是git syc容器）便可以进行数据同步，以便web服务器对外服务（**其实就是emptyDir的基础上之后多加了git repo中clone下来的数据而已**）

- hostPath:

  hostPath可以提供持久化存储，不会随pod删除而删除，它访问的是节点上的文件系统，容器可以通过挂载hostPath卷访问节点上的某些特定文件夹下的文件，但是**不推荐使用hostPath来作为数据库数据等需要一直保存在目录上的数据**，因为如果pod被重新调度到其他节点上就找不到hostPath上的数据了，所以应该用pv

- configMap:

  将每个configMap中的键值对暴露为文件放到卷中，挂载到容器当中之后从而容器可以进行读取，与emptyDir和gitRepo同理
  
- PersistentVolume（持久化存储，后端存储）

  **PV是对分布式存储资源的抽象**，pv的提供者可以是各种云提供商，如AWS，GCE，也可以是nfs，glusterfs，ceph等开源分布式存储系统，具体实现具体对待。pv的信息包括存储能力（比如容量多大），访问模式（是ReadWriteOnce，ReadOnlyMany，还是ReadWriteMany），回收策略（与pvc解绑后对pv的处理是保留，回收还是删除）等。

  pvc是对pv的申请，可以与pv进行一一绑定，只要有pv满足pvc中申请的容量和访问模式，k8s就会将其绑定起来，pv一旦与pvc绑定起来就不能再去与其他pvc绑定了，但是如果pv大小不满足pvc申请量，pvc会一直pending，直到有足够大小的pv加入。**pvc就相当于是卷，然后pod使用其来挂载到对应目录，相当于是pod->pvc->pv三层结构**

  - 动态绑定（**使用StorageClass**)：

    原始的静态绑定如下：

    ![img](https://www.kubernetes.org.cn/img/2018/06/20180604211538.png)

    原始静态绑定需要管理员实现创建好pv，工作是非自动的，且pv的量是静态的，很有可能造成资源浪费，所以推荐使用动态绑定：

    ![img](https://www.kubernetes.org.cn/img/2018/06/%E5%9B%BE%E7%89%872.png)

    现在加入一个资源叫StorageClass，用户创建pvc后，storageclass的控制器会根据pvc指定的storageClassName来找到对应的存储类，然后动态的创建pv，再将其与pvc绑定，这样的好处是**既动态创建pv，减少了管理员的工作量，又使用存储类来抽象描述一种存储类别，比如是”快速存储“还是”慢速存储“，是”冗余存储“还是”无冗余存储“等等，非常直观**

    如果没有默认存储类，且pvc没有指定storageClassName，则这个pvc只能与没有指定存储类的pv绑定，否则没有指定storageClassName的pvc将交由默认存储类处理

    

## ConfigMap

将配置单独列为资源，方便移动和修改，**这样pod定义时不用定死和重复环境变量配置**，也增加了配置的重用性，而且使用环境变量和命令参数作为pod配置的方法无法在进程运行时更新配置，**configmap暴露为卷可以进行热更新,无需重新创建pod或重启容器**

**configmap中存放的就是key-value对**

kubectl create configmap [configmap-name]即可创建configmap（或者kubectl create -f xxx.yaml）

### 参数形式：

- --from-literal*key*value 直接用key*value形式定义变量

- --from-file*[file or directory] 读取文件为变量，如果没指定key，则直接用file的名字为key，value为file中的数据，如果读取的是整个文件夹，则文件夹下所有的文件都会被列为key*value

**--from-literal和--from-file都可以写多个**

### configmap的读取形式：

- pod的yaml中指定env或者envFrom来给容器当中的环境变量赋值，在这基础之上，还可以再传给pod中的yaml中的args来给命令参数赋值（**适用于变量值较短的场景**）
- 使用configmap卷来存放configmap中的所有键值对，然后挂载到容器当中（**推荐，将变量值整合成为文件**）。**需要注意的是，挂载会覆盖挂载点下原已有的文件，如果有需要不覆盖的话可以用subpath指定需要挂载的某个文件加到挂载点目录当中**

## Secret

原理与configmap类似，可以存放二进制数据，采用base64编码，专门用来存储敏感数据（安全），**configmap用来存储非敏感数据（一些配置）**

用secret卷挂载到容器当中即可引用secret内的key-value键值对，**最好不要用环境变量引用，因为环境变量会涉及到安全问题**

Tips:

- k8s通过将secret分发到需要访问secret的pod所在的节点的内存中，这样也方便删除
- 如果未给pod指定service account，k8s会给pod指定其所在namespace的default service account，会在每个容器的/var/run/secrets/kubetnetes.io/serviceaccount目录挂载一个默认secret卷:default-token-xxxx，这个secret中含三个条目，分别是：
  - ca.crt：CA的证书，当pod访问API服务器时（通过https），验证API服务器返回的证书就是CA签发的
  - namespace: pod所在的namespace
  - token: 用来通过api server的认证 

## DownwardAPI

使用configmap和secret无法让应用获取到**pod的ip，pod的名称，pod运行节点的名称**等等只有pod成功部署后才会获取到的元数据，通过DownwardAPI的方式可以暴露一个pod这些部分元数据

### 暴露方式：

- 直接在pod manifest的container当中声明环境变量，引用manifest当中的某些字段（环境变量方式）
- 通过挂载DownwardAPI卷来获取文件，从而获取元数据（卷方式，**卷方式才能暴露pod的标签和注解**）



## Deployment 

deployment用于部署应用并以声明的形式升级应用

创建deployment时，replicaset也会跟着创建，由rs来管理pod，deployment管理多个rs

**更改deployment的**pod模板**即可完成升级，由k8s的控制器操作完成**

### 升级方式

- 蓝绿部署

  在运行新版本的pod之前，service流量始终定位到旧版本pod，一旦确定新版本功能运行正常，修改service的选择器，定位到新版本的pod，删除旧版本的replicaset

  ![image-20201021161437006](/Users/chenzicong/Library/Application Support/typora-user-images/image-20201021161437006.png)

- 滚动升级

  新的rs控制新版本的pod，通过更改新旧rs的期望数量来弹性扩缩旧版本与新版本的pod数量，新版本pod一点点增多，旧版本pod一点点减少，直到所有更新为止

  ![6D21A0ED0EAF37931D60F5EF5098258D](/Users/chenzicong/Library/Containers/com.tencent.qq/Data/Library/Caches/Images/6D21A0ED0EAF37931D60F5EF5098258D.jpg)

- 金丝雀升级

  一小部分pod升级为新版本，牺牲一小部分用户的体验（beta版本），如果新版本适用则全部升级完成，否则回滚到上个版本

### kubectl rollout 命令

进行升级deployment时，可以通过kubectl rollout进行升级查看和回滚等

- kubectl rollout status deployment [name]

  查看滚动升级状态

- history deployment [name] 

  查看升级历史（可以指定revision来回滚指定版本）

- pause 暂停滚动升级

- resume 恢复滚动升级

- undo 回滚到上个版本

  可添加--to-revision参数回到history给出的指定revision版本

### deployment manifest重要字段

- maxSurge

  除deployment定义的期望值外，最多允许超过的pod可用实例数量

- maxUnavailable

  相对于期望值，最多不可用pod的数量

可以通过修改maxSurge和maxUnavailable控制滚动升级速度

- minReadySeconds

  pod至少要运行minReadySeconds秒，才能视其为可用状态，因为maxUnavailable限制了不可用pod数量，所以不可用的话，滚动升级不会继续

## StatefulSet

**用来维护有状态应用**，因为有状态应用可能需要主机名，ip等不变

statefulset维护的pod副本名是可预知的，按照statefulset名称加顺序索引值组成，每次扩缩容都是扩缩索引最高的那个

statefulset对于有状态应用做出的改变：

- 对于ip可能的改变：

  需要你除statefulset之外额外定义一个headless service，然后根据域名来访问各个pod，这样每个pod的dns记录固定

- 对于主机名维持不变

- 对于pod删除而存储可能发生的变动：

  如果statefulset维护的pod被删除了，与replicaset相同，statefulset会重新创建pod，但这个pod可能会调度到其他节点上，对于有状态应用来说，有状态的pod应该挂载原来相同的存储，这就需要pvc和pv，创建一个statefulset的时候，不仅需要写pod模板，还需要写pvc，**statefulset对每个pod都创建一个或多个独立的pvc**，因为挂载的pvc不变，所以新实例读取的仍然是旧实例的数据

  注意：

  如果pod被意外删除，**其挂载的pvc不会被删除**，因为如果pvc被删除，其绑定的pv可能会被回收或者删除，那其上的数据就丢失了，所以无论是缩容还是意外删除了statefulset的pod，都会保留pvc，意外缩容了statefulset之后仍然能通过扩容，将pod重新挂载原来丢失了挂载的pvc，但是statefulset只能**线性缩容**，不能在有一个实例不健康的时候进行缩容，这时同时有两个pod下线，那数据就丢失了。如果想删除pv，需要**手动**删除绑定的pvc



## ServiceAccount

user account是给集群外部用户设计的，而service account是为了**pod访问api server**而设计的（每创建一个service account就会有对应的一个secret被创建，这个secret当中内含ca.cert，namespace，token，挂载点/var/run/secrets/kubernetes.io/serviceaccount，跟secret章节中提到的default token是一样的）

**tips:** 

Api server的访问过程需要经过的插件：认证(authenticaiton)-->授权(authorization)-->准入控制(admission control)

当创建一个namespace时，就会创建一个默认的default serviceaccount（每个namespace底下一个），所以service account是局限于其所在的namespace的。当创建一个pod时，默认指定的service account就是pod所在namespace底下的default（除非指定某个service account），**这个service account在默认没有绑定rolebinding或者clusterrolebinding的情况下，是没有查看和修改集群资源权限的**

### RBAC

四种RBAC资源（role可以理解为对于资源的权限，binding就是将权限赋予某个account）

#### 某个具体命名空间

- role:**对仅限于某个特定命名空间**的资源的操作权限
- rolebinding:**仅限于某个特定命名空间**的权限绑定

#### 集群范围或任意命名空间

- clusterrole:**对于集群资源或所有命名空间资源或非资源型URL**的操作权限
- clusterrolebinding:**对于集群资源或所有命名空间资源或非资源型URL**的权限绑定

**当binding绑定到某个service account或user account之后，这个account就获得了对于role规定的权限**

![img](https://tva1.sinaimg.cn/large/008eGmZEly1gni8tkzq8nj32c30u04qq.jpg)

一些重要的系统自定clusterrole：

- view：除了role，rolebinding和secret外可以读取大部分资源
- edit：除了role，rolebinding外可以读取和修改大部分资源
- admin: 除了ResourceQuota能读取和修改所有资源
- cluster admin:完全能读取和修改所有资源



