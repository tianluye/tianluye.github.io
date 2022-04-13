---
title: ES max_result_window
date: 2021-07-18 17:26:10
tags: [搜索, ElasticSerach]
# 文章分类，若多个标签，则用数组表示 #
categories: ElasticSerach
# 文章出处名称 #
from: 
# 文章出处链接 #
url: https://tianluye.github.io/2021/07/18/ES-max_result_window/
# 文章作者名称 #
author: tianluye
# 文章作者签名 #
about: 每天进步一点点，加油！
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: true
# 文章摘要 #
description: ES max_result_window.
top: 1
---

在使用 Elasticsearch进行 search查询时，出现了 `Result window is too large` 的问题。

报错如下：

```
QueryPhaseExecutionException[Result window is too large, from + size must be less than or equal to: [10000] but was [10200]. 
See the scroll api for a more efficient way to request large data sets. 
This limit can be set by changing the [index.max_result_window] index level parameter.]
```

从上面的报错信息，可以看到ES提示我结果窗口太大了，目前最大值为10000，而我却要求给我10200。并且在后面也提到了要求我修改 `index.max_result_window` 参数来增大结果窗口大小。

最后通过查阅了解到出现这个问题是由于 ElasticSearch的默认 **深度翻页** 机制的限制造成的。ES默认的分页机制一个不足的地方是，比如有5010条数据，当你仅想取第5000到5010条数据的时候，ES也会将前5000条数据加载到内存当中，所以ES为了避免用户的过大分页请求造成 ES服务所在机器内存溢出，默认对深度分页的条数进行了限制，默认的最大条数是10000条，这是正是问题描述中当获取第10000条数据的时候报 `Result window is too large` 异常的原因。

要解决这个问题，可以使用下面的方式来改变ES默认深度分页的 `index.max_result_window` 最大窗口值

```
curl -XPUT http://127.0.0.1:9200/my_index/_settings -d '{ "index" : { "max_result_window" : 500000}}'
```

---

**注意事项**

通过上述的方式解决了我们的问题，但也引入了另一个需要我们注意的问题，窗口值调大了后，虽然请求到分页的数据条数更多了，但它是用牺牲更多的服务器的内存、CPU资源来换取的。要考虑业务场景中过大的分页请求，是否会造成集群服务的OutOfMemory问题。

ES的官方文档中对深度分页也做了讨论，核心的观点如下：

根据文档的大小，分片的数量以及使用的硬件，分页10,000到50,000个结果（1,000到5,000页）应该是完全可行的。 但是，从价值观上来看，使用大量的CPU，内存和带宽，分类过程确实会变得非常重要。 为此，我们强烈建议不要进行深度分页。

自己的看法是，ES作为一个搜索引擎，更适合的场景是使用它进行搜索，而不是大规模的结果遍历。 大部分场景下，没有必要得到超过10000个结果项目， 例如，只返回前1000个结果。如果的确需要大量数据的遍历展示，考虑是否可以用其他更合适的存储。