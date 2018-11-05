
---
title: Nginx（一） 简介与安装
date: 2016-09-28 12:00:00
author: pleuvoir
img: /images/code.jpg
tags:
  - Linux
  - Nginx
categories:
  - 技术
---



### 一、Nginx 简介

#### Apache

Apache 仍然是市场占用量最高的 web 服务器，据最新数据统计，市场占有率目前是 50% 左右。主要优势在于一个是比较早出现的一个 Http 静态资源服务器，同时又是开源的。所以在技术上的支持以及市面上的各种解决方案都比较成熟。Apache 支持的模块非常丰富。

#### Nginx

Nginx 是俄罗斯人编写的一款高性能的 HTTP 和反向代理服务器，在高连接并发的情况下，它能够支持高达 50000 个并发连接数的响应，但是内存、CPU 等系统资源消耗却很低，运行很稳定。目前 Nginx 在国内很多大型企业都有应用，据最新统计，Nginx 的市场占有率已经到 33% 左右了。而 Apache 的市场占有率虽然仍然是最高的，但是是呈下降趋势。而 Nginx 的势头很明显。选择 Nginx 的理由也很简单：第一，它可以支持 5W 高并发连接；第二，内存消耗少；第三，成本低，如果采用 F5、NetScaler 等硬件负载均衡设备的话，需要大几十万。而 Nginx是开源的，可以免费使用并且能用于商业用途。


### 二、架构中的作用

nginx 在系统架构（网关入口）中的作用，总结如下：

1. 路由功能（与微服务对应）：域名/路径，进行路由选择后台服务器
2. 负载功能（与高并发高可用对应）：对后台服务器集群进行负载
3. 静态服务器（比 tomcat 性能高很多）：将静态文件单独做一个服务器；在 mvvm 模式中，充当文件读取职责

总结：实际使用中，这三种功能会混合使用。比如先分离动静，再路由服务，再负载机器。


### 三、正向代理与反向代理

1. 代理：客户端自己请求出现困难。客户请了一个代理，来代自己做事，就叫正向代理。
    比如代理律师，代购，政府机关办事的代理人等等。

2. 反向代理，服务端推出的一个代理招牌（服务器请来的）。


### 四、Nginx 安装

#### 源码编译安装方式（推荐）

```bash
# 进入自己准备存放下载文件的目录
cd /user/local/src

# 如果 wget 不能用，可先安装 yun -y install wget
wget http://nginx.org/download/nginx-1.9.15.tar.gz

# 下载完成后解压
tar zxvf nginx-1.9.15.tar.gz

# 按顺序安装依赖
## 正则相关
yum -y install pcre pcre-devel    
yum -y install zlib zlib-devel
## https 相关
yum install -y openssl openssl-devel

# 进入
cd nginx-1.9.15

# 编译，指定安装位置 /usr/local/nginx，指定安装模块 https
./configure   --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module 

# 当出现 creating objs/Makefile 表示编译成功，然后安装
make && make install

# 查看 nginx 安装后的位置
whereis nginx（// 此时在 nginx: /usr/local/nginx）
```

至此 nginx 就安装完成了。进入安装后的目录，执行 `sbin/nginx` 启动，如果 80 端口没有被占用那么访问服务器 ip，就能看见 nginx 的欢迎页

#### yum 方式（简单但不建议使用）

```bash
# centos6 系统是红帽，它维护了一个 yum 扩展源，但是这个没原始的包那么安全，并且依赖比较分散
yum install epel-release -y  
yum install nginx -y
```
