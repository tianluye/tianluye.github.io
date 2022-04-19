---
title: github访问缓慢问题解决
date: 2022-04-19 22:03:01
tags: [github, 系统]
categories: 系统
# 文章出处名称 #
from: 
# 文章出处链接 #
url: https://tianluye.github.io/2021/07/17/LinkedList/
# 文章作者名称 #
author: tianluye
# 文章作者签名 #
about: 道无涯，学无止境
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: true
# 文章摘要 #
description: 
top: 10
---

自己访问博客地址： `https://tianluye.github.io/ `

发下特别慢，于是乎想到有没有类似于镜像配置，可以让我快速访问。不懂就问百度，哈哈，还真的有，具体操作如下：

#### 1、修改 hosts文件

```
## Mac
$ sudo vim /etc/hosts

## Windows
C:\Windows\System32\drivers\etc\hosts
```

#### 2、将以下内容复制到 hosts文件中

```
# github
151.101.185.194 github.global.ssl.fastly.net
192.30.253.112 github.com 
151.101.112.133 assets-cdn.github.com 
151.101.184.133 assets-cdn.github.com 
185.199.108.153 documentcloud.github.com 
192.30.253.118 gist.github.com
185.199.108.153 help.github.com 
192.30.253.120 nodeload.github.com 
151.101.112.133 raw.github.com 
23.21.63.56 status.github.com 
192.30.253.1668 training.github.com 
192.30.253.112 www.github.com 
151.101.13.194 github.global.ssl.fastly.net 
151.101.12.133 avatars0.githubusercontent.com 
151.101.112.133 avatars1.githubusercontent.com
140.82.114.4 github.com
199.232.5.194 github.global.ssl.fastly.net
```

#### 3、刷新 DNS

```
## Mac
dscacheutil -flushcache

## Windows
ipconfig/flushdns
```

---

#### Tips

Github的 IP地址是不断变化的，如果发现网站打不开了，可以获取新的 IP地址修改 hosts里面的内容，方式如下：

在网站 https://www.ipaddress.com/ 输入你要解析的域名

例如：github.com的 IP获取方式，在输入框输入以下内容：git

- github.com
- github.githubassets.com
- github.global.ssl.fastly.net

![image](https://cdn.jsdelivr.net/gh/tianluye/tianluye.github.io/image/00168.jpg)





