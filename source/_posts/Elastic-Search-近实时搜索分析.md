---
title: Elastic Search 近实时搜索分析
date: 2021-08-12 18:00:39
tags: [搜索, ElasticSerach]
# 文章分类，若多个标签，则用数组表示 #
categories: ElasticSerach
# 文章出处名称 #
from: 
# 文章出处链接 #
url: https://elasticstack.blog.csdn.net/article/details/103641544?utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.base&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.base
# 文章作者名称 #
author: tianluye
# 文章作者签名 #
about: 每天进步一点点，加油！
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: true
# 文章摘要 #
description: Elastic Search 近实时搜索分析.
top: 1
---

### Elastic Search 近实时搜索分析

#### 1、Refresh 及 Flush

两者都用于使文档在索引操作后立即可供搜索。 在 Elasticsearch 中添加新文档时，我们可以对索引调用 `_refresh` 或 `_flush` 操作，以使新文档可用于搜索。 要了解这些操作的工作方式，必须熟悉 Lucene中的 Segments，Reopen( 重新打开 ) 和 Commits( 提交 )。Apache Lucene 是 Elasticsearch 中的基础查询引擎。

#### 2、Lucene 中的 Segments

在 Elasticsearch 中，<font color=#0099ff>最基本的数据存储单位是 shard</font>。 但是，通过 Lucene 镜头看，情况会有所不同。 在这里，每个 Elasticsearch 分片都是一个 Lucene 索引 (index)，每个 Lucene 索引都包含几个 Lucene segments。 一个 Segment 包含映射到文档里的所有术语（terms) 一个反向索引 (inverted index)。

