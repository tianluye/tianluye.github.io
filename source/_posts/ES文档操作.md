---
title: ES文档操作
date: 2021-07-18 16:25:50
tags: [搜索, ElasticSerach]
# 文章分类，若多个标签，则用数组表示 #
categories: ElasticSerach
# 文章出处名称 #
from: 
# 文章出处链接 #
url: https://tianluye.github.io/2021/07/18/ES文档操作/
# 文章作者名称 #
author: tianluye
# 文章作者签名 #
about: 每天进步一点点，加油！
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: true
# 文章摘要 #
description: ES文档操作.
top: 1
---

### 创建文档 

这里的文档可以类比为关系型数据库中的表数据。但是需要注意的是，ES中已经弱化了【表概念】，万物皆索引！

kibana上向 ES发送 POST命令：

```
POST shopping/_doc
{     
  "title":"小米手机",     
  "category":"小米",     
  "images":"http://www.gulixueyuan.com/xm.jpg", 
  "price":3999.00 
}
```

返回结果为：

```
{
  "_index" : "shopping",
  "_type" : "_doc",
  #可以类比为 MySQL 中的主键，ES 随机生成 
  "_id" : "Es_9f3kBw9jrJTK9XUO2",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 5,
  "_primary_term" : 1
}
```

此处发送请求的方式必须为 POST，不能是 PUT，否则会发生错误。PUT方式要支持幂等，显然每次请求都会创建一个新的数据，且主键由 ES随机生成，所以这里不能使用 PUT命令。

一个索引下只能存在一个 Type（表）：

```
POST shopping/asd
{     
  "title":"小米手机",     
  "category":"小米",     
  "images":"http://www.gulixueyuan.com/xm.jpg", 
  "price":3999.00 
  
} 

{
  "error" : {
    "root_cause" : [
      {
        "type" : "illegal_argument_exception",
        "reason" : "Rejecting mapping update to [shopping] as the final mapping would have more than 1 type: [_doc, asd]"
      }
    ],
    "type" : "illegal_argument_exception",
    "reason" : "Rejecting mapping update to [shopping] as the final mapping would have more than 1 type: [_doc, asd]"
  },
  "status" : 400
}
```

如何指定生成文档的 _id 属性？

```
POST shopping/_doc/1
{     
  "title":"小米手机",     
  "category":"小米",     
  "images":"http://www.gulixueyuan.com/xm.jpg", 
  "price":3999.00 
}

{
  "_index" : "shopping",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 6,
  "_primary_term" : 3
}
```

假设我这边再次执行这个 POST语句会怎么样？结果是：若 _id 不存在，则是创建；若 _id 存在，则是修改。

```
{
  "_index" : "shopping",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 3,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 8,
  "_primary_term" : 3
}
```

实际上，若是指定了 _id ，那么使用 PUT 命令也是可以完成文档的新增和修改操作的。因为 _version 属性会不断的变化，也是能够满足 PUT 的幂等性！

---

### 更新文档


#### 局部更新

在请求路径后加 `_update` 标志是局部更新，否则全局更新。且需要注意有固定格式 `doc`：

```json
http://10.122.7.9:9200/test_index/doc/SS1808170004/_update
{
    "doc": {
        "lastUpdateTime": "2019-03-07 09:29:44"
    }  
}
```