---
layout: post
title: 深入理解Java集合框架-LinkHashMap
data: 2017/5/20
subtitle: ""
description: ""
thumbnail: "/img/collection.jpeg"
tags:
  - Java集合框架
categories:
  - Java
---


上篇我们分析了``HashMap``, 知道了遍历``HashMap``时, 顺序是不能够保证的.如果遍历时需要顺序, 那么应该用``LinkedHashMap``, 也就是我们这次要来分析的另外一个集合类.

> ## UML

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fio5jh7fepj30lg0hkgm9.jpg)

从UML图来看, ``LinkedHashMap``的继承于``HashMap``, 可见``LinkedHashMap``是基于``HashMap``来扩展的. 如果理解了``HashMap``的话, 那``LinkedHashMap``应该算是很简单了.

> ## 解析

* ``LinkedHashMap``是底层的数据结构是``HashMap``和``双向链表``. ``LinkedHashMap``内部维护着一条双向链表, 在遍历``LinkedHashMap``时, 会遍历内部的链表, 这样就可以保证遍历的顺序是特定的.至于遍历的顺序, 有两种可选: 按照插入的顺序(默认); 按照访问的顺序(结点每次被访问时, 都会把结点移动到链表尾).

* 由于``LinkedHashMap``内部维护一条链表, 因此它在性能上要稍微逊色于``HashMap``.

> ### 结点

```java
/**
 * HashMap.Node subclass for normal LinkedHashMap entries.
 */
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

``LinkedHashMap``内部的Entry继承于``HashMap.Entry``, 并且加了两个成员变量, 用来记录前一个结点和后后一个结点.

```java
    /**
     * The head (eldest) of the doubly linked list.
     */
    transient LinkedHashMap.Entry<K,V> head;

    /**
     * The tail (youngest) of the doubly linked list.
     */
    transient LinkedHashMap.Entry<K,V> tail;
```

``head``为头结点, ``tail``为尾结点. 分析到这里, 我们可以确认``LinkedHashMap``内部确实维护一条双向链表.

> ### hock函数

还记得上次分析``HashMap``时,说过``HashMap``留给``LinkedHashMap``的三个hock函数吗? 不记得的话也没关系, 耐心看下面的分析就会想起来了.

```java
    // Callbacks to allow LinkedHashMap post-actions
    void afterNodeAccess(Node<K,V> p) { } //结点被访问后, LinkedHashMap回调的函数
    void afterNodeInsertion(boolean evict) { } //结点插入时, LinkedHashMap回调的函数
    void afterNodeRemoval(Node<K,V> p) { } //结点被移除时, LinkedHashMap回调的函数
```

上面三个hock函数定义在``HashMap``内部, 并且为空的函数.这三个函数是预留给``LinkedHashMap``用的. 当结点被访问, 插入和被移除时,``LinkedHashMap``回调的函数.

> ###  put(K, V)和afterNodeInsertion(boolean evict)

``LinkedHashMap``内部没有重写``put(K, V)``函数, 意味着它的话复用了``HashMap``的``put(K, V)``方法, 那...我们暂且回调``HashMap``吧.

```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict); //回调函数
        return null;
    }
```

在``put(K, V)``内部会调用`` afterNodeInsertion(evict)``. 回到``LinkedHashMap``看看这个函数的实现.

```java
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
```

``LinkedHashMap``这个函数的实现是为了移除比较``老``的元素(越``老``的元素会在链表的越前面).但是这个函数的默认实现总是不会移除``老``的元素. 因为

```java
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
```

这个方法一直返回false. 重写这个方法, 我们可以控制当``LinkedHashMap``内部的元素数量达到一定数量时, 移除比较老的元素.

前面说``LinkedHashMap``是可以根据插入的顺序进行遍历, 那么内部肯定在``put(K, V)``的时候, 会构造链表.那么构造的操作在哪里?

```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
```

``LinkedHashMap``重写了newNode(int, K, V, Node<V, K>), 在其中写了构造链表的逻辑. 具体实现``linkNodeLast``

```java
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null) //如果为尾结点为空,证明链表为空, 直接将头结点和为结点指点新插入的结点
            head = p;
        else { //插入新结点
            p.before = last;
            last.after = p;
        }
    }
```

上面的逻辑为向双向链表中插入结点.

> ### get(K)和afterNodeAccess(Node<K,V> e)

```java
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
```

前面我们说过, ``LinkedHashMap``支持按照访问的顺序来遍历.而get(K)操作正是访问操作. 默认情况下, ``LinkedHashMap``是按照插入顺序来遍历的, 因为字段``accessOrder``字段默认为``false``, 如果要支持按照访问顺序的话, 需要显示调用这个构造方法, 将``accessOrder``指定为``true``

```java
public LinkedHashMap(int initialCapacity, float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```

我们接着看看``afterNodeAccess``这个回调函数

```java
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```

函数的逻辑为: 将访问的结点移动到链表的尾端.

至于``remove(K)``函数和``afterNodeRemoval(Node<K,V> e)``的关系和前面将的两个回调函数的思想一样. ``remove(K)``内部会回调``afterNodeRemoval(Node<K, V> e)``函数, ``afterNodeRemoval(Node<K, V> e)``会将结点从双向链表中移除.

> ### 迭代器

和``HashMap``一样, 内部有提供了三个视图(KeySet, ValueSet, set<Entry<K, V>), 因此, ``LinkedHashMap``内部也有三个迭代器. 不同的是, ``LinkedHashMap``迭代器的遍历顺序是有的. 下面我只分析``LinkedValueIterator``.

```java
    abstract class LinkedHashIterator {
        LinkedHashMap.Entry<K,V> next;
        LinkedHashMap.Entry<K,V> current;
        int expectedModCount;

        LinkedHashIterator() {
            next = head;
            expectedModCount = modCount;
            current = null;
        }

        public final boolean hasNext() {
            return next != null;
        }

        final LinkedHashMap.Entry<K,V> nextNode() {
            LinkedHashMap.Entry<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            current = e;
            next = e.after;
            return e;
        }

        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }
```

``LinkedHashIterator``是``LinkedHashMap``内部迭代器的基类, 在构造函数将链表的头结点复制给 ``next``, 然后``nextNode()``函数内, 顺序的访问之前构造好的链表.

```java
    final class LinkedValueIterator extends LinkedHashIterator
        implements Iterator<V> {
        public final V next() { return nextNode().value; }
    }
```

接着``LinkedValueIterator``继承了``LinkedHashIterator``, 提供了一个``next()``的方法.

> ## 总结

* ``LinkedHashMap``是基于``HashMap``来实现的, 它和``HashMap``不同之处是: 遍历元素的时候的是有序的. 因为它内部维护了一个双向链表. 遍历的顺序可以有两种: 按照结点的插入顺序; 按照元素被访问的顺序.默认是按结点被插入的顺序.

* ``LinkedHashMap``可以用来实现``LRU``缓存.

* 由于``LinkedHashMap``内部维护了一个双向链表, 因此, 在性能上要稍微逊色于``HashMap``.

* 如果对遍历元素的顺序是无要求的话, 应该使用``HashMap``, 反之应该使用``LinkedHashMap``.
