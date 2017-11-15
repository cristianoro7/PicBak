---
layout: post
title: 深入理解Java集合框架-ArrayList
data: 2017/5/20
subtitle: ""
description: ""
thumbnail: "/img/collection.jpeg"
tags:
  - Java集合框架
categories:
  - Java
---


> ## UML

![](http://ww1.sinaimg.cn/large/006VdOYcgy1filh3gnlr2j30n90iw75f.jpg)

从上面的UML图, 我们可以看出 ArrayList实现了三个标记接口, 他们分别是:RandomAccess, Serializable, Cloneable. RandomAccess接口表示ArrayList支持随机访问其中的元素, 也就是ArrayList可以随机访问其中的元素, 并且时间复杂度为O(1). Serializable接口表示ArrayList可以被序列化. Cloneable接口说明ArrayList可以被克隆(内部实现为浅克隆).

回到UML图, ``Collection``接口是集合的一个基类接口,它继承了``Iterable``接口,将遍历的任务交给了Iterable接口. ``AbstracCollection``是一个抽象类, 他实现了``Collection``接口, 在其内部实现了一些默认行为, 同理``AbstractList``也是一个抽象类, 实现了``List``接口的一些默认行为. 从这里可以看出, Java的集合框架用到了适配器的模式, 利用AbstractXX一系列抽象类来实现一些默认行为, 其他的让具体子类去实现或者重写.

> ## 解析

分析完``ArrayList``的继承结构后, 我们开始来分析``ArrayList``的实现.

```java
 /**
     * Default initial capacity.
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    private int size;
```

上面为``ArrayList``中的字段, 其中``elementData``字段是保存 ``ArrayList``中的元素, 从这点可以看出, ``ArrayList``的底层数据结构是数组. ``size``字段记录集合中的元素.

> ### 构造方法

```java
    /**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

平常我们使用ArrayList时, 都会调用空的构造方法, 如上.这个构造方法中, 只是简单的将``elementData``赋值为``DEFAULTCAPACITY_EMPTY_ELEMENTDATA``, 也就是一个空的数组对象. 因为此时我们并没有添加元素, 构造一个空的数组也是合情合理的.

> ### 添加元素

实例已经得到了, 我们来看看添加元素的方法:

```java
/**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```

我们要向集合添加元素, 由于之前我们构造的是一个空的数组, 那么内部肯定会帮我们扩容数组, 也就是``ensureCapacityInternal(size + 1)``. 我们进入该方法:

```java
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
```

 这个方法主要是比较一下``elementData``对象是不是为空, 空的话, 就取``DEFAULT_CAPACITY``和``minCapacity``字段的最大值. 由于我们数组的大小为0, 所以, 取得的是``DEFAULT_CAPACITY``(也就是10). 进入:``ensureExplicitCapacity(minCapacity)``

```java
     private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```

``modCount``是用来记录``ArrayList``内部结构发生变化的次数, 主要用来实现``fast-fail``机制. 接着会调用``grow(int)``, 该方法是``ArrayList``扩容的方法.

 我们接下去,看看``ArrayList``的扩容策略:

```java
     private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1); //右移1位,也就是除以2
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

扩容的大小为: 新容量 = 旧容量+旧容量 / 2. 在计算出新容量后, 会与旧容量作差, 再根据结果进行扩容. 主要有下面两种情况:

* 当初始化时(也就是调用空的构造方法), 首次添加会扩容为10.
* 下次扩容时, 会加上原来容量的一半.

> ### 随机访问元素

经过前面的分析, ``ArrayList``实现了``RandomAccess``接口, 表示``ArrayList``具有随机访问元素的能力, 这种能力是数组本身就有的.

```java
    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
```

在``get(E)``中, 会首先检查一下index是否会越界, 会的话, 直接抛异常, 不会的话, 直接访问数组对应的index.


> ### 移除元素

```java
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```

上面是``ArrayList``中的移除元素的方法. 首先检查边界, 没有越界的话, 通过index访问元素. 接着再将index后面的元素向先移动. 最后手动将被移除的元素复制为null, 让其能够被GC回收.

这里需要注意的是: 虽然元素被移除了, 但是空间还是留着.

> ### 迭代器模式

在Java集合框架中, 使用了迭代器模式去遍历集合中的元素, 这使得在遍历元素时, 不用去关心集合的底层实现的数据结构, 各种集合只需要实现符合自己数据结构的迭代器.

在``ArrayList``中, 默认实现了两个迭代器, 它们分别是:单向迭代器, 双向迭代器.

> #### 单向迭代器

单向迭代器是众多迭代器的最简单的,也是最常用的. 内部实现是只能向一个方向遍历数据.

> #### 双向迭代器

与单向迭代器比, 双向迭代器能够向两个方向遍历数据

> #### fast-fail机制

如果一个线程利用迭代器遍历集合时, 另外一个线程向集合中添加元素. 这时会抛出异常. 这也说明了``ArrayList``是线程不安全的.

> ## 总结

* ``ArrayList``实现的底层数据结构为数组.
* 由于``ArrayList``是基于数组来实现的, 因此决定了``ArrayList``的使用场景. 它适合用在需要频繁访问元素的场景, 因为其快速访问特性. 但是它不适合使用在需要频繁随机插入和删除的场景, 因为数组每次随机插入和删除元素时, 都需要移动后面的元素.
* ``ArrayList``具有自动扩容的特性, 默认的容量为10, 后面每次扩容都会增加原来的一半.
