---
title: Elasticsearch初识
date: 2021-07-18 11:11:08
tags: [搜索, ElasticSerach]
# 文章分类，若多个标签，则用数组表示 #
categories: ElasticSerach
# 文章出处名称 #
from: 
# 文章出处链接 #
url: https://tianluye.github.io/2021/07/18/Elasticsearch初识/
# 文章作者名称 #
author: tianluye
# 文章作者签名 #
about: 只有自身的强大，才足以让自己有更多的选择!
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: true
# 文章摘要 #
description: Elasticsearch初识.
# 文章置顶 #
sticky: 100
top: 100
---

`Elasticsearch`是一个基于 `Apache Lucene(TM)`的开源搜索引擎，是一个`开源的高扩展的分布式全文搜索引擎`。

`Elasticsearch`使用 Java开发并使用 Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的 `RESTful API来隐藏Lucene的复杂性`，从而让全文搜索变得简单。

这里说到的全文搜索引擎指的是目前广泛应用的主流搜索引擎。它的工作原理是计算机索引程序通过扫描文章中的每一个词，对每一个词建立一个索引，指明该词在文章中出现的次数和位置，当用户查询时，检索程序就根据事先建立的索引进行查找，并将查找的结果反馈给用户的检索方式。这个过程类似于通过字典中的检索字表查字的过程。 

Elasticsearch不仅仅是 Lucene和全文搜索，我们还能这样去描述它：

- 分布式的实时文件存储，每个字段都被索引并可被搜索
- 分布式的实时分析搜索引擎
- 可以扩展到上百台服务器，处理PB级结构化或非结构化数据

### Elasticsearch Or Solr 

搜索效率的比较：

