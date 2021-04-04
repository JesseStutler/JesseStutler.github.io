---
title: "Docker基础知识"
date: 2021-03-07T10:56:15+08:00 
description: "对于时下最常用的容器实现技术docker的介绍"
tags: [docker]
featured_image: "/images/docker-logo.jpg"
images: ["/images/docker-logo.jpg"]
categories: docker 
comment : true
---

# Docker

## Docker拉取镜像流程图

![截屏2020-08-12 上午10.33.14](https://tva1.sinaimg.cn/large/008eGmZEgy1gob5syundsj318o0ogn1v.jpg)

## Docker CLI

### 镜像命令

- `docker images 查看本地的镜像`

  docker images [image-name[:tag]]
  默认不加参数就是-a，或者指定image的名字，可在image之上再加版本号
  --a 列出所有镜像
  --q [image] 列出镜像的id（-aq是列出所有的镜像id）

- `docker search 镜像`

  搜索远程仓库镜像（docker hub上查看更详细）

- `docker rmi [repo[:tag]] `

  删除本地镜像，使用方法与images相同，注意rmi是删除镜像，rm是删除容器

  或者根据docker images -q [image]列出的id进行删除

- `docker tag source_image[:tag] target_image[:tag] `

  改镜像标签名（不然默认传到docker hub上的library仓库会被拒绝）

- `docker pull 镜像`

  默认拉取的是docker hub上的Image，也可以用一个容器跑一个local docker registry，然后让其他使用了docker pull并指定了docker registry地址和端口的机子从这台运行了docker registry的机子上拉镜像

- `docker push 镜像`

   将镜像上传到docker hub上的仓库或指定仓库

- `docker save -o [tar文件名] [镜像名] `

  用来将镜像保存到tar文件当中，可以指定一个或多个镜像名
  也可以使用gzip压缩tar，使用docker save [镜像名] | gzip > [xxx.tar.gz]

- `docker load -i [tar文件名或gzip等压缩格式] `

  用来读取文件形成镜像

### 容器命令

> **docker run 创建容器并运行**

```shell
docker run [选项] 镜像名 [参数]
-it 获取stdin并运行一个伪tty(默认是/bin/bash，可在命令最后自己指定shell)
--name 为容器指定一个名字(最好加上方便些，不然就要根据container id来指示是哪个容器)
最好加上自己docker hub上的repo名，这样可以直接push到自己的repo，而不用tag再改镜像名，如 jesse/centos:tag
-d 后台运行，如果容器没有对外提供服务或运行前台程序则会立刻停止
-p 将容器内的端口与host os的端口绑定起来，如-p host_port:container_port可以通过访问host的某个端口来访问容器内的某个服务
-P 大写P,将镜像指定的暴露端口与host os上的随机端口绑定起来，一般需要Dockerfile中有EXPOSE进行结合
-v 将容器内的某个文件夹与host os的文件夹绑定起来（即数据同步，相当于挂载）
1.如-v host_dir:container_dir 指定路径挂载，可以将容器内的指定文件夹数据同步到host os的指定目录上，这样删除容器时不至于数据丢失（比如容器内装有Mysql，这样删除容器就相当于把整个数据库都删了）
2.如-v volume_name:container_dir 具名挂载，会指定卷名，可以通过docker volume inspect volume_name查看在Host os上的目录位置
--volumes-from 继承已有的容器的数据卷（共享目录）
e.g:
docker run --name centos -it centos /bin/bash
```

> **docker ps 查看容器**

```shell
不指定参数就是查看正在运行的容器
-a 查看所有容器（包括停止的容器）
-q 列出容器id
```

> docker rm [container id 或者 container name] 删除容器

```shell
不加参数需要指定container id或者container name，正在运行的容器无法删除
-f 强制删除容器
docker ps -aq | xargs docker rm -f 强制删除所有容器
```

> docker start 启动容器
>
> docker restart 重启容器
>
> docker stop 停止容器

> docker logs 查看日志

```shell
-f 实时跟随
-t 显示时间
```

> docker top 显示容器内正在运行的进程

> docker inspect 显示容器的所有**配置信息（元数据）d**

> **进入正在运行的容器**

```shell
1.docker exec [container] [command]
在容器里跑一个命令
-it 与docker run相同，获取stdin并进入伪tty
e.g:
docker exec -it [container id或者container name] bash ———— 在容器里跑一个新的bash并进入

2.docker attach [container]
直接进入到container当中
```

> docker cp 将容器中的文件拷贝到外部os或将外部os的文件拷贝至容器中

`docker cp container:path dest_path`

> docker commit 由现有容器制作镜像

```shell
-a 指定作者名
-m 这次提交要发布的消息
docker commit [选项] container [image[:tag]]
```

### 卷命令

> docker volume 

```shell
特别说明：不同的容器可以指定相同的volume
ls 列出所有的卷
inspect 查看指定卷的信息（可查看挂载在host os上具体哪个位置）
```



## Docker基本命令图

![image-20200812163714285](https://tva1.sinaimg.cn/large/008eGmZEgy1gob60cd4z5j316t0u0k9j.jpg)



## Docker层的概念

![img](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1597383379693&di=8a718adcadd59dbfafaf89136bcf6efe&imgtype=0&src=http%3A%2F%2Fimg2020.cnblogs.com%2Fblog%2F2030366%2F202006%2F2030366-20200630103737429-2119801149.png)



## Dockerfile命令

tips:

- 命令最好大写，写一个命令代表构建

- 选用合适的镜像，不要选用过大的父镜像从而构建出臃肿庞大的镜像，选用小巧的父镜像（如alpine、busybox、debian等）

- **使用多阶段构建形式（多个FROM命令，以最后一个FROM为根），例如将编译环境和构建环境分离开来可以避免镜像过大（因为构建环境只需要编译出来的可执行文件，而不需要编译所需的库和命令等一大堆文件）**

  COPY --from=指定的阶段名

  可以将指定阶段中编译好的文件拷贝到当前构建环境中，甚至可以从指定镜像中进行拷贝

  e.g:

  ```dockerfile
  # 编译阶段 命名为 builder
  FROM golang:1.10.3 as builder
   
  # ... 省略
   
  # 运行阶段
  FROM scratch
   
  # 从编译阶段的中拷贝编译结果到当前镜像中
  COPY --from=builder /build/server /
  ```

  

> **docker build** 根据Dockerfile构建镜像命令

```shell
docker build [选项] URL|PATH
-f 要用哪一个Dockerfile（指定Path）
-t 给镜像取名字，可加tag（name:tag）
e.g:
docker build -f Dockerfile -t name:tag . 将当前的文件夹作为环境构造镜像
可以以PATH指定的目录作为上下文环境来构建镜像，最好是以空目录为上下文环境，里面只放Dockerfile（除非是放生成镜像必须的文件，因为上下文过大的话发送给daemon会很慢），也可以是指定URL作为构建环境，也就是Git Repository，比如Github
```

> **FROM**

```dockerfile
FROM image
Dockerfile都要以FROM开始，构建一个基本镜像
e.g:
FROM scratch
这是Docker Hub上大多数镜像的选择，构建一个默认镜像
```

>  **RUN**

```dockerfile
RUN shell_command
build镜像时所要执行的命令
一般以执行shell命令的格式最佳,默认是以RUN /bin/sh -c command形式执行的
一行写不下时可以通过\换行,执行多个命令最好用&&来相连，这样可以少些一些RUN，因为RUN、ADD、COPY这些命令写一个命令就会加一层，这样会使容器不那么臃肿庞大
```

>CMD

```dockerfile
CMD ["executable","param1","param2"]一般以这种格式最佳（注意都要双引号，因为是JSON格式）
CMD是容器运行时所要执行的命令，旨在指定run容器时所要做的默认动作，跟docker run后跟一个command原理是一样的
特别说明：Dockerfile中只能有一个CMD,如果有多个CMD，只有最后一个生效
```

> ENTRYPOINT

```
使用方法与CMD相同，但是docker run后面如果跟command会把CMD要执行的命令覆盖掉，ENTRYPOINT要执行的命令不会，甚至能给ENTRYPOINT追加参数
```

> LABEL

```dockerfile
LABEL key="value" key="value" ... key="value"
给镜像加元数据，以键值对的形式多组追加，可以通过\换行,比如在LABEL里加上maintainer（维护者）等信息
可以通过docker inspect来查看
```

> EXPOSE

```dockerfile
让运行时的容器在指定端口进行监听，也就是暴露端口,默认是tcp，可以指定udp，如EXPOSE 53/udp
可以通过docker run中的-p来覆盖，也可以通过-P将host os上的随机端口与暴露的端口进行绑定
```

> **ENV**

```dockerfile
为运行的容器设置环境变量
ENV key value 一个ENV设置一个环境变量
```

> ADD

```dockerfile
ADD <src>...<dest>
将src的文件，文件夹拷贝到dest，src可以是一个remote URL，dest可以是相对路径也可以是绝对路径，如果是相对路径那就是拷贝到WORKDIR/相对路径下
特别说明：dest如果是文件夹，想把文件拷贝到dest文件夹下，必须在最后加/

COPY指令与ADD用法相同，但是ADD可以从远程url上拷贝，COPY只能拷贝本地宿主机上的文件
```

> WORKDIR

```
设置进入容器时的工作目录，能使用ENV设置的环境变量
```



## 容器网络配置



> docker network 

```shell
connect 将一个容器部署进某个配置的网络当中，通过ls查看网络名
ls 列出所有网络
inspect 查看网络描述信息
create 创建网络，可通过-d指定方式，有bridge，host等等，可通过--subnet指定子网和后缀
...
要让容器使用docker network创建的网络，在docker run时指定--net即可
```

