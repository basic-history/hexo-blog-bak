
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
./nginx -c [nginx.conf文件]

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

```
#user  nobody;  				#主模块命令， 指定Nginx的worker进程运行用户以及用户组，默认由nobody账号运行。
worker_processes  1;			#指定Nginx要开启的进程数。
worker_rlimit_nofile 100000;    #worker进程的最大打开文件数限制
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
#pid        logs/nginx.pid;
events {
use epoll;
worker_connections  1024;
}


以上这块配置代码是对nginx全局属性的配置。

user: 主模块命令，指定Nginx的worker进程运行用户以及用户组，默认由nobody账号运行。    
   
worker_processes: 指定Nginx要开启的进程数。

error log:
用来定义全局错设日志文件的路径和日志名称。日志输出级别有debug，info，notice，warn，error，crit 可供选择，其中debug输出日志最为详细，面crit（严重）输出日志最少。默认是error

pid: 用来指定进程id的存储文件位置。

event：
设定nginx的工作模式及连接数上限，其中参数use用来指定nginx的工作模式（这里是epoll，epoll是多路复用IO(I/O Multiplexing)中的一种方式）,nginx支持的工作模式有select ,poll,kqueue,epoll,rtsig,/dev/poll。其中select和poll都是标准的工作模式，kqueue和epoll是高效的工作模式，对于linux系统，epoll是首选。
worker_connection是设置nginx每个进程最大的连接数，默认是1024，所以nginx最大的连接数max_client=worker_processes * worker_connections。
进程最大连接数受到系统最大打开文件数的限制，需要设置ulimit。
```

### 四、 location 规则

语法规则： location [=|~|~*|^~] /uri/ {… }


| 符号|    含义|
| :--------| :--------| 
| =| = 开头表示精确匹配（完全相同也算精准匹配） |
| ^~|（一般匹配）^~开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。nginx不对url做编码，因此请求为/static/20%/aa，可以被规则^~ /static/ /aa匹配到（注意是空格），匹配完成不再进行正则匹配|
| ~| ~ 开头表示区分大小写的正则匹配 |
|~*| ~* 开头表示不区分大小写的正则匹配 |
| !~和!~*| !~和!~*分别为区分大小写不匹配及不区分大小写不匹配的正则 |
| /| 用户所使用的代理（一般为浏览器） |
| $http_x_forwarded_for| 可以记录客户端IP，通过代理服务器来记录客户端的ip地址|
| $http_referer| 可以记录用户是从哪个链接访问过来的|

优先级

普通匹配location：
无前缀：达到完整匹配，也是=
"="
"^~" 

普通匹配满足最长匹配原则

正则匹配location：
"~" 表示区分大小写；
"~*"表示不区分大小写

![](https://i.imgur.com/tiKDPkw.png)

### 五、配置文件分离

`nginx`中配置多个`server(虚拟主机)`，为了方便维护，会将文件单独提取出来，通过导入的形式使其生效。


创建配置文件存放目录

```
mkdir /etc/nginx
mkdir /etc/nginx/conf.d
cd /etc/nginx/conf.d/

# 创建一个最基础的文件（常规的nginx服务）
vi www.conf
```

复制以下内容并保存

```
server {
listen       8080;	# 监听 8080 端口
server_name  39.105.110.40;	# 监听此 ip 的访问（一般为入口ip）

location /a {						# 如果请求的后缀是 /a 即使不加 = 也是精准匹配
        rewrite ^/  /a.html break;	#  /a 请求返回  html/static/路径下的 a 页面
        root   html/static/;		# 可能有用也可能没用，因为会被其他的覆盖
	}

location /b/a {		
        rewrite ^/  /b.html break;	# /b/a 或 /b/a/1 请求返回 b 页面
        root   html/static/;  #
	}

location /b/d/a {
        rewrite ^/  /d.html break;	# /b/d/a 或 /b/d/a/1 请求返回 c 页面（这里有点不符合常理，原因是精准匹配后依然后走下面的正则匹配，所以给我们看起来像是正则匹配的优先级高一样）
        root   html/static/;
	}

location ^~/b/c/a {		# 匹配 uri 路径（常用）
        rewrite ^/  /d.html break;	# /b/c/a 或 /b/c/a/1 请求返回 d 页面
        root   html/static/;
	}

location ~ /b/d {
        rewrite ^/  /c.html break;	# /b/d 或 /b/d/1 或 /b/d/a 请求返回 c 页面
        root   html/static/;
	}

location ~ /b/d/a {
        rewrite ^/  /a.html break;
        root   html/static/;
	}
}

```

修改 `ngxin.conf` 增加导入配置，当然需要删除原来相同的配置

```
include /etc/nginx/conf.d/*.conf;
```

重启以后，测试。

其中 `server_name` 可以是域名。可以通过修改 `windows hosts文件解析成域名，格式如此: 39.105.110.40 pleuvoir.cn`，注意不能加端口。


#### 反向代理
```
location /tweets {
	proxy_pass https://www.oschina.net/;
}
```

注意地址后面有没有 `/`，如果带了斜杠，表示标签是关闭的，那么 location 的 `/tweets` 不会自动带在需要访问的原始地址后面，即：如果我们目标访问地址是`https://www.oschina.net/tweets`，那么如果带了斜杠，请求地址应该为`pleuvoir.cn:8080/tweets/tweets`，此时`tweets`是不会传递到`tomcat`的后台，也就是说`https://www.oschina.net/ = pleuvoir.cn:8080/tweets`。如果没有，那么访问地址是`http://pleuvoir.cn:8080/tweets`。这里为了演示，location 的匹配直接用了 tweets，实际情况不是一样的。通过其他的规则去匹配然后再决定要不要加上这个`/`。

#### 负载均衡

1、轮询（默认）

```
upstream nginx {
server 172.17.0.4:8081;
server 172.17.0.5:8081;
}
```
每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。

2、weight

指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。down 暂时不参与负载。例如：
```
upstream nginx {
server 172.17.0.4:8081 weight=2;
server 172.17.0.5:8081 weight=1;
}
```

3、ip_hash

每个请求按访问ip的hash结果分配，这样同一客户端的请求总是发往同一个后端服务器，可以解决session的问题。例如：
```
upstream nginx {
ip_hash;
server 172.17.0.4:8081;
server 172.17.0.5:8081;
}
```


演示一套负载均衡的配置，需求如下:

172.17.0.2作为代理nginx
172.17.0.3作为静态服务器，读html文件
172.17.0.4为后台服务器1，提供web服务
172.17.0.5为后台服务器2，提供web服务

```
# 1
upstream nginx {
server 172.17.0.4:8081 weight=2;
server 172.17.0.5:8081 weight=1;
}

server {
        listen       8080;
        server_name  172.17.0.2;    

        location /nginx {
                proxy_pass http://nginx; # 对应 1 处的别名
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }


html代理：
location /static {
	proxy_pass http://172.17.0.3/; #这里是转发到另外一台服务器，由它来提供静态文件
}
```