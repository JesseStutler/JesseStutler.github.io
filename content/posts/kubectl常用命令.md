---
title: "kubectl常用命令"
date: 2021-03-07T10:30:17+08:00 
tags: [k8s]
featured_image: "/images/k8s-logo.png"
images: ["/images/k8s-logo.png"]
categories: k8s
comment : true
---

# k8s命令

## 引言

本文列举了一些kubectl的常用命令以及其对应的常用参数

kubectl的详细信息可参照：https://kubernetes.io/docs/reference/kubectl/overview/

如果熟悉了kubectl的朋友，对于经常性的`kubectl get `和`kubectl describe`查找resource感到繁琐，笔者在这里推荐一款开源的增强型kubectl软件k9s：https://github.com/derailed/k9s，相信对于vim熟悉的朋友会喜欢这款开源软件，而且可以当简略的dashboard使用

## kubectl

- `kubectl create -f [yaml或者json文件]`

  通过yaml或者json文件创建一个组件

  -n 指定命名空间，如果不指定，默认是在default命名空间下，其他命令也一样

- `kubectl get [组件] [组件名]`
  
  获取组件的基本信息，如果想获取详细信息用kubectl describe
  
  -o wide 显示更多信息，-o yaml 以yaml格式显示组件信息
  
  --show-labels 多显示标签
  
  -l 标签键=值 根据标签来筛选出pod基本信息，多个键值对用逗号分隔
  
  -L [标签名] 多显示指定标签名的标签列，多个标签用逗号分隔
  
  --all-namespaces 列出所有命名空间的组件
  
- `kubectl logs [podname]`

  查看pod内容器的日志输出，如果只有一个容器不用指定容器名，如果有多个容器，想查看指定容器的日志需要-c参数指定

- `kubectl label [组件] [组件名] key=value [--overwrite]`

  修改或添加组件的标签，用key=value形式，如果要复写之前的标签，需要多加一个--overwrite

  如果要删除之前的标签，直接在key后跟一个减号即可（即key-）

- `kubectl delete [组件] [组件名1] [组件名2] [...]`

  删除组件

  -all 删除所有组件

  注：删除命名空间，里面的组件也会一并删除

- `kubectl scale [组件] [组件名] --replicas`

  设置组件管理的资源数量，组件可以是Deployment, ReplicaSet, Replication Controller, StatefulSet or job

- `kubectl exec [pod-name] -- [pod中的容器中需要执行的命令]`

  在指定pod中的容器中运行一个命令，这里--表示kubectl的结束，如果不写的话可能会被kubectl当成参数处理，会造成歧义

  -c 指定容器

- `kubectl explain [source]`

  查看资源文档，包括资源的manifest有哪些字段，apiVersion是多少等等

### 修改source配置的命令

- `kubectl apply -f [文件]`

  修改kubernetes资源，如果没有创建资源会创建资源，如果已经创建了资源则进行修改，文件可以是yaml文件

- `kubectl edit [source] [source-name]`

  用vi编辑source的yaml配置

- `kubectl patch [source] [source-name] -p`

  使用JSON格式修改或添加source属性

  **改单个属性的时候很方便，不用像edit一样要打开Yaml找到对应属性**

- `kubectl set image [source] [source-name] [container-name]=[new image]`

  将source中的容器镜像修改为新的镜像



