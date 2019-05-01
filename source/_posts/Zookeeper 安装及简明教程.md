
---
title: Zookeeper 安装及简明教程
date: 2019-02-18 12:00:00
author: pleuvoir
img: /images/code.jpg
tags:
  - Zookeeper
categories:
  - 技术
---

## Zookeeper 安装

ZooKeeper的安装模式分为三种，分别为：单机模式（stand-alone）、集群模式和集群伪分布模式。

### windows 安装

jdk 是必须的。

[下载](https://www.apache.org/dyn/closer.cgi/zookeeper/) 

完成后解压。

如解压的后的目录是 `D:\01 dev\zookeeper-3.4.13` ，建议在 `D:\01 dev` 新建 data 及 log 文件夹。

复制 `D:\01 dev\zookeeper-3.4.13\conf\zoo_sample.cfg` 文件到当前目录下，并改名为 `zoo.cfg`。

修改 `zoo.cfg` 文件，改为如下配置：

```
## 快照文件位置
dataDir=D:/01 dev/data
## 事务日志位置
dataLogDir=D:/01 dev/log
```

此时安装已经完成了。

### Linux 安装

```shell
## 下载
wget mirrors.shu.edu.cn/apache/zookeeper/zookeeper-3.4.13/zookeeper-3.4.13.tar.gz
## 解压文件
tar -xzvf zookeeper-3.4.13.tar.gz

cd zookeeper-3.4.13/conf
cp zoo_sample.cfg zoo.cfg

## 此时配置不配也可以启动
./zkServer.sh start
```

此时安装已经完成了。

### Linux 集群安装

1）安装jdk运行jdk环境

`上传jdk1.8安装包`

2）安装jdk1.8环境变量

```
vi /etc/profile

export JAVA_HOME=/usr/local/jdk1.8.0_181
export ZOOKEEPER_HOME=/usr/local/zookeeper
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

刷新profile文件

source /etc/profile

关闭防火墙
```

3）下载zookeeper安装包

`wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.10/zookeeper-3.4.10.tar.gz`

4）解压Zookeeper安装包

`tar -zxvf zookeeper-3.4.10.tar.gz`

5）修改Zookeeper文件夹名称
 
`重命名： mv zookeeper-3.4.10 zookeeper`

6）修改zoo_sample.cfg文件

```
cd /usr/local/zookeeper/conf
mv zoo_sample.cfg zoo.cfg
修改conf: vi zoo.cfg 修改两处
（1） dataDir=/usr/local/zookeeper/data（注意同时在zookeeper创建data目录）
（2）最后面添加
server.0=192.168.212.154:2888:3888
server.1=192.168.212.156:2888:3888
server.2=192.168.212.157:2888:3888
```

7）创建服务器标识
```
服务器标识配置：
创建文件夹： mkdir data
创建文件myid并填写内容为0： vi
myid (内容为服务器标识 ： 0)
```

8）复制zookeeper

```
进行复制zookeeper目录到node1和node2
还有/etc/profile文件
把node1、 node2中的myid文件里的值修改为1和2
路径(vi /usr/local/zookeeper/data/myid)
```

9）启动zookeeper

```
启动zookeeper：
路径： /usr/local/zookeeper/bin
执行： zkServer.sh start
(注意这里3台机器都要进行启动)
状态： zkServer.sh 
status(在三个节点上检验zk的mode,一个leader和俩个follower)
```

此时安装已经完成了。

### 目录结构

```
bin            存放系统脚本
conf           存放配置文件
contrib        zk附加功能支持
dist-maven     maven仓库文件
docs           zk文档
lib            依赖的第三方库
recipes        经典场景样例代码
src            zk源码
```

其中 bin 和 conf 是非常重要的两个目录，平时也是经常使用的。


#### bin

![](https://i.loli.net/2019/02/20/5c6d009de9df4.jpg)

其中 `zkServer` 为启动服务端，默认端口为 2181；`zkCli` 是客户端命令行工具。

#### conf 

conf 目录为配置文件存放的目录，zoo.cfg 为核心的配置文件

![](https://i.loli.net/2019/02/20/5c6d00acbabf9.png)
![](https://i.loli.net/2019/02/20/5c6d00c51cdeb.png)
![](https://i.loli.net/2019/02/20/5c6d00cd6252a.png)
![](https://i.loli.net/2019/02/20/5c6d00d3da7bc.png)
![](https://i.loli.net/2019/02/20/5c6d00db36d50.png)


## ZK 特性

### 数据节点

在 Zookeeper 中，数据结构类似于树，每一个节点叫 `ZNode`,每一个节点都是有值的。

我们可以在 Windows 下使用 `ZooInspector` 客户端工具查看。

![ZooInspector](https://i.imgur.com/jLRsYQX.png)

#### 节点类型

1）临时节点

  当与客户端断开连接会 ZNode 自动删除

2）持久节点
  
  当与客户端断开连接会 ZNode 还在

3） 顺序节点

  当两个同样的命令执行会自增，`create /node1 0000001`； `create /node1 0000002`；非顺序节点会报错

当节点名称带序号时，创建同样的节点会自增，如果不带序号创建同名节点会报错。也就是说，<b>对于持久节点和临时节点，同一个 Znode 下，节点名称是唯一的。<b>


### Watcher

Watcher 即为 Zookeeper 发生节点变更时触发的事件，利用此机制我们可以监听变化做出操作。具体使用详见 

## 高级应用

配置中心，服务注册与发现，分布式锁，选举。



以上所有内容可参考[代码示例](https://github.com/pleuvoir/reference-samples/tree/master/zk-sample)
