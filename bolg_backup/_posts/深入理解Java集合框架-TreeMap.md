---
layout: post
title: 深入理解Java集合框架-TreeMap
data: 2017/5/20
subtitle: ""
description: ""
thumbnail: "/img/collection.jpeg"
tags:
  - Java集合框架
categories:
  - Java
---

这次要介绍的``Map``跟之前介绍的``Map``有点不一样. 之前的``Map``, 例如: ``HashMap``, ``LinkedHashMap``都是基于散列技术. 而这次要介绍的``TreeMap``则不同, ``TreeMap``是基于一种叫``红黑树``的数据结构. 接下来, 我们先看看``TreeMap``的UML图片

> ## UML

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fizax4c2efj30o60c2dgl.jpg)

跟之前介绍过的``Map``一样, ``TreeMap``实现了``Cloneable``和``Serializable``, 因此它支持克隆(浅克隆)和序列化.

``SortMap``这个接口提供了一些有序的视图, 比如: key有序视图. 实现该接口的数据结构表明其遍历Key或者Value的时候,是有序的. ``NavigableMap``继承``SortMap``接口, 并提供了更为丰富的视图操作.

> ## TreeMap特性

``TreeMap``是基于红黑树来实现的. 红黑树是一种自平衡的二叉查找树, 插入, 查找, 删除等操作的时间复杂度都是``O(logn)``. 由于红黑树也是一种二叉查找树, 因此, 遍历``Treemap``是有序的.

``TreeMap``的结点的默认顺序是``Key``的自然顺序(``Key``必须实现``Comparator``). 当然, ``TreeMap``内部也支持外部提供``Comparator``来指定结点的排列顺序.

``TreeMap``的Key是不支持null的, 而Value则支持null.

在了解了``TreeMap``一些特性后, 我们接着来分析``TreeMap``常用的操作和它的迭代器.

> ## 解析

* 常用操作
* 迭代器

> ### 常用操作

> #### 结点

```java
    static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left; //左孩子
        Entry<K,V> right; //右孩子
        Entry<K,V> parent; //父结点
        boolean color = BLACK;
}
```

``Entry``是红黑树的结点类型, 除了K和V外, 还包含了左孩子, 右孩子和父结点.

> #### 字段

```java
  /**
     * The comparator used to maintain order in this tree map, or
     * null if it uses the natural ordering of its keys.
     *
     * @serial
     */
    private final Comparator<? super K> comparator; //比较器, 可以通过构造函数赋值

    private transient Entry<K,V> root; //红黑树的根结点

    /**
     * The number of entries in the tree
     */
    private transient int size = 0; //红黑树中的结点数

    /**
     * The number of structural modifications to the tree.
     */
    private transient int modCount = 0; //用于实现fast-fail机制
```

> #### put(K, V)

```java
    public V put(K key, V value) {
        Entry<K,V> t = root;
        if (t == null) { //如果没有根结点的话, 证明树是空的
            compare(key, key); // type (and possibly null) check

            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        if (cpr != null) { //有指定比较器的情况
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        else { //采用默认的Key自然顺序插入结点
            if (key == null)
                throw new NullPointerException();
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key); //比较
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null); //找到合适的插入位置
        }
        Entry<K,V> e = new Entry<>(key, value, parent);
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
        fixAfterInsertion(e); //恢复红黑树的特性
        size++;
        modCount++;
        return null;
    }
```

插入操作分为两种情况:

* 根结点为空(红黑树为空): 直接将新结点赋值给根结点
* 根结点不为空:如果有指定比较器的话, 使用比较器来新结点的父节点. 没有的话, 根据Key的自然顺序来找新结点的父结点.

插入结点后, 可能会违背红黑树的特性, 因此, 每次插入后, 都需要进行一些操作来恢复红黑树的特性.

> #### get(Object)

```java
    public V get(Object key) {
        Entry<K,V> p = getEntry(key);
        return (p==null ? null : p.value);
    }
```

``get``操作是获取key对应的value. 方法中主要调用了``getEntry(key)``来获取结点.

