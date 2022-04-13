---
title: ArrayList
date: 2021-07-17 15:30:46
tags: [数据结构, Java]
categories: Java基础
top: 1
---

ArrayList使用动态数组的方式存储数据，此数组元素数大于实际存储的数据以便增加和插入元素，允许直接按序号索引元素，所以在查询搜索上速度比较快。但是插入元素要涉及数组元素移动等内存操作，包括了一些列的数组 copy操作，效率比较慢。

ArrayList和LinkedListed都是非线程安全的，可以通过工具类Collections中的synchronizedList方法将其转换成线程安全的容器后再使用。

定义了 2个私有属性：Object[] elementData（ArrayList用来存储的动态数组），int size（实际存储的数据个数）。所以，elementData.length >= size。

ArrayList提供了三个构造器：

```java
public ArrayList(int initialCapacity) { // initialCapacity，自定义初始化的动态数组长度
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
    this.elementData = new Object[initialCapacity];
}
public ArrayList() {
    this(10); // 不带参数的构造器，默认为存储的动态数组分配了 10个长度。
}
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray(); // 将集合元素分配到动态数字里
    size = elementData.length; // 此时，size的大小等于动态数字的长度
    // c.toArray might (incorrectly) not return Object[] (see 6260652)
    if (elementData.getClass() != Object[].class)
        elementData = Arrays.copyOf(elementData, size, Object[].class);
}
```

ArrayList并不能确定确定数据量的大小，所以使用动态数组进行存储。这也导致了必然会出现了内存空间的浪费。提供了 trimToSize，减少空间的浪费，但是却增加了时间的浪费。

```java
public void trimToSize() {
    modCount++; // modCount，记录 list的变化次数
    int oldCapacity = elementData.length;
    if (size < oldCapacity) {
        // 进行数组拷贝，减少了空间花销，代价是增大了时间消耗
        elementData = Arrays.copyOf(elementData, size);
    }
}
```

当动态数组的空间被使用完了，就需要去扩展其长度，又是要进行数组拷贝。

```java
// 向外部暴露出的方法，参数是要存储数据的个数，即 size
public void ensureCapacity(int minCapacity) {
    if (minCapacity > 0)
        ensureCapacityInternal(minCapacity);
}

private void ensureCapacityInternal(int minCapacity) {
    modCount++; // 不管是否扩容，都要自增
    if (minCapacity - elementData.length > 0) // 判断是否需要扩容
        grow(minCapacity);
}
// 正真的扩容逻辑
private void grow(int minCapacity) {
    int oldCapacity = elementData.length; // 获取原动态数组的长度
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 扩大为原来的 1.5倍，右移是除法
    if (newCapacity - minCapacity < 0) // 取参数和扩容后的最大值
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0) // 判断是否超过定义的动态数组长度最大值
        newCapacity = hugeCapacity(minCapacity);
    // 扩容成功后，将原动态数组里面的内容拷贝到新数组里面
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

注意，调用 size()、isEmpty()等方法，长度指的都是 size，而不是动态数组的长度。获取元素在 ArrayList中的位置（只返回第一个匹配），也是要进行遍历动态数组的，取值就可以直接使用下标。

ArrayList的克隆（实现了 Cloneable接口）

```java
public Object clone() {
    try {
        @SuppressWarnings("unchecked")
        ArrayList<E> v = (ArrayList<E>) super.clone(); // 进行 clone
        // 设置 clone后的动态数组，再次拷贝
        v.elementData = Arrays.copyOf(elementData, size);
        v.modCount = 0; // 将修改次数置为 0
        return v;
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError();
    }
}
```

将 ArrayList转为数组（从动态数组里，拷贝长度为 size个元素）：

```java
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}
```

ArrayList的取值和赋值：

```java
public E get(int index) {
    rangeCheck(index); // 判断 index是否合法
    return elementData(index);
}
public E set(int index, E element) {
    rangeCheck(index);
    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue; // 设置值，返回旧值。注意，设置值的 index最大等于 size
}
```

ArrayList的新增：

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!! 如果需要扩容，进行扩容
    elementData[size++] = e; // 同时 size的大小也自增了
    return true;
}

public void add(int index, E element) {
    rangeCheckForAdd(index); // 判断新增的 index是否合法 !(index > size || index < 0)
    ensureCapacityInternal(size + 1);  // Increments modCount!! 如果需要扩容，进行扩容
    // 被 native修饰
    System.arraycopy(elementData, index, elementData, index + 1, size - index);
    // 由于上面的拷贝，index处元素可以被新元素替换掉，而不会影响其他位置的元素。
    elementData[index] = element;
    size++; // 长度自增
}

// 把 dest从 destPos位置开始，拷贝 length个元素，拼接在 src的 srcPos位置后
public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos, int length);
```

ArrayList的移除：

```java
public E remove(int index) {
    rangeCheck(index);
    modCount++;
    E oldValue = elementData(index); // 将要被移除的值，作为该方法的返回值
    int numMoved = size - index - 1;
    if (numMoved > 0) // 判断移除的是否是最后一个元素
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null; // 让 oldValue被 GC进行回收
    return oldValue;
}
```

ArrayList的清空（可以看出，modCount是针对于整个 ArrayList，而非里面的单个元素）：

```java
public void clear() {
    modCount++;
    // Let gc do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;
    size = 0;
}
```

ArrayList的操作，是基于动态数组，数据变化大都使用数组拷贝，所以操作起来，性能较差。