---
layout: post
title: 深入理解Java集合框架-LinkedList
data: 2017/5/20
subtitle: ""
description: ""
thumbnail: "/img/collection.jpeg"
tags:
  - Java集合框架
categories:
  - Java
---


上篇文章中我们学了``ArrayList``, 知道了``ArrayList``比较适合需要频繁访问元素的场景. 但是在插入和删除元素时, 表现得效率低下. 这次, 我们来分析适合使用在频繁插入和删除元素的场景的集合: ``LinkedList``.

> ## UML

![](http://ww1.sinaimg.cn/large/006VdOYcgy1filur9c5nfj30p10h7dh8.jpg)

我们先来看看UML图, ``LinkedList``在继承关系上, 跟``ArrayList``基本相同. 我们这里只分析不同点.

* ``LinkedList``继承``AbstractSequentialList``, ``AbstractSequentialList``这个是顺序访问列表的默认实现类, 换句话说, ``LinkedList``访问元素时, 是顺序访问的, 不是像``ArrayList``那样, 具有随机访问元素的能力.
* 由于``LinkedList``是顺序访问列表, 因此它并没有实现``RandomAccess``接口.
* ``LinkedList``实现了``Deque``接口, ``Deque``接口是双向队列的一个接口.因此``LinkedList``可以看做是一个双端队列. 而且它还可以作为栈来使用.

> ## 解析

```java
    transient int size = 0;

    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;
```

我们来先看看``LinkedList``中的字段. ``size``记录集合内部的元素个数.``first``和``next``代表表头和表尾. ``Node``是``LinkedList``的内部类, 代表链表的结点.从这里可以看出, ``LinkedList``的内部实现是基于双向链表来实现的, 这也解释了为什么``LinkedList``适合使用在频繁插入和删除的场景.

> ### 构造方法

```java
    /**
     * Constructs an empty list.
     */
    public LinkedList() {
    }
```

这个构造函数是一个空实现.证明一开始初始化时, 内部是不给结点分配内存的.

> ### 添加元素

``LinkedList``既可以作为双向队列, 也可以作为栈. 说明其支持向队头队尾进行操作.我们接下来分析这些操作

> #### 队头插入元素

```java
    /**
     * Inserts the specified element at the beginning of this list.
     *
     * @param e the element to add
     */
    public void addFirst(E e) {
        linkFirst(e);
    }

        /**
     * Inserts the specified element at the front of this list.
     *
     * @param e the element to insert
     * @return {@code true} (as specified by {@link Deque#offerFirst})
     * @since 1.6
     */
    public boolean offerFirst(E e) {
        addFirst(e);
        return true;
    }

        /**
     * Pushes an element onto the stack represented by this list.  In other
     * words, inserts the element at the front of this list.
     *
     * <p>This method is equivalent to {@link #addFirst}.
     *
     * @param e the element to push
     * @since 1.6
     */
    public void push(E e) {
        addFirst(e);
    }
```

上面三个操作都是向队头添加元素, 最终调用``addFirst(E)``, 而``addFirst(E)``又会调用``linkFirst(E)``.向队头插入元素的实现都是这个函数, 我们来分析一波:

```java
    /**
     * Links e as first element.
     */
    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f); //创建一个新的结点,
        first = newNode;
        if (f == null)
            last = newNode; //如果队列为空的话,
        else
            f.prev = newNode; // 队列不为空的话.
        size++;
        modCount++; //记录内部结构发生变化的次数, 用于实现fast-fail机制.
    }
```

``linkFirst``函数将一个元素插入到队头, 如果队列为空的话, 会将新插入的元素赋值给队列结点. 如果不为空的话, 将该元素复制为插入前的队头结点的前驱.

> #### 队尾插入元素

```java
    /**
     * Inserts the specified element at the end of this list.
     *
     * @param e the element to insert
     * @return {@code true} (as specified by {@link Deque#offerLast})
     * @since 1.6
     */
    public boolean offerLast(E e) {
        addLast(e);
        return true;
    }

        /**
     * Adds the specified element as the tail (last element) of this list.
     *
     * @param e the element to add
     * @return {@code true} (as specified by {@link Queue#offer})
     * @since 1.5
     */
    public boolean offer(E e) {
        return add(e);
    }

        /**
     * Appends the specified element to the end of this list.
     *
     * <p>This method is equivalent to {@link #addLast}.
     *
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        linkLast(e);
        return true;
    }

        /**
     * Appends the specified element to the end of this list.
     *
     * <p>This method is equivalent to {@link #add}.
     *
     * @param e the element to add
     */
    public void addLast(E e) {
        linkLast(e);
    }

```

上面的方法都是向队尾添加元素.它们最终都会调用``linkLast``.

```java
    /**
     * Links e as last element.
     */
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```

``linkLast``中的实现会向队尾添加一个结点, 思路和前面的大致一样.这里不多分析.

> #### 指定位置插入

既然``LinkedList``是基于链表来实现的, 那么``LinkedList``肯定是支持指定位置插入.

```java
    /**
     * Inserts the specified element at the specified position in this list.
     * Shifts the element currently at that position (if any) and any
     * subsequent elements to the right (adds one to their indices).
     *
     * @param index index at which the specified element is to be inserted
     * @param element element to be inserted
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
```

``add(int, E)``方法用于向指定位置插入一个元素, 如果index是等于size的话, 会直接插入到队尾. 如果不是的话, 会用``linkBefore(element, node(index))``. 我们来看看``LinkedList``怎么通过index定位到一个结点的

```java
    /**
     * Returns the (non-null) Node at the specified element index.
     */
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
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

总体是比较index是否大于size / 2, 小于的话,就从队头遍历, 大于的话从队尾开始遍历. 这里虽然做了一些优化, 但是总体速度还是挺慢的. 接下来的``linkBefore``就是常规的向一个结点前插入元素的操作.

> ### 移除元素

移除元素的套路跟添加元素的套路一样. 有向队尾,队头,或者指定位置移除元素, 这三种操作, 最后都会调用到下面的方法:

```java
    /**
     * Unlinks non-null first node f.
     */
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }

    /**
     * Unlinks non-null last node l.
     */
    private E unlinkLast(Node<E> l) {
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }

    /**
     * Unlinks non-null node x.
     */
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
```

接下去的获得元素的操作跟插入元素的思路都差不多.这里不多说.我们接下来看看``LinkedList``中的迭代器.

> ### 迭代器

``LinkedList``中实现了两个迭代器, 一个是双向迭代器, 另外一个是单向迭代器.

> ## 总结

* ``LinkedList``是基于链表来实现的, 决定了它比较适合使用在需要频繁插入和删除的场景. 但是不适合使用在经常要访问特定位置的元素. 因为每次都会遍历链表.
* ``LinkedList``既可以作为双向队列,也可以作为栈来使用.
