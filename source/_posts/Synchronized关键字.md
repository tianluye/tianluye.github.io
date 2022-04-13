---
title: Synchronized关键字
date: 2021-07-17 19:24:13
# 文章标签，若多个标签，则用数组表示 #
tags: [Java, 并发]
# 文章分类，若多个标签，则用数组表示 #
categories: Java基础
# 文章出处名称 #
from: 
# 文章作者名称 #
author: tianluye
# 文章作者签名 #
about: 道无涯，学无止境
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: false
# 文章摘要 #
description: Synchronized关键字.
top: 1
---

synchronized是 Java中的关键字，是一种同步锁。它修饰的对象有以下几种：

- 修饰一个代码块，被修饰的代码块称为同步语句块，其作用的范围是大括号{}括起来的代码，作用的对象是调用这个代码块的对象
- 修饰一个方法，被修饰的方法称为同步方法，其作用的范围是整个方法，作用的对象是调用这个方法的对象
- 修改一个静态的方法，其作用的范围是整个静态方法，作用的对象是这个类的所有对象
- 修改一个类，其作用的范围是synchronized后面括号括起来的部分，作用主的对象是这个类的所有对象

可以看出，修饰代码块和方法，同步范围都是调用者这个对象。修饰的是 static静态方法（静态方法属于类，而不属于对象）和类，同步范围就是 class类，而不是这个类对应的实例对象。

---

#### 修饰代码块、修饰方法

```java
public class SyncThread implements Runnable {
    private static int count;
    public SyncThread() {
        count = 0;
    }
    @Override
    public void run() {
        synchronized (this) { // 此处的 this就是调用者对象
            for (int i = 0; i < 5; i++) {
                try {
                    System.out.println(Thread.currentThread().getName() + ": " + (count++));
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
    public static int getCount() {
        return count;
    }
}

// main方法调用
SyncThread syncThread = new SyncThread();
Thread thread1 = new Thread(syncThread, "SyncThread1");
Thread thread2 = new Thread(syncThread, "SyncThread2");
thread1.start();
thread2.start();
```

可以看到结果是先打印了 SyncThread1线程的执行结果，全部执行完了，才会去执行 SyncThread2。因为使用了 synchronized关键字修饰了 this对象，线程 SyncThread1，SyncThread2是并发执行的，由于一开始线程 SyncThread1获得了 this对象锁，所以线程 SyncThread2就处于阻塞状态了。当线程 SyncThread1让出对象锁后，SyncThread2才能获得到对象锁，才能进行后续操作。

如果将上面的实现 Runnable接口的实例，换成 2个，结果就不一样了。

```java
Thread thread1 = new Thread(new SyncThread(), "SyncThread1");
Thread thread2 = new Thread(new SyncThread(), "SyncThread2");
thread1.start();
thread2.start();
```

这样运行后，就可以看到并发效果了。（加锁的对象就不是同一个对象了，只是同一种 class类型而已）

这里面需要注意的是，synchronized关键字所加的位置的不同，对象锁的作用范围也是不同的。
就比如，synchronized关键字修饰的是代码块和方法时，如果一个线程已经获取到了对象锁的话，就可以访问到这个对象的所有方法，包括 synchronized关键字修饰的方法。而其他线程由于获取不到这个对象锁，只能访问普通的方法，而不能访问 synchronized关键字修饰的同步方法。

当没有明确的对象作为锁，只是想让一段代码同步时，可以创建一个特殊的对象来充当锁：

```java
public class Test implements Runnable {
   private byte[] lock = new byte[0];  // 特殊的instance变量
   public void method() {
      synchronized(lock) {
         // todo 同步代码块
      }
   }

   public void run() {

   }
}
```

说明：零长度的byte数组对象创建起来将比任何对象都经济――查看编译后的字节码：生成零长度的byte[]对象只需3条操作码，而Object lock = new Object()则需要7行操作码。

实际上，synchronized修饰方法和代码块是基本一致的，都是锁定了整个方法时的内容：

```java
public synchronized void say() {
    // ...
}

public void say() {
    synchronized (this) {
        // ...
    }
}
```

在用synchronized修饰方法时要注意以下几点：

- synchronized关键字不能继承，如果子类重写了父类的同步方法，那么子类的方法不是同步方法。但是子类中可以调用父类的同步方法：super.method();
- 在定义接口方法时不能使用synchronized关键字
- 构造方法不能使用synchronized关键字，但可以使用synchronized代码块来进行同步

#### 修饰一个静态的方法和类

静态方法是属于类的而不属于对象的。同样的，synchronized修饰的静态方法锁定的是这个类的所有对象

```java
public synchronized static void method() {}

class ClassName {
   public void method() {
      synchronized(ClassName.class) {
         // todo
      }
   }
}
```

#### 总结

- 无论synchronized关键字加在方法上还是对象上，如果它作用的对象是非静态的，则它取得的锁是对象；如果synchronized作用的对象是一个静态方法或一个类，则它取得的锁是对类，该类所有的对象同一把锁
- 每个对象只有一个锁（lock）与之相关联，谁拿到这个锁谁就可以运行它所控制的那段代码
- 实现同步是要很大的系统开销作为代价的，甚至可能造成死锁，所以尽量避免无谓的同步控制

参考：https://blog.csdn.net/luoweifu/article/details/46613015