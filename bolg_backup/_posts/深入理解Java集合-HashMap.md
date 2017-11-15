---
layout: post
title: 深入理解Java集合-HashMap
data: 2017/5/20
subtitle: ""
description: ""
thumbnail: "/img/collection.jpeg"
tags:
  - Java集合框架
categories:
  - Java
---

前两篇文章分别介绍了``ArrayList``和``LinkedList``, 这次我们来分析另外一个key-value键值对的映射集合-HashMap.按照前面的习惯,我们先来看看``HashMap``的UML

> ## UML

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fimu0do73tj30ii0cv3z0.jpg)

``HashMap``实现了``Cloneable``, ``Serializable``, 所以``HashMap``支持克隆(浅克隆)和序列化. ``Map``接口提供了一系列接口和三个视图. ``AbstractMap``则是实现了``Map``接口, 并且实现了``Map``接口中的一些方法, 换句话说``AbstractMap``是``Map``家族里面的一个基本骨架, 具体的子类根据需要重写``AbstractMap``中的方法即可.

> ## 回顾Hash表

学过数据结构的都应该清楚, Hash表是将一个key的hash值取模后映射到一个数组中的特定位置. 但是, 随着Hash表中的键值对的增多, 会出现冲突. 所谓的冲突是两个key的hash值取模后得到的数定位到了数组中的相同位置.对于冲突这种情况, 常用的解决方法有:开放地址法和链地址法.

> ## 解析

回顾完了Hash表的基础知识后, 我们来讲讲``HashMap``的实现方式.

总体来说, ``HashMap``的实现为: 数组+链表+红黑树. 当出现冲突时, ``HashMap``是使用链地址法来解决冲突.但是如果冲突越来越多, 链表就会变得越来越长.这样导致的结果是:原本访问一个Key对应的value的时间复杂度会从O(1)退化为O(n).因此, ``HashMap``使用一种名为红黑树的数据结构来解决时间复杂度退化的这个问题.红黑树查找key值对应的value的时间复杂度为O(log n)(优于O(n)).

下面, 先来看看``HashMap``中的字段, 我们将重点介绍两个字段. 这两个字段影响着``HashMap``的性能.

```java
/**
 * The table, initialized on first use, and resized as
 * necessary. When allocated, length is always a power of two.
 * (We also tolerate length zero in some operations to allow
 * bootstrapping mechanics that are currently not needed.)
 */
transient Node<K,V>[] table; //内部的数组

/**
 * Holds cached entrySet(). Note that AbstractMap fields are used
 * for keySet() and values().
 */
transient Set<Map.Entry<K,V>> entrySet; //返回包含key-value的视图

/**
 * The number of key-value mappings contained in this map.
 */
transient int size; //内部储存的key-value的个数

/**
 * The number of times this HashMap has been structurally modified
 * Structural modifications are those that change the number of mappings in
 * the HashMap or otherwise modify its internal structure (e.g.,
 * rehash).  This field is used to make iterators on Collection-views of
 * the HashMap fail-fast.  (See ConcurrentModificationException).
 */
transient int modCount; //记录内部数据结构发生变化的次数, 主要用来实现fail-fast

/**
 * The next size value at which to resize (capacity * load factor).
 *
 * @serial
 */
// (The javadoc description is true upon serialization.
// Additionally, if the table array has not been allocated, this
// field holds the initial array capacity, or zero signifying
// DEFAULT_INITIAL_CAPACITY.)
int threshold;

/**
 * The load factor for the hash table.
 *
 * @serial
 */
final float loadFactor;
```

loadFactor是负载因子, 这个字段表示``HashMap``内部容量满载的一个界限. threshold为扩容的界限,它的计算公式为: threshold = capacity * loadFactor.当size > threshold时, ``HashMap``内部就会扩容.

``loadFactor``内部默认实现为0.75. 这个数值是时间和空间的一个折中值.``loadFactor``如果设置为比较大, 也就是``threshold``会比较大, 那么空间的利用率就会变大, 但是空间利用率变大的代价是查找速度变慢了, 因因为冲突率会提高. 如果``loadFactor``设置为比较小的话, ``threshold``会比较小, 虽然冲突率会变小, 因为会频繁扩容.但这也一定程度上浪费空间内存.

> ### 构造函数

``HashMap``的构造函数有几个重载, 我们既可以使用无参的构造方法, 使用内部默认指定的``capacity``和``loadFactor``.也可以调用指定这两个参数的构造函数.

如果没有特殊情况.我们一般调用无参的构造函数.它会帮我们指定默认的``loadFactor``.

> ### 添加元素

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