```java
    final Entry<K,V> getEntry(Object key) {
        // Offload comparator-based version for sake of performance
        if (comparator != null)
            return getEntryUsingComparator(key); //如果比较器不为空的话, 使用指定的比较器来比较
        if (key == null)
            throw new NullPointerException(); //不支持Key为null的操作
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        Entry<K,V> p = root;
        while (p != null) { //循环比较查找, 如果比较结果是<0的话, 向左子树查找, >0的话向右子树查找, 知道等于.
            int cmp = k.compareTo(p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
        return null;
    }
```

根据二叉查找树的特性, 先比较结点, 如果大于0, 则向右子树查找, 如果小于0, 则左子树查找, 直到找到等于的结点.
如果不存在的话, 返回 null;

> #### remove

```java
    public V remove(Object key) {
        Entry<K,V> p = getEntry(key); //获取结点
        if (p == null)
            return null;

        V oldValue = p.value;
        deleteEntry(p); //从红黑树中删除结点, 并做恢复红黑树特性的操作
        return oldValue;
    }
```

``remove``操作分两步: 获取要删除的结点, 找得到的话, 从红黑树中删除, 并且做恢复红黑树的操作, 最后返回被删除的结点; 如果找不到结点, 返回null.

> #### firstKey和firstEntry

```java
    public K firstKey() {
        return key(getFirstEntry());
    }
```

``firstKey``是获取最小的Key值. 主要实现是在``getFirstEntry()``中, key(Entry)只是提取Entry中的key.

```java
    final Entry<K,V> getFirstEntry() {
        Entry<K,V> p = root;
        if (p != null)
            while (p.left != null) //循环遍历左子树
                p = p.left;
        return p;
    }
```

根据二叉查找树的特性, 想要获取树的最小值, 只要一直遍历左子树, 就能得到.

firstEntry的实现跟firstKey差不多.  不同的是, firstEntry得到Entry后, 会被包装成一个不可改变的Entry

```java
    static <K,V> Map.Entry<K,V> exportEntry(TreeMap.Entry<K,V> e) {
        return (e == null) ? null :
            new AbstractMap.SimpleImmutableEntry<>(e);
    }
```

``Entry``在这个方法中被包装成了一个不可改变Value的Entry, 我们来看看为什么不改变

```java
        public V setValue(V value) {
            throw new UnsupportedOperationException();
        }
```

上面的``setValue``是SimpleImmutableEntry中的方法, 只要调用了, 都会抛出一个异常, 因此不支持改变Value.

与``firstKey``和``firstValue``对应的有``lastKey``和``lastValue``

```java
    public K lastKey() {
        return key(getLastEntry());
    }

    /**
     * @since 1.6
     */
    public Map.Entry<K,V> lastEntry() {
        return exportEntry(getLastEntry());
    }
```

操作的思想都差不多, lastXX是获取Key最大的结点. 因此, 在查找时会循环查找右子树.

> #### NavigableMap接口

接着, 我们来看看``NavigableMap``接口中的一些实现

```java
Map.Entry<K,V> lowerEntry(K key); //返回小于Key的最大结点

K lowerKey(K key);

Map.Entry<K,V> floorEntry(K key); //返回小于等于Key的最大结点

K floorKey(K key);

Map.Entry<K,V> ceilingEntry(K key); //返回大于等于该Key的最小结点

K ceilingKey(K key);

Map.Entry<K,V> higherEntry(K key); //返回大于该Key的最小结点

K higherKey(K key);

Map.Entry<K,V> pollFirstEntry(); //删除最小的结点

Map.Entry<K,V> pollLastEntry(); //删除最大的结点
```

上面的接口是获取红黑树中, 指定的结点. 这些接口的实现的思路都差不多, 这里只分析``lowerEntry``

```java
    public Map.Entry<K,V> lowerEntry(K key) {
        return exportEntry(getLowerEntry(key));
    }
```

主要的实现在``getLowerEntry``中

```java
    final Entry<K,V> getLowerEntry(K key) {
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = compare(key, p.key);
            if (cmp > 0) { //如果大于0,
                if (p.right != null)
                    p = p.right;  //查找右子树,
                else
                    return p; //返回小于Key的最大值
            } else { //如果<=0
                if (p.left != null) {
                    p = p.left; //查找左子树
                } else { //回退父结点查找
                    Entry<K,V> parent = p.parent;
                    Entry<K,V> ch = p;
                    while (parent != null && ch == parent.left) {
                        ch = parent;
                        parent = parent.parent;
                    }
                    return parent;
                }
            }
        }
        return null;
    }
```