![image](https://cdn.jsdelivr.net/gh/tianluye/tianluye.github.io/image/00117.png)
![image](https://cdn.jsdelivr.net/gh/tianluye/tianluye.github.io/image/00119.png)

ElasticSearch vs Solr 总结

　　（1）es基本是开箱即用，非常简单。Solr安装略微复杂一丢丢，可关注（solr6.6教程-基础环境搭建(一)）

　　（2）Solr 利用 Zookeeper 进行分布式管理，而 Elasticsearch 自身带有分布式协调管理功能。

　　（3）Solr 支持更多格式的数据，比如JSON、XML、CSV，而 Elasticsearch 仅支持json文件格式。

　　（4）Solr 官方提供的功能更多，而 Elasticsearch 本身更注重于核心功能，高级功能多有第三方插件提供，例如图形化界面需要kibana友好支撑

　　（5）Solr 查询快，但更新索引时慢（即插入删除慢），用于电商等查询多的应用；

　　　　  ES建立索引快（即查询慢），即实时性查询快，用于facebook新浪等搜索。

　　   　   Solr 是传统搜索应用的有力解决方案，但 Elasticsearch 更适用于新兴的实时搜索应用。

　　（6）Solr比较成熟，有一个更大，更成熟的用户、开发和贡献者社区，而 Elasticsearch相对开发维护者较少，更新太快，学习使用成本较高。

目前市面上流行的搜索引擎软件，主流的就两款：Elasticsearch 和 Solr。这两款都是基于 Lucene 搭建的，可以独立部署启动的搜索引擎服务软件。由于内核相同，所以两者除了服务器安装、部署、管理、集群以外，对于数据的操作 修改、添加、保存、查询等等都十分类似。 

---

### 数据格式 

Elasticsearch 是面向文档型数据库，一条数据在这里就是一个文档。为了方便大家理解，我们将 Elasticsearch 里存储文档数据和关系型数据库 MySQL 存储数据的概念进行一个类比：

| ElasticSearch | Index(索引)      | Type(类型) | Documents(文档) | Fields(字段) |
| ------------- | ---------------- | ---------- | --------------- | ------------ |
| Mysql         | Database(数据库) | Table(表)  | Row(行)         | Column(列)   |

ES 里的 Index 可以看做一个库，而 Types 相当于表，Documents 则相当于表的行。 
这里 Types 的概念已经被逐渐弱化，Elasticsearch 6.X 中，一个 index 下已经只能包含一个type，Elasticsearch 7.X 中, Type 的概念已经被删除了。 

---

### 基于 docker安装 Elasticsearch：

首先去 DockerHub官网上找到 ElasticSearch的镜像：

`https://hub.docker.com/_/elasticsearch?tab=description&page=1&ordering=last_updated`

可以看到官网上有介绍如何使用该镜像：

```
How to use this image

1、Create user defined network (useful for connecting to other services attached to the same network (e.g. Kibana)):
$ docker network create somenetwork

2、Run Elasticsearch:
$ docker run -d --name elasticsearch --net somenetwork -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:tag
```

进入到 es的 docker 容器后，查看其文件目录：

```
[root@localhost ~]# docker exec -it 495 bash
[root@4956d21a83bf elasticsearch]# ls
LICENSE.txt  README.asciidoc  config  jdk  logs     plugins
NOTICE.txt   bin              data    lib  modules
```


| 目录    | 含义           |
| ------- | -------------- |
| bin     | 可执行脚本目录 |
| config  | 配置目录       |
| jdk     | 内置 JDK目录   |
| lib     | 类库           |
| logs    | 日志目录       |
| modules | 模块目录       |
| plugins | 插件目录       |

注意：**9300 端口为 Elasticsearch 集群间组件的通信端口，9200 端口为浏览器访问的 http**

上面启动的 Es并没有挂载任何的数据卷，到时候修改 ES配置的话，都需要进入容器中才能进行修改。这样比较麻烦，下面来重新创建挂载了数据卷的容器。

---

删除上面创建的容器：`docker rm -f $(docker ps -aq)`

开始配置 elasticsearch，并使用 kibana连接到 ES，再安装 ES HEAD插件：

```
# 新建 es数据，配置目录和插件。并且给这三个文件夹赋 777 权限
[root@localhost ~]# mkdir -p /mydata/elasticsearch/{config,data,plugins}
[root@localhost ~]# chomd 777 /mydata/elasticsearch
# 新建并写入配置文件
[root@localhost ~]# echo "http.host: 0.0.0.0" >> /mydata/elasticsearch/config/elasticsearch.yml

# 创建/启动容器
[root@localhost ~]# docker run -d 
                    # 容器的名称
                    --name elasticsearch 
                    # 让 docker有 root权限启动容器，否则会报无权限访问 usr/share/elasticsearch/config/elasticsearch.yml
                    --privileged=true
                    # 暴露端口
                    -p 9201:9200 -p 9300:9300 
                    # 单集群模式
                    -e "discovery.type=single-node" 
                    # 指定 es占用的内容，一般不指定，使用默认
                    -e ES_JAVA_OPTS="-Xms256m -Xmx2048m"
                    # 挂载数据卷：配置，数据，插件。
                    -v /mydata/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml 
                    -v /mydata/elasticsearch/data:/usr/share/elasticsearch/data 
                    -v /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins
                    # 指定 es的网络
                    --net es-network 
                    elasticsearch:7.12.0

# 安装完毕后，测试访问
[root@localhost ~]# curl http://192.168.137.223:9200/?pretty
{
  "name": "4956d21a83bf",
  "cluster_name": "elasticsearch",
  "cluster_uuid": "ogiFA2pnR7KjIazXGk-dzg",
  "version": {
    "number": "7.12.0",
    "build_flavor": "default",
    "build_type": "docker",
    "build_hash": "78722783c38caa25a70982b5b042074cde5d3b3a",
    "build_date": "2021-03-18T06:17:15.410153305Z",
    "build_snapshot": false,
    "lucene_version": "8.8.0",
    "minimum_wire_compatibility_version": "6.8.0",
    "minimum_index_compatibility_version": "6.0.0-beta1"
  },
  "tagline": "You Know, for Search"
}

# 配置 kibana连接 ES。注意：两者的版本号要一致！
[root@localhost ~]# docker run -d 
                    # kibana的容器名称
                    --name kibana 
                    # 与 ES相同的网络
                    --net es-network 
                    # 连接到 ES
                    -e ELASTICSEARCH_HOSTS=http://192.168.137.223:9200
                    # 暴露的端口
                    -p 5601:5601 
                    # 要与 ES的版本保持一致
                    kibana:7.12.0
```

kibana启动完毕后，访问 `http://192.168.137.223:5601`，可以看到：

![image](https://cdn.jsdelivr.net/gh/tianluye/tianluye.github.io/image/00154.png)

### 安装 HEAD插件

DockerHub地址：https://hub.docker.com/r/mobz/elasticsearch-head

```
# 下载镜像
[root@localhost ~]# docker pull mobz/elasticsearch-head:5

# 启动 head插件前，需要设置 es可跨域访问
# 修改 es的配置文件：elasticsearch.yml
http.cors.enabled: true
http.cors.allow-origin: "*"

# 启动容器
[root@localhost ~]# docker run -d --name my-es_admin -p 9100:9100 mobz/elasticsearch-head:5
```

启动完成后，访问 `http:虚拟机地址IP:9100/`，将连接的地址修改为 `http:虚拟机IP:9200/` 进行访问 ES：

![image](https://cdn.jsdelivr.net/gh/tianluye/tianluye.github.io/image/00155.png)

但是一切准备就绪后，ES-HEAD打开界面连接 es时发现数据无法展示。于是网上查了下原因，说是elasticsearch 6增加了请求头严格校验的原因，并且返回结果是：

```
{
    "error" : "Content-Type header [application/x-www-form-urlencoded] is not supported",
    "status" : 406
}
```

所以我们需要修改一下elasticsearch-head 5的配置文件。

#### 1、因为docker容器里面无法使用vi/vim，所以需要先将文件拷贝出来。

命令：`docker cp es_head:/usr/src/app/_site/vendor.js ./`

说明：将容器里面 `/usr/src/app/_site/vendor.js` 文件拷贝到宿主机的当前目录下，其中 es_head为容器名，也可以写容器id。

#### 2、编辑文件

vi vendor.js

共有两处

1）6886行
`contentType: "application/x-www-form-urlencoded` 改成 `contentType: "application/json;charset=UTF-8"`

2）7573行
`var inspectData = s.contentType === "application/x-www-form-urlencoded" &&` 改成 `var inspectData = s.contentType === "application/json;charset=UTF-8" &&`

补充说明：

```
vi中显示行号的命令为
:set nu

vi中跳转到指定行的命令为
:行号
```

#### 3、将改完后的文件拷贝回容器

    docker cp vendor.js es_head:/usr/src/app/_site

无需重启，刷新页面即可。 