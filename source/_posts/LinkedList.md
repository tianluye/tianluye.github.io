---
# 文章展示的标题 #
title: LinkedList
# 文章创建日期 #
date: 2021-07-17 15:40:25
# 文章标签，若多个标签，则用数组表示 #
tags: [数据结构, Java]
# 文章分类，若多个标签，则用数组表示 #
categories: Java基础
# 文章出处名称 #
from: 
# 文章出处链接 #
url: https://tianluye.github.io/2021/07/17/LinkedList/
# 文章作者名称 #
author: tianluye
# 文章作者签名 #
about: 道无涯，学无止境
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: true
# 文章摘要 #
description: 链表数组.
top: 10
---



LinkedList使用双向链表实现存储（将内存中零散的内存单元通过附加的引用关联起来，形成一个可以按序号索引的线性结构，这种链式存储方式与数组的连续存储方式相比，内存的利用率更高），按序号索引数据需要进行前向或后向遍历（速度慢于 ArrayList），插入数据时只需要记录本项的前后项即（速度快于 ArrayList）。

LinkedList双向链表的关键在于节点的一个引用（指针），它能指出上一个节点和下一个节点。

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    // LinkedList节点构造，指定前面和后面的节点
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

LinkedList定义了链表的 size（长度）、first（头）、last（尾），而其构造器只有 2个：

```java
public LinkedList() {
    // 默认情况下，链表的长度为 0，不像 ArrayList初始化长度为 10
    // 毕竟链表需要额外的引用，初始化长度会造成内存的浪费
}

public LinkedList(Collection<? extends E> var1) {
    this();
    this.addAll(var1); // 初始化的时候，可以传入一个集合，作为链表的内容
}
```

```java
public boolean addAll(int var1, Collection<? extends E> var2) {
    // 检测新增的位置是否合法（var1大于等于0，且小于等于链表的长度）
    // 将集合 var2转为数组
    // 根据插入的位置，确定插入点的前后节点 node
    // 遍历数组，将里面的元素和链表连接起来
    // 若后节点为 null，则将现在节点的最后一个节点作为链表的尾节点
}
```

链表的搜索：实则就是移动链表的指针，看其指向的 item是否等于目标元素，返回计数器。

```java
// 注意：链表中的元素是可以重复的，这里只返回第一个元素的位置
public int indexOf(Object var1) {
    int var2 = 0;
    LinkedList.Node var3;
    if(var1 == null) {
        for(var3 = this.first; var3 != null; var3 = var3.next) {
            if(var3.item == null) {
                return var2;
            }
            ++var2;
        }
    } else {
        for(var3 = this.first; var3 != null; var3 = var3.next) {
            if(var1.equals(var3.item)) {
                return var2;
            }
            ++var2;
        }
    }
    return -1;
}
```

从链表中去除某个节点：

```java
E unlink(LinkedList.Node<E> var1) {
    Object var2 = var1.item;
    LinkedList.Node var3 = var1.next; // var3 -> 下一个节点
    LinkedList.Node var4 = var1.prev; // var4 -> 上一个节点
    if(var4 == null) { // 去除的节点是否是第一个节点
        this.first = var3;
    } else {
        var4.next = var3;
        var1.prev = null;
    }

    if(var3 == null) { // 去除的节点是否是最后一个节点
        this.last = var4;
    } else {
        var3.prev = var4;
        var1.next = null;
    }

    var1.item = null;
    --this.size;
    ++this.modCount;
    return var2; // 返回移除的节点
}
```

从链表中去除某个元素，就是遍历得到第一个匹配的节点，将其 unlink掉。

```java
public boolean remove(Object var1) {
    
}
```

从链表中获取指定 index位置的元素（折半查找）：

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

Node<E> node(int index) {
    if (index < (size >> 1)) { // 判断 index的位置是在链表的前半部分还是后半部分
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

从链表中改变指定 index位置的元素值：

```java
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
```

其他的操作方式思想都是取移动链表的指针来完成的。