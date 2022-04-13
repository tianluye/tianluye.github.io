---
title: BoolQueryBuilder toString问题
date: 2021-07-18 17:27:07
tags: [搜索, ElasticSerach]
# 文章分类，若多个标签，则用数组表示 #
categories: ElasticSerach
# 文章出处名称 #
from: 
# 文章出处链接 #
url: https://tianluye.github.io/2021/07/18/BoolQueryBuilder-toString问题/
# 文章作者名称 #
author: tianluye
# 文章作者签名 #
about: 每天进步一点点，加油！
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: true
# 文章摘要 #
description: BoolQueryBuilder toString问题.
top: 1
---

线上环境，因为一个 toString 问题，导致了页面查询导出功能不可用。

一开始，很是纳闷，不就是打印个对象嘛，咋还能报错呢？查看错误日志：

```java
BAD_REQUEST_ERROR:org.springframework.web.util.NestedServletException: Handler dispatch failed; nested exception is java.lang.StackOverflowError
    at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:982)
    at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:901)
    at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:970)
    at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:872)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:707)
    at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:846)
    .......

Caused by: java.lang.StackOverflowError
    at com.fasterxml.jackson.core.json.UTF8JsonGenerator._flushBuffer(UTF8JsonGenerator.java:2085)
    at com.fasterxml.jackson.core.json.UTF8JsonGenerator.writeRaw(UTF8JsonGenerator.java:684)
    at com.fasterxml.jackson.core.util.DefaultIndenter.writeIndentation(DefaultIndenter.java:91)
    at com.fasterxml.jackson.core.util.DefaultPrettyPrinter.beforeObjectEntries(DefaultPrettyPrinter.java:284)
    at com.fasterxml.jackson.core.json.UTF8JsonGenerator._writePPFieldName(UTF8JsonGenerator.java:377)
    at com.fasterxml.jackson.core.json.UTF8JsonGenerator.writeFieldName(UTF8JsonGenerator.java:183)
    at org.elasticsearch.common.xcontent.json.JsonXContentGenerator.writeFieldName(JsonXContentGenerator.java:147)
    at org.elasticsearch.common.xcontent.XContentBuilder.field(XContentBuilder.java:288)
    at org.elasticsearch.common.xcontent.XContentBuilder.field(XContentBuilder.java:270)
    at org.elasticsearch.common.xcontent.XContentBuilder.startArray(XContentBuilder.java:233)
    at org.elasticsearch.index.query.BoolQueryBuilder.doXArrayContent(BoolQueryBuilder.java:194)
    at org.elasticsearch.index.query.BoolQueryBuilder.doXContent(BoolQueryBuilder.java:165)
    at org.elasticsearch.index.query.QueryBuilder.toXContent(QueryBuilder.java:37)
    at org.elasticsearch.index.query.BoolQueryBuilder.doXArrayContent(BoolQueryBuilder.java:196)
    at org.elasticsearch.index.query.BoolQueryBuilder.doXContent(BoolQueryBuilder.java:165)
    at org.elasticsearch.index.query.QueryBuilder.toXContent(QueryBuilder.java:37)
    at org.elasticsearch.index.query.BoolQueryBuilder.doXArrayContent(BoolQueryBuilder.java:196)
    at org.elasticsearch.index.query.BoolQueryBuilder.doXContent(BoolQueryBuilder.java:165)
    at org.elasticsearch.index.query.QueryBuilder.toXContent(QueryBuilder.java:37)
    at org.elasticsearch.index.query.BoolQueryBuilder.doXArrayContent(BoolQueryBuilder.java:196)
    at org.elasticsearch.index.query.BoolQueryBuilder.doXContent(BoolQueryBuilder.java:165)
    at org.elasticsearch.index.query.QueryBuilder.toXContent(QueryBuilder.java:37)
    at org.elasticsearch.index.query.BoolQueryBuilder.doXArrayContent(BoolQueryBuilder.java:196)
    at org.elasticsearch.index.query.BoolQueryBuilder.doXContent(BoolQueryBuilder.java:165)
    at org.elasticsearch.index.query.QueryBuilder.toXContent(QueryBuilder.java:37)
    at org.elasticsearch.index.query.BoolQueryBuilder.doXArrayContent(BoolQueryBuilder.java:196)
    at org.elasticsearch.index.query.BoolQueryBuilder.doXContent(BoolQueryBuilder.java:165)
    ......

```

居然发生了堆栈溢出异常！按照之前的经验，StackOverflowError 异常一般都是死循环导致的。继续看下面的日志，可以看到 `BoolQueryBuilder.doXArrayContent` 一直在循环输出，这样势必会占用大量的堆栈信息，也就导致了我们上面看到的栈溢出异常了。

---

**为什么会出现这样的问题？**

发现组装 ES查询条件的时候，有下面的代码：

```java
private BoolQueryBuilder buildQueryDsl(CollectTransferReq req) {
    BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
    // 仓库
    if (ObjectUtil.isNotNull(req.getWarehouseId())) {
        boolQuery.filter(QueryBuilders.termQuery("warehouseId", req.getWarehouseId()));
    }
    if (CollectionUtils.isNotEmpty(req.getNationalLineTypes())) {
        boolQuery.filter(QueryBuilders.termsQuery("nationalLineType", req.getNationalLineTypes()));
    }
    // 车牌号
    if (StringUtils.equals(req.getLicenseNumber(), WmsMagicValue.ONE_STR)) {
        boolQuery.filter(boolQuery.mustNot(QueryBuilders.termQuery("licenseNumber", "")));
        // boolQuery.mustNot(QueryBuilders.termQuery("licenseNumber", ""));
    }
    logger.info("集货转运查询组装dsl出参：{}", boolQuery);
    return boolQuery;
}
```

这就相当于是在 `BoolQueryBuilder` 对象中，引入了 `BoolQueryBuilder` 属性，导致在 toString的时候，循环调用，从来发生栈溢出！

修改后的代码：

```java
if (StringUtils.equals(req.getLicenseNumber(), WmsMagicValue.ONE_STR)) {
    boolQuery.mustNot(QueryBuilders.termQuery("licenseNumber", ""));
}
```