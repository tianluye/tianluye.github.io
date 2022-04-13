---
title: PageHelper原理分析
date: 2022-01-29 14:55:48
tags: [Spring, Mybatis]
categories: Spring
top: 2
---

### 前言

![image](https://cdn.jsdelivr.net/gh/tianluye/tianluye.github.io/image/00167.png)

某天，在排查 sql大结果集问题中，定位到一处页面列表查询代码，如下：

```java
@Override
public Response<Result<BoxUpDetailRsp>> listBoxUpDetails(XxxReq req) {
    LOGGER.info("XXXX查询入参：{}", JsonTools.defaultMapper().toJson(req));
    validateReq(req);
    PageHelper.startPage(req.getPageNum(), req.getPageSize());
    List<XxxQueryVo> xxxList = xxxMapper.selectBoxUpDetail(req);
    PageInfo<AtBoxUpDetailQueryVo> pageInfo = new PageInfo<>(xxxList);
    //组装数据
    List<BoxUpDetailRsp> rsp = bulidExample(xxxList);
    return Response.buildSuccessResponseWithResult(rsp, (int) pageInfo.getTotal());
}
```

具体的 SQL如下：

```sql
<select id="selectBoxUpDetail" parameterType="BoxUpDetailReq" resultType="AtBoxUpDetailQueryVo">
  select
  *****************
  from  table1 a
  left join table2 b on a.outbound_code = b.outbound_code
  left join table3 c on  a.goods_sn = c.goods_sn and a.outbound_code = c.outbound_code and a.size = c.size
  where a.is_delete = 0
  <if test="req.warehouseId != null and req.warehouseId!=''">
    ......
  </if>
  ......
</select>
```

瞬间感觉自己破案了，是因为分页没有生效！所以导致了这条 SQL直接查询出了 5W+行的数据。

### 思考

不过，仔细想想，感觉不大对劲，为什么报警的频率这么低？按理说，这个列表页每天都会有好多人在用的。于是，自己到灰度环境验证了一波，等了好几分钟，也没有看到群里的告警信息。。。

实践是检验真理的唯一标准，动手进行单元测试。结果却出乎意料的是，SQL语句中是有 `LIMIT` 分页信息的。。。瞬间感觉整个人都不好了~

明明没有在 SQL中看到 `LIMIT` 这样的关键字段，那么问题出现在哪里呢？

### 分析

回顾下代码，发现在查询之前调用了 Mybatis的插件方法：

```java
PageHelper.startPage(req.getPageNum(), req.getPageSize());
```

直觉告诉我，肯定是这个插件做了什么操作，使得这个 SQL具备了分页的能力。猜测可能动态改变 SQL语句了，于是查看其源码：

```java
public static <E> Page<E> startPage(int pageNum, int pageSize, boolean count, Boolean reasonable, Boolean pageSizeZero) {
    Page<E> page = new Page(pageNum, pageSize, count);
    page.setReasonable(reasonable);
    page.setPageSizeZero(pageSizeZero);
    Page<E> oldPage = getLocalPage();
    if (oldPage != null && oldPage.isOrderByOnly()) {
        page.setOrderBy(oldPage.getOrderBy());
    }
    // 这里是重点
    setLocalPage(page);
    return page;
}

protected static final ThreadLocal<Page> LOCAL_PAGE = new ThreadLocal();

protected static void setLocalPage(Page page) {
    LOCAL_PAGE.set(page);
}
```

看到这里，心中已经有了大概的猜测了：**应该是在执行目标方法之前，存储分页信息到线程本地变量中，在真正执行 SQL前，从当前线程中获取分页信息，动态改变其 SQL，让其具备分页的能力！**

### 验证

查看其 PageHelper对应的 Jar包下，发现了一个拦截器：`com.github.pagehelper.PageInterceptor` ，看到其实现的方法，上面的猜测感觉就验证了一大半了：

```java
@Intercepts({@Signature(
    type = Executor.class,
    method = "query",
    args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
), @Signature(
    type = Executor.class,
    method = "query",
    args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}
)})
public class PageInterceptor implements Interceptor {
        private volatile Dialect dialect;
    private String countSuffix = "_COUNT";
    protected Cache<String, MappedStatement> msCountMap = null;
    private String default_dialect_class = "com.github.pagehelper.PageHelper";

    public PageInterceptor() {
    }

    public Object intercept(Invocation invocation) throws Throwable {
        try {
            Object[] args = invocation.getArgs();
            MappedStatement ms = (MappedStatement)args[0];
            Object parameter = args[1];
            RowBounds rowBounds = (RowBounds)args[2];
            ResultHandler resultHandler = (ResultHandler)args[3];
            Executor executor = (Executor)invocation.getTarget();
            CacheKey cacheKey;
            BoundSql boundSql;
            if (args.length == 4) {
                boundSql = ms.getBoundSql(parameter);
                cacheKey = executor.createCacheKey(ms, parameter, rowBounds, boundSql);
            } else {
                cacheKey = (CacheKey)args[4];
                boundSql = (BoundSql)args[5];
            }

            this.checkDialectExists();
            List resultList;
            // skip方法中会去获取当前线程本地变量中存取的分页信息，skip返回 false
            if (!this.dialect.skip(ms, parameter, rowBounds)) {
                // 还是从当前线程本地变量中存取的分页信息，判断是否要在执行查询前，先进行统计
                // PageHelper.startPage只传分页信息，默认先 count属性为 true，可以手动改为 false
                if (this.dialect.beforeCount(ms, parameter, rowBounds)) {
                    Long count = this.count(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                    // 做了一层优化，将 count值存入到 Page对象中
                    // 若 count的结果值小于 0或者小于当前页面所需要的的数值，则不再进行查询，直接返回
                    if (!this.dialect.afterCount(count.longValue(), parameter, rowBounds)) {
                        Object var12 = this.dialect.afterPage(new ArrayList(), parameter, rowBounds);
                        return var12;
                    }
                }
                // 执行分页逻辑查询
                resultList = ExecutorUtil.pageQuery(this.dialect, executor, ms, parameter, rowBounds, resultHandler, boundSql, cacheKey);
            } else {
                resultList = executor.query(ms, parameter, rowBounds, resultHandler, cacheKey, boundSql);
            }

            Object var16 = this.dialect.afterPage(resultList, parameter, rowBounds);
            return var16;
        } finally {
            if (this.dialect != null) {
                this.dialect.afterAll();
            }

        }
    }
    ......
}
```

最后，在 finally模块，很贴心的帮我们移除掉当前线程存储的本地变量：

```java
public void afterAll() {
    AbstractHelperDialect delegate = this.autoDialect.getDelegate();
    if (delegate != null) {
        delegate.afterAll();
        this.autoDialect.clearDelegate();
    }
    clearPage();
}
```

### 最后

其实上述生产问题，是因为这个列表页面查询的导出逻辑不合理！直接在 PC服务导出，所以才会报警！正常导出服务，应该都在 Job服务上，而 Job服务配置的数据库用户大结果集的告警阈值是 6W。