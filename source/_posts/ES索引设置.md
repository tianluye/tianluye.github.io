---
title: ES索引设置
date: 2021-07-18 16:43:26
tags: [搜索, ElasticSerach]
# 文章分类，若多个标签，则用数组表示 #
categories: ElasticSerach
# 文章出处名称 #
from: 
# 文章出处链接 #
url: https://tianluye.github.io/2021/07/18/ES索引设置/
# 文章作者名称 #
author: tianluye
# 文章作者签名 #
about: 每天进步一点点，加油！
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: true
# 文章摘要 #
description: ES索引设置.
top: 1
---

可以按索引设置索引级别，设置可能是：

- 静态：它们只能在索引创建时或在关闭的索引上设置
- 动态：可以使用 `update-index-settings` API在活动索引上更改它们

> 更改关闭的索引上的静态或动态索引设置可能导致不正确的设置，在不删除和重新创建索引的情况下，不可能纠正这些设置。

### 静态索引设置

下面是与任何特定索引模块无关的所有静态索引设置的列表：

##### index.number_of_shards

- 索引应该具有的分片的数量，默认为 5。此设置只能在创建索引时设置，它不能在关闭的索引上更改，注意：每个索引的分片数量限制为 1024个，这种限制是一种安全限制，以防止意外地创建可能由于资源分配而破坏集群稳定的索引。可以在集群的每个节点上通过指定系统变量 `export ES_JAVA_OPTS="-Des.index.max_number_of_shards=128"` 来修改该限制。

##### index.shard.check_on_startup

- 分片在打开之前是否应该检查是否损坏，当检测到损坏时，它将阻止碎片被打开，接受：
   1. false：（默认）打开碎片时不要检查是否有损坏。
   2. checksum：检查物理损坏
   3. true：检查物理和逻辑损坏，就CPU和内存使用而言，这要昂贵得多

##### index.codec

- 默认值使用 `LZ4` 压缩压缩存储的数据，但是可以将这个值设置为 `best_compression`，它使用 `DEFLATE` 获得更高的压缩比，代价是降低存储字段的性能。如果你正在更新压缩类型，则新的压缩类型将在段合并后应用，段合并可以使用强制合并来强制执行。

##### index.routing_partition_size

- 自定义路由值可以转到的碎片数，默认值为1，只能在创建索引时设置，这个值必须小于 `index.number_of_shards`，除非 `index.number_of_shards` 值也是1。

##### index.load_fixed_bitset_filters_eagerly

- 指示是否为嵌套查询预加载缓存过滤器，可能的值为true（默认值）和false。

### 动态索引设置

以下是与任何特定索引模块无关的所有动态索引设置列表：

##### index.number_of_replicas

- 每个主碎片拥有的副本数量，默认为1。

##### index.auto_expand_replicas

- 根据集群中数据节点的数量自动扩展副本的数量，设置为以破折号分隔的下界和上界（例如0-5），或将all用于上界（例如0-all），默认为false（即禁用）。注意，自动扩展的副本数量没有考虑任何其他分配规则，比如碎片分配感知、过滤或每个节点的总碎片，如果适用的规则阻止分配所有副本，则可能导致集群健康状态变为黄色。

##### index.search.idle.after

- 一个分片在被认为是搜索空闲之前不能接收搜索或获取请求的时间（默认是30s）。

##### index.refresh_interval

- 多久执行一次刷新操作，使最近对索引的更改对搜索可见，默认为1s，可以设置为-1以禁用刷新。如果没有显式设置此设置，至少在 `index.search.idle.after` 秒内没有看到搜索流量的碎片在收到搜索请求之前不会收到后台刷新，当刷新挂起时，搜索到空闲碎片，将等待下一次后台刷新（在1秒内）。此行为的目的是在不执行搜索的默认情况下自动优化批量索引，为了避免这种行为，应该将显式值1s设置为刷新间隔。

##### index.max_result_window

- 此索引的搜索 `from + size` 的最大值，默认为 10000，搜索请求占用的堆内存和时间与 `from + size` 成正比，这限制了内存。

