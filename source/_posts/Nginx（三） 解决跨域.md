
---
title: Nginx（三） 解决跨域
date: 2016-09-30 12:00:00
author: pleuvoir
img: /images/code.jpg
tags:
  - Linux
  - Nginx
  - 跨域
categories:
  - 技术
---

### 浏览器跨域问题

跨域是指从一个域名的网页去请求另一个域名的资源。

添加header头： Access-Control-Allow-Origin，表明允许网站执行

简单请求：
浏览器在跨源AJAX请求的头信息之中，自动在添加一个Origin字段（本次请求来自哪个源 ）。服务器根据这个值，在许可范围内，则在头信息包含 Access-Control-Allow-Origin 。

复杂请求：
会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求OPTIONS 


#### 1. 通过反向代理

让前端 ajax 请求自己项目中的地址，通过 proxy_pass 转发到真实的服务器

#### 2. Cors 解决方案

只要响应中有header头： Access-Control-Allow-Origin，表明允许网站执行

![](https://i.imgur.com/vmMzBpk.png)

案例:

a、当 chrome 发现 ajax 请求的网址，与当前主域名不一致（跨域）时，会在请求 header 中追加值页面主域名值，如：origin = http://static.pleuvoir.cn

b、nginx 在接收到 ajax 请求时，会查看 origin 值，即请求我的网址是谁？此处使用正则来校验，即：只要是 pleuvoir.cn 下的网址，都允许访问我。返回信息时，nginx 追加 header 值：access-control-allow-origin = static.pleuvoir.cn（回答浏览器，这个域名网址可以访问我）

nginx.conf 配置如下，这段配置直接贴也基本没问题，只需要修改下一级域名:

```
listen  80;
server_name  pleuvoir.cn;
# $http_origin 内置变量 = a 中 origin 的值
if ( $http_origin ~ http://(.*).pleuvoir.cn){
     set $allow_url $http_origin;
}
#是否允许请求带有验证信息
add_header Access-Control-Allow-Credentials true;
#允许跨域访问的域名,可以是一个域的列表，也可以是通配符*
add_header Access-Control-Allow-Origin  $allow_url;  #（重要）这里就是 nginx 的回答了，允许域名下网页访问
#允许脚本访问的返回头
add_header Access-Control-Allow-Headers 'x-requested-with,content-type,Cache-Control,Pragma,Date,x-timestamp';
#允许使用的请求方法，以逗号隔开
add_header Access-Control-Allow-Methods 'POST,GET,OPTIONS,PUT,DELETE';
#允许自定义的头部，以逗号隔开,大小写不敏感
add_header Access-Control-Expose-Headers 'WWW-Authenticate,Server-Authorization';
#P3P支持跨域cookie操作
add_header P3P 'policyref="/w3c/p3p.xml", CP="NOI DSP PSAa OUR BUS IND ONL UNI COM NAV INT LOC"';
add_header test  1;

if ($request_method = 'OPTIONS') {
     return 204;
}
```

c、chrome 收到 ajax 返回值后，查看返回的 header中 access-control-allow-origin 的值，发现其中的值是 static.pleuvoir.cn,正是当前的页面主域名。这是允许访问，于是执行ajax返回值内容。（ps：若此处access-control-allow-origin不存在，或者值不是static域名，chrome就拒绝执行返回值）

