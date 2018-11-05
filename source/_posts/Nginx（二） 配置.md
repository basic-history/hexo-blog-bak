
---
title: Nginx（二） 配置
date: 2016-09-29 12:00:00
author: pleuvoir
img: /images/code.jpg
tags:
  - Linux
  - Nginx
categories:
  - 技术
---

### 一、 目录结构

源码包编译以后的目录结构如下：

•	conf   配置文件
•	html   静态网页文件（存放静态文件，做静态资源服务）
•	logs   日志文件
•	sbin   二进制程序


nginx 中最重要的配置文件是 conf 中的 `nginx.conf`，基本上以后只需要和它打交道。

### 二、 nginx 启停

```bash
进入 sbin
# 检测配置文件是否有错误
./nginx -t 

# 指定配置文件启动，一般是 NGINX_HOME 下 /conf/nginx.conf
./nginx -c nginx.conf 的文件

# 停止
./nginx -s stop  

# 退出
./nginx -s quit

# 重新加载 nginx.conf
./nginx -s reload 
```

### 三、 nginx.conf 配置文件结构

main（全局设置）
events 设定 nginx 的工作模式及连接数上限 
http 服务器相关属性 
server （虚拟主机设置）
upstream （上游服务器设置，主要为反向代理、负载均衡相关配置） 
location （URL匹配特定位置后的设置）

![nginx 配置文件结构](https://i.imgur.com/wVQmJiM.png)