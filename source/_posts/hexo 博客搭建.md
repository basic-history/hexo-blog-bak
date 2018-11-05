
---
title: 搭建 hexo 博客
date: 2018-09-24 10:19:00
author: pleuvoir
tags:
  - 博客
categories:
  - 备忘
---


简单记录 github-page 搭建博客的步骤


### 安装博客框架

```
## 安装 hexo
$ npm install -g hexo-cli

## 建站
$ hexo init <folder>
$ cd <folder>
$ npm install
```

### 修改配置

关于 hexo 的大部分配置均可修改 ```_config.yml``` 完成

### 下载主题

[hexo-theme-matery](https://github.com/blinkfox/hexo-theme-matery "hexo-theme-matery")

按图索骥即可

### 部署

安装 hexo-deployer-git
```
$ npm install hexo-deployer-git --save
```

修改配置

```
deploy:
  type: git
  repo: <repository url>
  branch: [branch]
  message: [message]
```

```
### 部署到远端
$ hexo deploy
```