##### index.max_inner_result_window

- from + size的最大值，用于将内部命中定义和顶部命中聚合到此索引，默认为100，内部命中和顶部命中聚合占用堆内存，并且时间与from + size成正比，这限制了内存。

##### index.max_rescore_window

- 在此索引的搜索中，rescore请求的window_size的最大值，默认为index.max_result_window，其默认值为10000，搜索请求占用堆内存，时间与max(window_size, from + size)成正比，这限制了内存。

##### index.max_docvalue_fields_search

- 查询中允许的docvalue_fields的最大数量，默认为100，Doc-value字段代价高昂，因为它们可能导致每个字段每个文档的搜索。

##### index.max_script_fields

- 查询中允许的最大script_fields数量，默认为32。

##### index.max_ngram_diff

- NGramTokenizer和NGramTokenFilter的min_gram和max_gram之间允许的最大差异，默认为1。

##### index.max_shingle_diff

- 对于ShingleTokenFilter，max_shingle_size和min_shingle_size之间允许的最大差异，默认为3。

##### index.blocks.read_only

- 设置为true使索引和索引元数据只读，设置为false允许写入和元数据更改。

##### index.blocks.read_only_allow_delete

- 与index.blocks.read_only相同，但是允许删除索引来释放资源。

##### index.blocks.read

- 设置为true以禁用对索引的读取操作。

##### index.blocks.write

- 设置为true可禁用对索引的数据写操作，与read_only不同，此设置不影响元数据，例如，可以用write块关闭索引，但不能用read_only块关闭索引。

##### index.blocks.metadata

- 设置为true可禁用索引元数据的读写。

##### index.max_refresh_listeners

- 索引的每个碎片上可用的刷新侦听器的最大数量，这些侦听器用于实现refresh=wait_for。

##### index.analyze.max_token_count

- 使用_analyze API可以生成的最大令牌数量，默认为10000。

##### index.highlight.max_analyzed_offset

- 为高亮显示请求分析的最大字符数，此设置仅适用于在索引没有偏移量或术语向量的文本上请求高亮显示时，默认为1000000。

##### index.max_terms_count

- 可以在terms查询中使用的术语的最大数量，默认为65536。

##### index.max_regex_length

- 可用于Regexp查询的正则表达式的最大长度，默认为1000。

##### index.routing.allocation.enable

- 控制此索引的碎片分配，它可以被设置为：
   1. all（默认值）— 允许为所有碎片分配碎片。
   2. primaries — 只允许为主碎片分配碎片。
   3. new_primary — 只允许为新创建的主碎片分配碎片。
   4. none — 不允许碎片分配。

##### index.routing.rebalance.enable

- 启用此索引的碎片重新平衡，它可以被设置为：
   1. all（默认值）—允许对所有碎片进行碎片再平衡。
   2. primaries — 只允许对主碎片进行碎片再平衡。
   3. replicas — 只允许对复制碎片进行碎片再平衡。
   4. none — 不允许再平衡碎片。

##### index.gc_deletes

- 已删除文档的版本号仍可用于进一步版本化操作的时间长度，默认为60s。

##### index.default_pipeline

- 此索引的默认摄取节点管道，如果设置了默认管道且管道不存在，则索引请求将失败，可以使用pipeline参数重写默认值，特殊管道名称_none表示不应该运行摄取管道。

### 其他索引模块中的设置

其他索引设置在索引模块中可用：

##### 分析

- 用于定义分析程序、令牌器、令牌过滤器和字符过滤器的设置。

##### 索引碎片分配

- 控制将碎片分配到节点的位置、时间和方式。

##### 映射

- 为索引启用或禁用动态映射。

##### 合并

- 控制后台合并进程如何合并碎片。

##### 相似性

- 配置自定义相似性设置，以自定义搜索结果的评分方式。

##### 慢日志

- 控制记录查询和获取请求的速度。

##### 存储

- 配置用于访问碎片数据的文件系统类型。

##### 事务日志

- 控制事务日志和后台刷新操作。

---

参考文章：https://segmentfault.com/a/1190000019919494/