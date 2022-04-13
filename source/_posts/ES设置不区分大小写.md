---
title: ES设置不区分大小写
date: 2021-07-18 16:48:55
tags: [搜索, ElasticSerach]
# 文章分类，若多个标签，则用数组表示 #
categories: ElasticSerach
# 文章出处名称 #
from: 
# 文章出处链接 #
url: https://tianluye.github.io/2021/07/18/ES设置不区分大小写/
# 文章作者名称 #
author: tianluye
# 文章作者签名 #
about: 每天进步一点点，加油！
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: true
# 文章摘要 #
description: ES设置不区分大小写.
top: 1
---

默认情况下，ES查询是区分大小写的。但 ES也支持自定义 `settings` 和 `mappings` 中指定具体哪些字段不区分大小写。

创建索引的时候指定 `sex` 字段（注意，一定要是字符串类型！）不区分大小写。

```json
curl -XPUT 'http://10.123.7.3:9200/user' -d '{
  "settings": {
    "index": {
      "number_of_shards": "5",
      "number_of_replicas": "1",
      "max_result_window": "60000"
    },
    "analysis": {
      "normalizer": {
        "my_normalizer": {
          "type": "custom",
          "filter": [
            "lowercase"
          ]
        }
      }
    }
  },
  "mappings": {
    "doc": {
      "properties": {
        "id": {
          "type": "integer"
        },
        "name": {
          "type": "keyword"
        },
        "sex": {
          "type": "keyword",
          "normalizer": "my_normalizer"
        }
      }
    }
  }
}'
```

插入一条数据记录：

```
curl -XPOST 'http://10.123.7.3:9200/user/doc/1' -d '{
  "name":"Tom",
  "sex": "F"
}'
```

![image](https://cdn.jsdelivr.net/gh/tianluye/tianluye.github.io/image/00160.png)

---

使用 `name = 'tom'` 进行搜索，测试 ES默认区分大小写，结果应该是查询不到数据的：

![image](https://cdn.jsdelivr.net/gh/tianluye/tianluye.github.io/image/00161.png)

使用 `sex = 'f'` 进行搜索，测试上面设置的 `normalizer` 是否生效，结果应该是查询得到数据的：

![image](https://cdn.jsdelivr.net/gh/tianluye/tianluye.github.io/image/00162.png)

待办：https://zhuanlan.zhihu.com/p/358832729

