---
title: docker入门
date: 2019-05-24 12:00:00
author: pleuvoir
tags:
  - Linux
  - docker
categories:
  - 技术
---

### 安装docker

docker支持CentOS6 及以后的版本。


如果之前安装过则先卸载，如果是第一次安装则可跳过

```
## 查询是否安装过docker
yum list installed | grep docker

## 删除安装过的docker如果有的话
yum -y remove docker-engine.x86_64

## 删除镜像容器
rm -rf /var/lib/docker
```

这里`docker-engine.x86_64`即为查询出的一个结果


在不同的linux版本上安装方式不太相同，因为centos7自带了docker。查看linux版本，`cat /etc/redhat-release`

1. 如果是centos6：

```
sudo yum install http://mirrors.yun-idc.com/epel/6/i386/epel-release-6-8.noarch.rpm
sudo yum install docker-io
```

2. 如果是centos7：


```
## 安装最新版

## sudo yum install docker 这样安装不是最新版

sudo yum install -y yum-utils device-mapper-persistent-data lvm2

## 配置仓库速度更快
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo 

sudo yum install docker-ce 

```


查看docker版本

```
docker version
```

提示`Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?`，因为我们没有安装daemon。


启动docker

```
sudo service docker start
```

查看docker信息

```
docker info
```

设置随系统开机启动

```
sudo chkconfig docker on
```

至此安装完成。

3. windows

<https://hub.docker.com/editions/community/docker-ce-desktop-windows>

如果不是window10，可以使用boot2Docker工具


### docker体验


入门hello-world镜像

```
docker run hello-world 
```

会出现以下画面：

```
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

恭喜，成功来到了docker世界。其中`hello-world`是官方提供的一个镜像。

### docker基本操作

#### 镜像操作

docker images|rmi|tag|build|history|save|load]

- images：列出本地镜像列表
- rmi [镜像名：版本]：删除镜像
- tag [镜像名：版本] [仓库]/[镜像名：版本]：标记本地镜像，将其归入某一仓库
- build -t [镜像名：版本] [path]：Dockerfile 创建镜像
- history [镜像名：版本]: 查看指定镜像的创建历史
- save -o xxx.tar [镜像名：版本] /  save [镜像名：版本]>xxx.tar : 将镜像保存成 tar 归档文件
- load --input  xx.tar / docker load<xxx.tar : 从归档文件加载镜像

#### 容器操作

##### 容器生命周期管理

docker [run|start|stop|restart|kill|rm|pause|unpause]


*	run/create[镜像名]：  创建一个新的容器并运行一个命令
*	start/stop/restart[容器名]：启动/停止/重启一个容器
*	kill [容器名]： 直接杀掉容器，不给进程响应时间
*	rm[容器名]：删除已经停止的容器
*	pause/unpause[容器名]：暂停/恢复容器中的进程

##### 容器运维

docker [ps|inspect|exec|logs|export|import]

*	ps：查看容器列表（默认查看正在运行的容器，-a查看所有容器）
*	inspect[容器名]：查看容器配置元数据
*	exec -it [容器名] /bin/bash：进入容器环境中交互操作
*	logs --since="2019-02-01" -f --tail=10 [容器名]:查看容器日志 
*	cp path1 [容器名]:path 容器与主机之间的数据拷贝
*	export -o test.tar [容器名] / docker export [容器名]>test.tar : 文件系统作为一个tar归档文件
*	import test.tar [镜像名:版本号]:导入归档文件，成为一个镜像

#### 仓库操作

*   docker [login|pull|push|search] 

#### 实战

```bash
#查看所有镜像
docker images

#下载centos镜像
docker pull centos 

#第一种启动进入方式（前台运行）：启动容器并进入内部交互环境
docker run -it --name centos1 centos /bin/bash

#退出 ctrl+p+q会退出但不关闭容器 如果直接exit会关闭容器 

#现在没有退出，在外部退出（优雅关闭，也可以直接kill）
docker stop centos1

#再次启动,但不会进入shell
docker start centos1

#第二种启动进入方式（后台运行，-d代表后台运行，但必须配合it使用,it会拉起前端进程不会导致它自动退出）
docker run -dit --name centos1 centos

#再次进入
docker exec -it centos1 /bin/bash

# 查看容器的元数据信息
docker inspect centos1

#查看正在运行的容器
docker ps

#查看所有容器
docker ps -a

#删除容器（可以批量删除）
docker rm b098c66ef455 9e995983f718 b88bafdf8715

#删除镜像
docker rmi 9e995983f718
```

### QA

1. 下载镜像特别慢？

   配置daemon地址为国内。

   ```bash
   vi /etc/docker/daemon.json
   
   增加内容，直接复制即可，这个文件可能没有创建即可
   {
     "registry-mirrors": ["https://registry.docker-cn.com"]
   }
   
   #重启
   service docker restart
   ```