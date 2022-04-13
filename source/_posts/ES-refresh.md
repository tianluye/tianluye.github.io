---
title: ES refresh
date: 2021-08-12 16:38:44
tags: [搜索, ElasticSerach]
# 文章分类，若多个标签，则用数组表示 #
categories: ElasticSerach
# 文章出处名称 #
from: 
# 文章出处链接 #
url: https://tianluye.github.io/2021/08/12/ES_refresh/
# 文章作者名称 #
author: tianluye
# 文章作者签名 #
about: 每天进步一点点，加油！
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: true
# 文章摘要 #
description: ES refresh.
top: 1
---

### 前言

最近做了 **ES 数据异构** 这个项目，大致就是涉及宽表，将 Mysql 中若干张表的数据整合到 ES 中。项目中使用 **Rabbit MQ** 做消息同步，若指定表的数据发生变更，则会推送一条消息到 MQ 服务器上，然后再客户端进行消费，将变更的数据刷新到 ES 中。

大致的设计流程如下：

![image](https://cdn.jsdelivr.net/gh/tianluye/tianluye.github.io/image/00166.png)

---

### 问题起因

业务方在操作波次下发后，前端会再次请求查询接口，返回渲染页面。但是没有将最新的结果加载出来，必须要业务方再次点击查询按钮，才可以正确显示操作后的结果。

首先，我们需要明确的是： Elastic Search 是  <font color=#0099ff>近实时搜索</font> ！如果想要实时搜索，那么就不要选择使用 ES 进行存储了。至于原因，后面再进行分析。

>  尽管刷新是比提交轻量很多的操作，它还是会有性能开销。 当写测试的时候， 手动刷新很有用，但是不要在生产环境下每次索引一个文档都去手动刷新。 相反，你的应用需要意识到 Elasticsearch 的近实时的性质，并接受它的不足。 

除此之外，可以看到设计中，是将消息推送到 MQ 服务上后，再由消费者进行消费的，这个过程，也会带来一定的延迟。

---

### 解决问题的思路

#### 1、延迟加载

考虑使用 sleep 来延迟操作，但是没法保证一定准确，因为具体的业务处理时间和 ES 刷盘耗时未知。

####  2、调整数据刷新方案 

 如果需要调整数据刷新方案，则有三种途径： 

- 设置数据刷新间隔：`refresh_interval`。
- 调用数据刷新接口：`_refresh`。
- 设置数据刷新策略：`RefreshPolicy`。

 ##### 2.1、在ELK中，ElasticSearch 的主要应用在于大量写日志，而非近实时搜索，此时可以增大刷新间隔。如下： 

```
# 调整所有index的刷新间隔位5分钟
PUT
{
  "settings": {
    "refresh_interval": "5m" 
  }
}

# 调整指定index的刷新间隔为180秒
PUT /order_log
{
  "settings": {
    "refresh_interval": "180s" 
  }
}

```

 ##### 2.2、可以手动调用ElasticSearch提供的API进行数据刷新，如下： 

```
# 刷新全部index的数据
POST /_refresh 

# 刷新指定index的数据
POST /user/_refresh
```

 ##### 2.3、枚举`org.elasticsearch.action.support.WriteRequest.RefreshPolicy`定义了三种策略： 

- `RefreshPolicy#IMMEDIATE`:

  > 请求向ElasticSearch提交了数据，立即进行数据刷新，然后再结束请求。
  > 优点：实时性高、操作延时短。
  > 缺点：资源消耗高。

- `RefreshPolicy#WAIT_UNTIL`:

  > 请求向ElasticSearch提交了数据，等待数据完成刷新，然后再结束请求。
  > 优点：实时性高、操作延时长。
  > 缺点：资源消耗低。

- `RefreshPolicy#NONE`:

  > 默认策略。
  > 请求向ElasticSearch提交了数据，不关系数据是否已经完成刷新，直接结束请求。
  > 优点：操作延时短、资源消耗低。
  > 缺点：实时性低。

```java
/**
 * ElasticSearch立即更新的示例代码
 */
@Test
public void refreshImmediatelyTest() {
    //删除操作
    client.prepareDelete("index", "type", "1").setRefreshPolicy(WriteRequest.RefreshPolicy.IMMEDIATE).get();

    //索引操作
    client.prepareIndex("index", "type", "2").setSource("{\"age\":1}").setRefreshPolicy(WriteRequest.RefreshPolicy.IMMEDIATE).get();

    //更新操作
    client.prepareUpdate("index", "type", "3").setDoc("{\"age\":1}").setRefreshPolicy(WriteRequest.RefreshPolicy.IMMEDIATE).get();

    //批量操作
    client.prepareBulk()
            .add(client.prepareDelete("index", "type", "1"))
            .add(client.prepareIndex("index", "type", "2").setSource("{\"age\":1}"))
            .add(client.prepareUpdate("index", "type", "3").setDoc("{\"age\":1}"))
            .setRefreshPolicy(WriteRequest.RefreshPolicy.IMMEDIATE).get();
}

```

#### 3、最终解决的方案

最后分析，还是选择使用 `refresh` 操作来进行立即更新。因为项目中的 ES 客户端是 Jest，所以调整后的代码为：

```java
public boolean saveOrUpdate(WellenRoute wellenRoute, boolean isRefreshNow) {
    LOGGER.info("波次功能列表数据同步ES索引 WellenRoute: {}, isRefreshNow: {}", JsonTools.defaultMapper().toJson(wellenRoute), isRefreshNow);
    JestResult jestResult;
    try {
        Index.Builder route = new Index.Builder(wellenRoute).index(WELLEN_INDEX).type("wellenRoute");
        if (isRefreshNow) {
            route.setParameter("refresh", "wait_for");
        }
        Index index = route.build();
        jestResult = jestMultiThreadClient.execute(index);
        if (jestResult == null || !jestResult.isSucceeded()) {
            LOGGER.error("【出库组-波次功能列表数据刷新到 ES失败】: {}", jestResult == null ? "" : jestResult.getJsonString());
            throw new BblBusinessException(AtMessageCode.BAD_REQUEST, "波次功能列表数据刷新到ES失败");
        }
    } catch (Exception e) {
        LOGGER.error("【出库组-波次功能列表数据刷新到ES异常】: {}", ExceptionTools.getExceptionStackTrace(e));
        throw new BblBusinessException(AtMessageCode.BAD_REQUEST, "波次功能列表数据刷新到 ES异常");
    }
    return jestResult == null ? false : jestResult.isSucceeded();
}
```

---

**ES 客户端 Jest 指南**：https://github.com/searchbox-io/Jest/tree/master/jest

**refresh 参考**：https://www.elastic.co/guide/en/elasticsearch/reference/6.8/docs-refresh.html