上面都是在二叉查找树中查找符合一定条件的结点的算法.没什么好说的.

> ### 迭代器

```java
    abstract class PrivateEntryIterator<T> implements Iterator<T> {
        Entry<K,V> next;
        Entry<K,V> lastReturned;
        int expectedModCount;

        PrivateEntryIterator(Entry<K,V> first) {
            expectedModCount = modCount;
            lastReturned = null;
            next = first;
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Entry<K,V> nextEntry() {
            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            next = successor(e);
            lastReturned = e;
            return e;
        }

        final Entry<K,V> prevEntry() {
            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            next = predecessor(e);
            lastReturned = e;
            return e;
        }

        public void remove() {
            if (lastReturned == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            // deleted entries are replaced by their successors
            if (lastReturned.left != null && lastReturned.right != null)
                next = lastReturned;
            deleteEntry(lastReturned);
            expectedModCount = modCount;
            lastReturned = null;
        }
    }
```

``PrivateEntryIterator``是``TreeMap``的基类迭代器, 提供了默认的一些操作.

``nextEntry()``是获取下一个结点.主要获取的实现是在``successor``.

```java
    static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
        if (t == null)
            return null;
        else if (t.right != null) { //如果右子树不为空的话,
            Entry<K,V> p = t.right;
            while (p.left != null) //循环查找左子树, 找到大于t的最小结点
                p = p.left;
            return p;
        } else { //为空的话, 回退到父结点, 查找大于t的最小结点
            Entry<K,V> p = t.parent;
            Entry<K,V> ch = t;
            while (p != null && ch == p.right) {
                ch = p;
                p = p.parent;
            }
            return p;
        }
    }
```

* ``successor(Entry)``的作用是, 在红黑树中找出比给定结点大的最小结点. ``PrivateEntryIterator``构造函数传入的是红黑树中最小的结点. 所以循环调用``successor``的话, 得到的结点是按Key排序的, 换句话说: 遍历调用nextEntry函数, 得到的Entry顺序是按Key升序的.

```java
        final Entry<K,V> prevEntry() {
            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            next = predecessor(e);
            lastReturned = e;
            return e;
        }
```

同理, ``preEntry``是得到前一个结点.

接着我们来看看``PrivateEntryIterator``的子类.

```java
   final class ValueIterator extends PrivateEntryIterator<V> {
        ValueIterator(Entry<K,V> first) {
            super(first);
        }
        public V next() {
            return nextEntry().value;
        }
    }

    final class KeyIterator extends PrivateEntryIterator<K> {
        KeyIterator(Entry<K,V> first) {
            super(first);
        }
        public K next() {
            return nextEntry().key;
        }
    }
```

``KeyIterator`` 继承了``PrivateEntryIterator``, 并提供了next()方法, 用于获取下一个key. ``ValueIterator``也是同样的操作. 我们再来看看, 构造函数传入的是什么结点?

```java
    Iterator<K> keyIterator() {
        return new KeyIterator(getFirstEntry());
    }
```

前面分析过, ``getFirstEntry()``是获取key最小的一个结点. 所以 循环调用``KeyIterator``的``next()``, 得到的是按Key升序的序列.

那有没有按Key降序的迭代器? 有

```java
final class DescendingKeyIterator extends PrivateEntryIterator<K> {
        DescendingKeyIterator(Entry<K,V> first) {
            super(first);
        }
        public K next() {
            return prevEntry().key;
        }
        public void remove() {
            if (lastReturned == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            deleteEntry(lastReturned);
            lastReturned = null;
            expectedModCount = modCount;
        }
    }
```

``DescendingKeyIterator``为反向Key的迭代器, 使用该迭代器遍历的时候, 得到的序列是Key的降序序列.

```java
    Iterator<K> descendingKeyIterator() {
        return new DescendingKeyIterator(getLastEntry());
    }
```

每次传入``DescendingKeyIterator``构造函数的Entry都是最大的结点.

## 最后

``TreeMap``就讲到这里了. ``Map``家族大概在这里就讲完了. 下篇会将``Set``家族, ``Set``集合是基于``Map``来操作的, 如果理解``Map``集合后, ``Set``会很好理解.