下图显示了段的概念及其如何应用于 Elasticsearch 索引及其分片：

 ![img](https://img-blog.csdnimg.cn/20191221101112584.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9lbGFzdGljc3RhY2suYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70) 



这种分 Segment 的概念是，每当创建新文档时，它们就会被写入新的 Segment 中。 每当创建新文档时，它们都属于一个新的Segment，并且无需修改前一个 Segment。 如果必须删除文档，则在其原始 Segment 中将其标记为已删除。 这意味着它永远不会从 Segement 中物理删除。

与更新相同：文档的先前版本在上一个 Segment 中被标记为已删除，更新后的版本保留在当前 Segment 中的同一文档ID下。

#### 3、Lucene 中的 Reopen

 当调用 Lucene Reopen 时，将使累积的数据可用于搜索。 尽管可以搜索最新数据，但这不能保证数据的持久性或未将其写入磁盘。 我们可以调用 n 次重新打开功能，并使最新数据可搜索，但不能确定磁盘上是否存在数据。 

#### 4、Lucene 中的 Commits

Lucene 提交使数据安全。 对于每次提交，来自不同段的数据将合并并推送到磁盘，从而使数据持久化。 尽管提交是持久保存数据的理想方法，但问题是每个提交操作都占用大量资源。 每个提交操作都有其自己的内部 I/O 操作以及与其相关的读/写周期。 这就是为什么我们希望在基于 Lucene 的系统中一次又一次地重新使用重新打开功能以使新数据可搜索的确切原因。

#### 5、Elasticsearch 中的 Translog

Elasticsearch 采用另一种方法来解决持久性问题。 它在每个分片中引入一个事务日志（transaction log）。 已建立索引的新文档将传递到此事务日志和内存缓冲区中。 下图显示了此过程：

 ![img](https://img-blog.csdnimg.cn/20191221101920699.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9lbGFzdGljc3RhY2suYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70) 

#### 6、Elasticsearch 中的 refresh

当我们把一条数据写入到 Elasticsearch 中后，它并不能马上被用于搜索。新增的索引必须写入到 Segment 后才能被搜索到，因此我们把数据写入到内存缓冲区之后并不能被搜索到。新增了一条记录时，Elasticsearch 会把数据写到 translog 和 in-memory buffer (内存缓存区)中，如下图所示:
 ![img](https://img-blog.csdnimg.cn/20191221103533158.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9lbGFzdGljc3RhY2suYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70) 

在此期间，该文档不能被搜索，但是我们还是可以<font color=#DC143C>通过ID使用GET来获得该文档</font>。如果希望该文档能立刻被搜索，需要手动调用 `refresh` 操作。在 Elasticsearch 中，**默认情况下 _refresh 操作设置为每秒执行一次**。 在此操作期间，内存中缓冲区的内容将复制到内存中新创建的 Segment 中，如下图所示。 结果，新数据可用于搜索。

 ![img](https://img-blog.csdnimg.cn/20191221102058508.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9lbGFzdGljc3RhY2suYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70) 

这个 refresh 的时间间隔可以由 index 设置中 `index.refresh_interval` 来定义。只有在 buffer 的内容写入到 Segement 后，这个被写入的文档才变为可以搜索的文档。通常 buffer 里的内容被写入到 Segment 里去，有三个条件：

- 由索引中的设置所指定的 refresh_interval 启动的周期性的 refresh。在默认的情况下为 <font color= #FF0000> 1s </font>。这使对索引的最近更改可见以进行搜索。 默认为1s。 **可以设置为 -1 以禁用刷新**。 在 Elasticsearch  7.0 发布之后，如果未明确设置此设置，则至少在 `index.search.idle.after` 秒之后仍未看到搜索流量的分片在收到搜索请求之前将不会接收后台刷新。 命中空闲分片的搜索将等待下一次后台刷新（在1秒内）。 此行为旨在在不执行搜索时在默认情况下自动优化批量索引。 为了退出此行为，应将显式值 1s 设置为刷新间隔。
- 在导入文档时强制 refresh：`PUT twitter/_doc/1?refresh=true`
- 当 In Memory Buffer 满了，在默认的情况下为 node Heap 的 10%。

这个过程会产生一个叫 Lucene flush 的操作，也会生产一个 segment。执行完 refresh 后的结果如下：

 ![img](https://img-blog.csdnimg.cn/20191221102525761.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9lbGFzdGljc3RhY2suYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70) 

我们可以看出来，在 In-meomory buffer 中，现在所有的东西都是空的，但是 Translog 里还是有东西的。

refresh 的开销比较大,我在自己环境上测试10W条记录的场景下refresh一次大概要14ms，因此在批量构建索引时可以把 refresh 间隔设置成-1来临时关闭 refresh, 等到索引都提交完成之后再打开 refresh, 可以通过如下接口修改这个参数:

```
curl -XPUT 'localhost:9200/test/_settings' -d '{
    "index" : {
        "refresh_interval" : "-1"
    }
}'
```

#### 7、Translog 及持久化存储

但是，translog 如何解决持久性问题？ 每个 Shard 中都存在一个 translog，这意味着它与物理磁盘内存有关。 它是同步且安全的，因此即使对于尚未提交的文档，您也可以获得持久性和持久性。 如果发生问题，可以还原事务日志。 同样，在每个设置的时间间隔内，或在成功完成请求（索引，批量，删除或更新）后，将事务日志提交到磁盘。

#### 8、Elasticsearch 中的 Flush

 Flush 实质上意味着将内存缓冲区中的所有文档都写入新的 Lucene Segment，如下面的图所示。 这些连同所有现有的内存段一起被提交到磁盘，该磁盘清除事务日志（参见下图）。 此提交本质上是 Lucene 提交（commit）。 

 ![img](https://img-blog.csdnimg.cn/20191221104032180.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9lbGFzdGljc3RhY2suYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70) 

 Flush 会定期触发，也可以在 Translog 达到特定大小时触发。 这些设置可以防止 Lucene 提交带来的不必要的费用。 

#### 9、结论

在本指南中，我们探索了两个紧密相关的 Elasticsearch 操作，_flush 和 _refresh 显示了它们之间的共性和差异。 我们还介绍了 Lucene 的基础架构组件-重新打开（reopen) 并提交 (commits) - 这有助于掌握 Elasticsearch中 _refresh 和 _flush 操作的要点。

简而言之，_refresh 用于使新文档可见以进行搜索。 而 _flush 用于将内存中的段保留在硬盘上。 _flush 不会影响 Elasticsearch 中文档的可见性，因为搜索是在内存段中进行的，而不是 _refresh 会影响其可见性。

---

进阶学习：https://blog.csdn.net/qq_27529917/article/details/109389215