``put``函数是往``HashMap``中添加元素, 在分析添加元素的实现之前, 我们先来看看``hash(key)``函数,得到的hash值.

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
> #### hash函数

``hash``函数的实现思路: 将key的hashCode的高16位和低16位作异或操作.至于为什么是要这样操作?

一般而言, 哈希表的长度设置为一个素数的话, 发生冲突的次数会比较少.但是为了优化``HashMap``的扩容和``rehash``操作, ``HashMap``的长度被设置为总是为2的次方(不是一个素数).既然被设置为一个合数, 那么冲突的次数肯定会还比较多. ``hash``函数通过再次散列来减少冲突率. 但 ``HashMap``内部的hash函数没有被设置得很复杂,``HashMap``内部的``hash``函数只是简单的高低16位进行异或操作. 面对冲突的情况, ``HashMap``内部的优化有红黑树,并且本来key的hashCode已经挺分散了. 从质量和系统消耗的角度出发, 没有必要设置复杂的``hash``函数.

分析完了``hash(int key)``函数, 我们回到``put(K key)``, 来看看真正的添加操作.

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0) //检查HashMap是否为空
        n = (tab = resize()).length; //空的话, 先进行扩容
    if ((p = tab[i = (n - 1) & hash]) == null) //判断HashMap中有没有存该Key
        tab[i] = newNode(hash, key, value, null); //没有存该值, 直接赋值.
    else { //HashMap不为空的情况.
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k)))) //判断HashMap有没存该Key
            e = p; //没有的话直接赋值
        else if (p instanceof TreeNode) //判断这个结点是不是红黑树结点
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value); 是的话, 插入到红黑树结构中
        else { //插入到链表结构中
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash); //如果链表的结点个数为8的话, 将链表转化为红黑树, 这样提高查找的速度
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
            afterNodeAccess(e); //hock函数, 用于LinkHashMap中
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold) //如果size > threshold的话, 需要进行扩容
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

上面注释已经将函数解析得挺清楚了, 下面主要介绍这个``hash & (n - 1)``操作. ``hash & (n - 1)``其实等价于取模操作, 但是比普通的 % 操作高效, 因为``hash & (n - 1)``运用的位运算.

一个数和2的n次方进行去模操作, 余数是由这个数的低n决定,所以``hash & (n - 1)``就是取得hash的底n位,也就是余数.如下图:

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fimz01cbwzj30d60dm74h.jpg)

前面说过``HashMap``内部的长度总是为2的次方, ``n``为``HashMap``的长度,所以跟上面的例子原理一样.

``afterNodeAccess(e);``和``afterNodeInsertion(evict);``是hock函数, 是预留给``LinkedHashMap``使用的.

链表的构造使用的是头插法, 后插入的会在链表头.

如果一直向``HashMap``中添加元素的话, 其内部会扩容, 扩容通俗地说就是用更大的数组来代替之前的小数组以装下更多的元素.1.8的``HashMap``内部的扩容机制做了一些优化, 接下来, 我们来详细分析其中的优化.

> #### 扩容机制

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // 扩大为原来的两倍
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

``resize()``函数为``HashMap``的扩容机制,扩容的情况有两种, 一种的初始化默认的容量, 另外一种是扩大为原来的两倍.

扩大容量后, 还需要把原来的数据搬到新的数组中.

```java
if (oldTab != null) {
    for (int j = 0; j < oldCap; ++j) {
        Node<K,V> e;
        if ((e = oldTab[j]) != null) {
            oldTab[j] = null;
            if (e.next == null)
                newTab[e.hash & (newCap - 1)] = e; //定位到新的位置, 并且赋值
            else if (e instanceof TreeNode)
                ((TreeNode<K,V>)e).split(this, newTab, j, oldCap); //如果结点是红黑树的结点的话, 会进行修剪树的操作
            else { // preserve order 头结点后面有元素,且为链表的结构
              //重新定位链表的策略: 定义两条链表, 构造完链条链表后, 再将他们的头结点定位到数组对应的index
                Node<K,V> loHead = null, loTail = null; //留在原来的位置的链表
                Node<K,V> hiHead = null, hiTail = null; //移动在原来位置+2的n次方位置的链表
                Node<K,V> next;
                do {
                    next = e.next;
                    if ((e.hash & oldCap) == 0) { //新增位为0
                        if (loTail == null)
                            loHead = e;
                        else
                            loTail.next = e;
                        loTail = e;
                    }
                    else { //新增位为1
                        if (hiTail == null)
                            hiHead = e;
                        else
                            hiTail.next = e;
                        hiTail = e;
                    }
                } while ((e = next) != null);
                if (loTail != null) {
                    loTail.next = null;
                    newTab[j] = loHead; //将链表定位到原来的位置
                }
                if (hiTail != null) {
                    hiTail.next = null;
                    newTab[j + oldCap] = hiHead; //将链表定位到原来的位置 + 2的n次方
                }
            }
        }
    }
}
```

