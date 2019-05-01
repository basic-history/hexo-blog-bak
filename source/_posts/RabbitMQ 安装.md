
---
title: RabbitMQ 安装
date: 2017-09-28 12:00:00
author: pleuvoir
img: /images/code.jpg
tags:
  - Linux
  - RabbitMQ
  - 消息中间件
categories:
  - 技术
---

在 Linux 中安装 RabbitMQ 演示 操作系统版本为 `CentOS7`

### 安装 erlang 和 RabbitMQ

```
# 切换用户，这一步看情况
su - root

## 安装
wget https://packages.erlang-solutions.com/erlang-solutions-1.0-1.noarch.rpm

// 如果失败，那么继续执行下一条
rpm -Uvh erlang-solutions-1.0-1.noarch.rpm

yum install epel-release

yum install erlang

wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.6/rabbitmq-server-3.6.6-1.el7.noarch.rpm

yum install rabbitmq-server-3.6.6-1.el7.noarch.rpm
```

至此， `RabbitMQ` 已经安装完成了，现在我们把它启动一下。因为是直接下载，所以可能会失败是正常的，多次尝试即可。


### 管理
```
# 启动
service rabbitmq-server start

# 查看状态
service rabbitmq-server status

# 安装管理控制台
rabbitmq-plugins enable rabbitmq_management  

# 如果需要做推送可以开启 stomp 代理（这一步不是必须的）
rabbitmq-plugins enable rabbitmq_web_stomp

# 重启RabbitMQ
service rabbitmq-server stop
service rabbitmq-server start

# 开启相对应的端口，方便外网查看
 （如果是 aliyun 可以参考开放端口 https://jingyan.baidu.com/article/03b2f78c31bdea5ea237ae88.html）
 （如果压根就没开启可以参考进行开启，当然下面的两行也就不需要了 https://jingyan.baidu.com/article/5552ef47f509bd518ffbc933.html）
firewall-cmd --permanent --add-port=15672/tcp
firewall-cmd --permanent --add-port=5672/tcp

# 查看已有虚拟主机并增加名为 cc 的虚拟主机
rabbitmqctl list_vhosts
rabbitmqctl add_vhost cc
rabbitmqctl list_vhosts

# 增加名为 pleuvoir 的用户并配置 administrator 角色并增加相应的权限
rabbitmqctl add_user pleuvoir 123456
rabbitmqctl set_permissions -p cc pleuvoir '.*' '.*' '.*'
rabbitmqctl set_user_tags  pleuvoir  administrator
```

登录 host:15672 即可查看管理控制台

如果需要在 windows 下安装，可以参考 [windows 下安装 RabbitMQ](https://github.com/pleuvoir/reference-samples/tree/master/spring-amqp-example)