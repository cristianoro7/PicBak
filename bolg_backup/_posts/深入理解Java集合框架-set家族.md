---
layout: post
title: 深入理解Java集合框架-set家族
data: 2017/5/20
subtitle: ""
description: ""
thumbnail: "/img/collection.jpeg"
tags:
  - Java集合框架
categories:
  - Java
---

``Set``集合的最大特点是能够保证内部的元素唯一性. 这种特性是建立在``Map``的基础上的. 换句话说: ``Set``通过组合的模式, 在``Map``的基础上扩展了一些特性. 由于``Set``是建立在``Map``的基础上的. 如果理解了``Map``的话, ``Set``会很好理解. 接下来我们来看看``Set``家族的UML

> ## UML

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fizhykmqgzj30x10hwq3y.jpg)

``Set``接口对应``Map``接口, 定义了``Set``的一些方法. ``AbstractSet``对应``AbstractMap``, 为``Set``家族提供了一些默认的实现. ``HashSet``通过组合``HashMap``并且利用``HashMap``的Key值的唯一性, 来保证内部元素的唯一性. ``LinkedHashSet``通过组合``LinkedHashMap``, 使得Set内部的元素是有序的. 同种道理, ``TreeSet``也是基于``TreeMap``来实现的.

> ## Set家族

* ``HashSet``
* ``LinkedHashSet``
* ``TreeSet``

> ### HashSet

```java
    private transient HashMap<E,Object> map;

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();
```

前面说过``Set``家族都是通过组合``Map``家族来实现的. 从上面的字段, 可以看出``HashSet``是基于``HashMap``来实现的. 而``PRESENT``则是一个虚假的值.

```java
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
```

当向``HashSet``中添加元素时, 将元素作为map的key, 而value则是用一个虚假的值``PRESENT``. 由于``HashMap``中的Key是唯一的. 而``HashSet``中的元素是作为Key储存在``HashMap``中的.这样就保证了``HashSet``元素中没有重复的值.

> ### LinkedHashSet

``LinkedHashSet``继承于``HashSet``, 它对应``Map``家族的``LinkedHashMap``. 但是当你查看``LinkedHashSet``的时候, 可能会发现并没有``LinkedHashMap``.

```java
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
```

上面这个构造函数是``HashSet``为``LinkedHashSet``预留的.

```java
    public Iterator<E> iterator() {
        return map.keySet().iterator();
    }
```

```LinkedHashSet``中还提供了一个迭代器接口, 迭代器遍历时是有序的遍历. 因为该迭代器是``LinkedHashMap``的一个实现.

> ### TreeSet

``TreeSet``对应于``TreeMap``, ``TreeSet``的实现思路跟前面两个``Set``差不多.. 这里不分析.

> ## 最后

到这里...不知不觉已经分析完了集合框架. 但是并没有包含并发的集合工具. 接下来准备看完并发后再来梳理并发集合工具.