结合代码中的注释, 我们来解析这个扩容机制, 首先遍历原来的数组, 然后拿到每个位置的头结点, 再判断头结点后面有没有元素, 没有的话, 直接用``&``运算定位到新的位置.如果有的话, 先判断这个结点是不是红黑树的结点, 是的话做修剪树的操作.不是的话, 说明后面的结点的结构是链表.

前面说过, ``HashMap``的每次扩容为原来的2倍, 这样的话, 原来的元素不是定位到原地, 就是在原地移动2的次幂.我们结合扩容前和扩容后的图来解释:

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fin22vrzufj30nb0c50tb.jpg)

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fin27c3g43j30o10bpmxq.jpg)

扩容后,决定位置的位数多了一位, 拿16扩容到32来说, 16的时候, 决定位置是由结果的后4位来决定, 如果扩容为32位后, 决定位置是由结果的后5位.如果新增的那一位为0的话,表示新的位置是原来的位置, 如果为1的话, 新的位置为原来的位置 + 原来的容量(原来的容量为2的n次方).

所以代码中``e.hash & oldCap``就是为了取得新增的那一位, 如果为0的话, 说明新的位置为原来的位置, 如果为1的话,则需要移动2的n次幂(也就是 原来的位置 + 旧的容量).

上面重新组装链表的时候, 思路是这样的: 因为新的位置不是在原来的位置, 就是需要在原来的位置上移动2的次幂.所以, 定义两条链表, 一条表示新位置是原来的位置的链表(简称L1), 另外一条表示新的位置在原来的位置移动2的次幂(简称L2). 接着会先遍历原来的链表, 再进行``e.hash & oldCap``, 如果为0的话, 将该结点插入到L1, 如果为1的话, 插入到L2. 插入的方法为尾插法, 这样重构后的链表不会乱序(1.7之前的版本是会乱序的). 最后,将新构造的两条链表定位到对应的位置,也就是下面的代码:

```java
if (loTail != null) {
    loTail.next = null;
    newTab[j] = loHead; //将链表定位到原来的位置
}
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead; //将链表定位到原来的位置 + 2的n次方
}
```

1.8的扩容机制的优化主要是不需要再重新计算hash值, 只是做个``&``操作就行了, 并且重构后的链表不会乱序. 这种优化的一个重要的前提是容量为2的n次幂. 因此 ``HashMap``内部的容量为什么不定义为素数而是定义为2的n次幂. 这样做是为了减少扩容时的``rehash``操作.


到这里, 我们已经把``HashMap``内部的精华都分析完了, 其他操作都挺简单的, 也没什么好讲的.最后, 做个总结

> ## 总结

* ``hash``函数
* 扩容机制
* 冲突优化
* 其他

> ### ``hash``函数

一般而言, 哈希表的长度设置为一个素数的话, 发生冲突的次数会比较少.但是为了优化``HashMap``的扩容和``rehash``操作, ``HashMap``的长度被设置为总是为2的次方(不是一个素数).既然被设置为一个合数, 那么冲突的次数肯定会还比较多. ``hash``函数通过再次散列来减少冲突率. 但 ``HashMap``内部的hash函数没有被设置得很复杂,``HashMap``内部的``hash``函数只是简单的高低16位进行异或操作. 面对冲突的情况, ``HashMap``内部的优化有红黑树,并且本来key的hashCode已经挺分散了. 从质量和系统消耗的角度出发, 没有必要设置复杂的``hash``函数.

> ### 扩容机制

* ``HashMap``的扩容后的长度总是2的次幂, 这个设置主要是为了优化扩容机制. ``HashMap``内部扩容时,不需要重新计算hash值,并且链表的顺序还是保持原来的顺序.

* 频繁进行扩容会一定程度上影响``HashMap``性能.因此,如果预先知道需要储存的数据有很多的话, 可以直接设置一个较大的容量来减少扩容的次数.

> ### 冲突优化

对于hash冲突的情况, 如果冲突的链表结点个数大于8的话, ``HashMap``会将链表转化为红黑树的结构, 将查找的复杂度从O(N)优化为O(logN).

> ### 其他

* ``HashMap``既支持``Key``为``null``, 也支持``value``为``null``.

* 遍历``HashMap``时, 顺序是不保证的, 如果需要有序的遍历, 应该使用``LinkedHashMap``
