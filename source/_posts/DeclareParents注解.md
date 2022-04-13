---
title: DeclareParents注解
date: 2021-08-11 15:30:04
tags: [Spring, AOP]
categories: Spring
top: 1
---

#### 前言

看 《Spring实战》 书籍的时候，在 *AOP面向切面* 看到了一个很有意思的注解： `@DeclareParents `。

Java 不是动态语言，一旦编译了，就很难再为该类添加新的功能。回想一下 Spring中切面，容器会实现包装 bean相同接口的代理类。如果除了实现这些接口，代理类也能暴露新的接口，不就是可以认为是在运行时，让这个 bean看起来多了某些方法吗？

```
被代理类 A
代理类 B
新接口 C
class B implements A, C
```

在面向切面编程（AOP）时，为了避免侵入性，减少对业务代码的结构影响，同时也遵循面向切面编程的思想，所以通过代理来增强被代理类的功能是更好的选择。面向对象编程的一个目的是，将非业务代码从业务代码中分离出来，那么应该通过代理实现增强被代理类，而不是将非业务代码（这里指通过代理增强被代理类的功能代码）写进业务代码模块中。

#### 解析

这个注解实际还是利用动态代理的模式，为代理类（运行时产生的字节码）实现一个接口。

 `@DeclareParents` 注解由 2个部分组成：

- value属性指定了那种类型的 bean 要引入该接口。
- defaultImpl 属性指定了为引入功能提供实现的类。

```java
@DeclareParents(value = "com.*.ClassX+", defaultImpl = Xxx.class)
```

#### 示例

 定义一个 Person 的空类及其子类  Man ：

```java
@Component
public class Person {
}

@Component
public class Man extends Person {
    public void sayIdentification() {
        System.out.println("我是男生。");
    }
}
```

 定义一个名为 EatService 的接口及它的实现类 EatServiceImpl。我们将要把 EatServiceImpl 的 eat() 方法添加到Person 子类型:

```java
@Component
public interface EatService {
    void eat(String food);
}

@Component
public class EatServiceImpl implements EatService {
    @Override
    public void eat(String food) {
        System.out.println(food);
    }
}
```

 SpringAop 配置类 ：

```java
@Component
@Aspect
public class AopConfig {
    // ...aop.Person后面的 + 号，表示只要是 Person及其子类都可以添加新的方法
    @DeclareParents(value = "com.tly.spring.aop.Person+", defaultImpl = EatServiceImpl.class)
    public EatService eatService;
}
```

 SpringConfig 配置类 ：

```java
@ComponentScan
@Configuration
@EnableAspectJAutoProxy
public class SpringConfig {
}
```

单元测试类：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = SpringConfig.class)
public class AopTest {

    @Autowired
    private Man man;

    @Test
    public void test() {
        // 通过类型转换，man 对象就拥有了 EatServiceImpl 类的方法
        EatService eatService = (EatService)man;
        eatService.eat("苹果");
        man.sayIdentification();
    }
}
```
