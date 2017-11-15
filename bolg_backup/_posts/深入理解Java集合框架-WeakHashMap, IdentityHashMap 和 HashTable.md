---
layout: post
title: 深入理解Java集合框架-WeakHashMap, IdentityHashMap 和 HashTable
data: 2017/5/20
subtitle: ""
description: ""
thumbnail: "/img/collection.jpeg"
tags:
  - Java集合框架
categories:
  - Java
---

> ## WeakHashMap

``WeakHashMap``总体实现和``HashMap``差不多, 不同的时, ``WeakHashMap``中的Key是弱引用类型, ``WeakHashMap``内部的Key是会被自动回收的. 另外需要关注的是, ``WeakHashMap``并没有向``HashMap``那样, 在1.8做了优化.

> ### 弱引用Key

```java
    private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
        V value;
        final int hash;
        Entry<K,V> next;

        /**
         * Creates new entry.
         */
        Entry(Object key, V value,
              ReferenceQueue<Object> queue,
              int hash, Entry<K,V> next) {
            super(key, queue);
            this.value = value;
            this.hash  = hash;
            this.next  = next;
        }

        @SuppressWarnings("unchecked")
        public K getKey() {
            return (K) WeakHashMap.unmaskNull(get());
        }

        public V getValue() {
            return value;
        }

        public V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            K k1 = getKey();
            Object k2 = e.getKey();
            if (k1 == k2 || (k1 != null && k1.equals(k2))) {
                V v1 = getValue();
                Object v2 = e.getValue();
                if (v1 == v2 || (v1 != null && v1.equals(v2)))
                    return true;
            }
            return false;
        }

        public int hashCode() {
            K k = getKey();
            V v = getValue();
            return Objects.hashCode(k) ^ Objects.hashCode(v);
        }

        public String toString() {
            return getKey() + "=" + getValue();
        }
    }
```

``WeakHashMap``内部的结点是继承弱引用类型, 并且指定了一个``ReferenceQueue<Object>``, 当Key被GC回收时, Key对应的对象会被添加到``ReferenceQueue<Object>``这个队列中. 这个队列是定义在``WeakHashMap``内部的

```java
    /**
     * Reference queue for cleared WeakEntries
     */
    private final ReferenceQueue<Object> queue = new ReferenceQueue<>();
```

``WeakHashMap``的实现原理跟``HashMap``是差不多的. 不过``HashMap``在1.8后, 做了很多优化, 但是``WeakHashMap``并没有跟随``HashMap``进行优化.

由于``WeakHashMap``和``HashMap``差不多, 我们只分析重要的实现方法.

``WeakHashMap``的增删查改, 都会做一个同步操作, 什么是同步操作? 因为``WeakHashMap``的Key是弱引用类型, Key值会随时被GC回收. 虽然Key被回收了,但是对应的Value还是没有被回收的. 所以, 同步操作就是移除被回收Key对应的Value.

```java
    /**
     * Expunges stale entries from the table.
     */
    private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) { //判断queue队列中是否有被回收的Key, 有的话, 需要移除对应Value
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
```

这个方法是同步操作的方法, ``WeakHashMap``增删改查时, 都会调用的方法. 原理是这样的: 每当弱引用类型Key被GC回收时, 由于``WeakHashMap``内部指定了``ReferenceQueue<Object>``, 因此, 被移除的Key会被添加到该队列. 这个方法就是调用队列的``poll()``方法, 获得被移除的Key, 然后利用该Key, 移除在``WeakHashMap``内部对应的Value.

> ### 总结

* ``WeakHashMap``和``HashMap``一样, Key和Value都支持null.

* ``WeakHashMap``内部的继承WeakReference, 将Key指定为弱引用类型, 这样Key会被自动回收. 虽然Key会被自动回收, 但是Value不会被自动回收. 因此, ``WeakHashMap``内部每次增删改查时, 都会做同步操作.

> ## IdentityHashMap

``IdentityHashMap``继承于``AbstractMap``,并实现了``Map``接口. 总体来说, ``IdentityHashMap``跟``HashMap``差别还是很大的. 虽然继承结构相同, 但是实现的思想却是截然不同.

``IdentityHashMap``内部并没有结点, 它的Key和Vaule都是储存在数组内的. 因此, ``IdentityHashMap``解决的冲突是开放地址法.

上面说到``IdentityHashMap``的Key和Value都是存在数组内的. Key和Value总是连续的存放.

另外值得一提的是: ``IdentityHashMap``是利用 ``==``来比较Key的相等性.

```java
private static final int DEFAULT_CAPACITY = 32;

public IdentityHashMap() {
    init(DEFAULT_CAPACITY);
}

private void init(int initCapacity) {
    // assert (initCapacity & -initCapacity) == initCapacity; // power of 2
    // assert initCapacity >= MINIMUM_CAPACITY;
    // assert initCapacity <= MAXIMUM_CAPACITY;

    table = new Object[2 * initCapacity];
}
```

``IdentityHashMap``默认的容量是64. 长度跟``HashMap``一样, 都是2的次幂.还是跟之前的套路一样, 只分析一些重要的实现.

```java
    public V put(K key, V value) {
        final Object k = maskNull(key);

        retryAfterResize: for (;;) {
            final Object[] tab = table;
            final int len = tab.length;
            int i = hash(k, len); //获得数组的index

            for (Object item; (item = tab[i]) != null; //有冲的话, 查找下一个index
                 i = nextKeyIndex(i, len)) {
                if (item == k) {
                    @SuppressWarnings("unchecked")
                        V oldValue = (V) tab[i + 1];
                    tab[i + 1] = value;
                    return oldValue;
                }
            }

            final int s = size + 1;
            // Use optimized form of 3 * s.
            // Next capacity is len, 2 * current capacity.
            if (s + (s << 1) > len && resize(len)) //当储存的元素大于 容量的3分之一的话, 扩容
                continue retryAfterResize;

            modCount++;
            tab[i] = k;
            tab[i + 1] = value; //Value始终是保存在Key的下个index
            size = s;
            return null;
        }
    }
```

通过``hash``函数, 获得index, 如果index对应已经存有元素的话, 会调用``nextKeyIndex(int, int)``.

```java
    private static int nextKeyIndex(int i, int len) {
        return (i + 2 < len ? i + 2 : 0);
    }
```

``IdentityHashMap``是使用开放地址法解决冲突. 当发生冲突时, 它会寻找下个位置, 而下个位置是 i + 2. 之所以是i + 2,而不是i + 1的原因是 Key和Value总是连续的储存的, 因此寻找下个位置时, 需要跳多一个位置.如下图

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fip01fuwt1j30er06bmx6.jpg)

如果index没有储存元素的话, 会先检查内部储存元素数量是不是大于容量的1/3, 是的话会进行扩容操作.每次扩容都是扩为原来的2倍.

> ## HashTable

``HashTable``是一个遗留类, 已经被``HashMap``替代了. 从功能上来说, 跟HashMap差不多. 主要比较它跟``HashMap``的异同.

### null

``HashMap``是允许K或者V为null的, 但是``HashTable``不允许K或者V为null

### 线程安全

``HashMap``是线程不安全的, 而``HashTable``是线程安全的, 因为内部的方法都是加了锁. 但是在并发的环境, 不建议使用``HashTable``, 应该使用``ConcurrentHashMap<K,V>``. ``ConcurrentHashMap<K,V>``是使用分段锁来控制同步, 显然性能上要比``HashTable``好.

### 容量

``HashMap``的容量为2的n次幂, 而``HashTable``为素数.
