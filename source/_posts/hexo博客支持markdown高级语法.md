---
title: hexo博客支持markdown高级语法
date: 2021-08-21 20:23:50
tags: [hexo]
# 文章分类，若多个标签，则用数组表示 #
categories: hexo
# 文章出处名称 #
from: 
# 文章出处链接 #
url: https://tianluye.github.io/2021/08/21/hexo博客支持markdown高级语法/
# 文章作者名称 #
author: tianluye
# 文章作者签名 #
about: 道无涯，学无止境
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: true
# 文章摘要 #
description: hexo博客支持markdown高级语法.
---

默认情况下，hexo搭建的博客主题是不支持的 markdown的高级语法，即流程图的绘制。更多的流程图绘制可以参考下面的文章：

https://blog.csdn.net/zywvvd/article/details/110150639

https://www.runoob.com/markdown/md-advance.html

### 安装插件

通过安装插件 ` hexo-filter-mermaid-diagrams ` 来实现：

```
npm install hexo-filter-mermaid-diagrams
```

### 修改配置文件

 在hexo的 `_config.yml`文件（根目录的并非主题的）中，添加以下内容： 

```ymal
# mermaid chart
mermaid: ## mermaid url https://github.com/knsv/mermaid
  enable: true  # default true
  version: "7.1.2" # default v7.1.2
  options:
```

Tip：不知道是不是我引入的主题 `LiveForCode` 的原因，一直读取不到 `mermaid` 配置属性。

### 布局文件修改

到自己的主题目录下：

> E:\blog\themes\LiveForCode\layout\_partial

不同的主题，布局文件类型也是不一样的（具体的可以参考：[hexo-filter-mermaid-diagrams](https://github.com/webappdevelp/hexo-filter-mermaid-diagrams)），我的博客中是用 `ejs` 。所以使用下面的语句：

```js
<% if (theme.mermaid.enable) { %>
  <script src='https://unpkg.com/mermaid@<%= theme.mermaid.version %>/dist/mermaid.min.js'></script>
  <script>
    if (window.mermaid) {
      mermaid.initialize({theme: 'forest'});
    }
  </script>
<% } %>
```

因为读取不到 `mermaid` 配置属性，所以最终的布局文件被我改为：

```js
<div id="footer"></div>

<script src='https://unpkg.com/mermaid@7.1.2/dist/mermaid.min.js'></script>
<script>
    if (window.mermaid) {
      mermaid.initialize({theme: 'forest'});
    }
</script>
```

### 重新发布

```
hexo d -g
```

