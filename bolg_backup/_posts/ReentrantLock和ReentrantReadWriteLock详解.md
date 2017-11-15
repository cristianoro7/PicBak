---
layout: post
title: ReentrantLock和ReentrantReadWriteLock详解
data: 2017/10/29
subtitle: ""
description: ""
tags:
  - Java并发包#Lock
categories:
  - Java
---

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fkz6p5rdydj30m80b47at.jpg)

``ReentrantLock``和``ReentrantReadWriteLock``是``Java``并发包中提供的锁, 他们都属于可重入锁.但是``ReentrantLock``是一种悲观锁, 它总是假设竞争条件总是会发生, 所以它同一时刻只能有一个线程获得锁, 而``ReentrantReadWriteLock``是属于乐观锁, 它假设竞争条件并不会经常发生, 所以同一时刻能让多个线程执行.

他们的一个共同点是: 都支持公平和非公平性的获取锁.

> ## ``ReentrantLock``

前面分析过``AQS``, 它是并发包中的锁的基本骨架. 所以``ReentrantLock``内部也是基于``AQS``实现的. ``ReentrantLock``内部将获取锁和释放锁的方法都代理给其内部类: ``Sync``, 而``Sync``是继承``AQS``的, 为了支持公平性和非公平性的锁, ``Sync``有两个子类, 分别为``NonfairSync``和``FairSync``, 他们都重写了``tryAcquire(int)``方法来实现自己公平和非公平获取锁的逻辑, 由于释放锁的逻辑都一样, 因此``tryRelease(int)``由``Sync``重写.

既然``ReentrantLock``基于``AQS``实现的, 所以它支持三种方式获取锁:

* ``lock()``: 不支持中断的获取锁

* ``lockInterruptibly()``: 响应中断地获取锁

* ``tryLock(long timeout, TimeUnit unit)``: 响应中断并且支持超时获取锁.

由于三种方式在``AQS``中已经分析过, 所以这里只分析``lock()``. 主要分析公平性和非公平性地获取锁的逻辑

> ### NonfairSync: 非公平锁

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

上面是非公平性获取锁的逻辑: 如果同步状态为0的话, 证明锁还没被获取, 所以利用``CAS``来获取同步状态, 成功的话, 设置当前线程持有锁. 如果同步状态不为0的话, 判断获取锁的线程是否为当前线程, 如果是的话, 更新同步状态, 从这里也可以看出来``ReentrantLock``是可重入锁.

> ### FairSync: 公平锁

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
}
```

公平和非公平的区别是: 公平性意味着哪个线程等待的时间更长, 就应该首先让它获取锁, 也就是获取锁的顺序必须是``FIFO``. 所以每次获取同步状态时, 都会调用``hasQueuedPredecessors()``判断当前节点是不是有前驱节点, 如果有的话, 证明已经有线程在等待, 因此不获取同步状态, 这样就实现了公平性地获取锁.
至于其他逻辑和前面的大致一样.

> ### tryRelease(int releases)

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

公平锁和非公平锁的释放锁的逻辑都是一样的, 由于``ReentrantLock``是可重入的, 因此只有当同步状态为0的时候才需要将当前持有锁的线程设置为``null``, 也就是释放锁.

> ### 小结

``ReentrantLock``内部默认是非公平锁, 因为非公平锁的吞吐量会比公平锁高, 这是由于非公平锁每次获取锁的时候, 不需要理会当前节点是否有前驱节点, 也就是前面有线程在等待, 只要被唤醒了, 并且能够获取同步状态的话, 就可以获取. 而公平锁每次获取锁时, 当前节点必须是头节点, 也就是它的前面没有线程在等待. 在这种情况下, 如果非头结点的线程被唤醒的话或者是刚刚释放锁的线程又立刻获取锁, 是不能获取锁的, 只能再进行多一次调度. 因此, 非公平锁的吞吐量会比公平锁的高.

> ## ReentrantReadWriteLock

与``ReentrantLock``不同,  ``ReentrantReadWriteLock``的伸缩性要强一些, 因为它将读写锁分离. 在进行读操作的时候, 多个线程是可以同时获取锁的, 这样能更大地提高并发度. 下面是``ReentrantReadWriteLock``的特性

* 公平性: 与``ReentrantLock``一样, ``ReentrantReadWriteLock``内部也是支持公平性和非公平性锁, 默认是非公平性的锁.

* 可重入: ``ReentrantReadWriteLock``支持一个线程多次获取锁.

* 读写分离: ``ReentrantReadWriteLock``内部维护两个锁, 一个是读锁, 另外一个是写锁. 在读多写少的情况下, ``ReentrantReadWriteLock``能发挥更大的并发度.  因为``ReentrantReadWriteLock``支持多个读线程同时获取锁. ``ReentrantReadWriteLock``的读写规则: 当一个写线程获得锁后, 其他线程不能获取锁. 当读线程获得锁后, 其他的读线程可以获得锁, 但是写线程不能.

* 锁降级: 当一个线程持有写锁后, 再获取读锁, 然后释放写锁, 这个过程称为一个锁的降级. ``ReentrantReadWriteLock``不支持锁的升级, 因为锁的升级有可能带来竞争条件的问题.

> ### 读写同步状态设计

``ReentrantReadWriteLock``内部也是基于``AQS``实现的, 但是``AQS``内只有一个``int``类型的变量, 那么``ReentrantReadWriteLock``怎么表示两个同步状态呢? 答案是通过位运算来处理.

``int``类型有32位, ``ReentrantReadWriteLock``将高16位作为读状态, 低16位作为写状态. 这样每次通过一定的位运算来获取高16位或者低16位.

> ### WriteLock

写锁是一种互斥锁, 同一个时刻锁只能被一个线程获取.

> #### 非公平性写锁

跟``ReentrantLock``一样, ``ReentrantReadWriteLock``内部支持三种类型的获取, 因此下面只分析一种.

```java
public void lock() {
    sync.acquire(1);
}
```

在``lock()``方法中, 会调用``AQS``的``acquire(int)``, 在``acquire(int)``中又会回调子类重写的模板方法

```java
protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough:
     * 1. If read count nonzero or write count nonzero
     *    and owner is a different thread, fail.
     * 2. If count would saturate, fail. (This can only
     *    happen if count is already nonzero.)
     * 3. Otherwise, this thread is eligible for lock if
     *    it is either a reentrant acquire or
     *    queue policy allows it. If so, update state
     *    and set owner.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c); //获取写的同步状态
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

如果``c != 0``且 ``w(写状态)``为0的话, 证明读状态不为0, 说明此时获得锁的是读线程, 所以会直接返回``false``来表示获取失败. 如果``w != 0``的话, 说明此时写线程拥有锁, 此时如果拥有锁是当前线程的话, 会更新同步状态来支持可重入的特性, 可重入的次数为65535.

如果``c == 0``的话, 说明同步状态还没被获取, 此时应该会先调用``writerShouldBlock()``, 在非公平的写锁的实现为:

```java
final boolean writerShouldBlock() {
    return false; // writers can always barge
}
```

非公平性的写锁默认是返回``false``, 接着利用``CAS``更新同步状态, 如果成功的话, 设置当前线程为执行线程, 否则的话返回``false``, 进入``AQS``中排队.

> #### 非公平性写锁

公平锁和非公平锁获取锁的基本一样, 不一样的话``writerShouldBlock()``的实现不一样.

```java
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

公平锁每次获取锁时, 为了保证公平性, 都需要看看同步队列中是否有比当前线程等待时间更长的线程, 如果有的话, 就不能获取.

> #### 释放锁

``ReentrantReadWriteLock``释放锁的逻辑跟``ReentrantLock``一样.

> ### 读锁

读锁是一种共享锁, 多个读线程可以同时获取锁. 但是当读线程持有锁时, 写线程是不能持有锁的.

> 非公平读锁

```java
protected final int tryAcquireShared(int unused) {
    /*
     * Walkthrough:
     * 1. If write lock held by another thread, fail.
     * 2. Otherwise, this thread is eligible for
     *    lock wrt state, so ask if it should block
     *    because of queue policy. If not, try
     *    to grant by CASing state and updating count.
     *    Note that step does not check for reentrant
     *    acquires, which is postponed to full version
     *    to avoid having to check hold count in
     *    the more typical non-reentrant case.
     * 3. If step 2 fails either because thread
     *    apparently not eligible or CAS fails or count
     *    saturated, chain to version with full retry loop.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
```

首先获取写状态, 如果写状态为不0且持有写锁的线程是当前执行的线程, 说明这是一个锁降级的操作, 因此允许继续获取同步状态.

如果写状态为0且``getExclusiveOwnerThread() != current``为``true``的话, 表示持有写锁的不是当前执行的线程, 所以返回-1表示失败.

在尝试获取同步状态前, 会先调用``readerShouldBlock()``判断是否应该阻塞读线程, 这个方法在公平和非公平锁的实现不一样.

```java
final boolean readerShouldBlock() {
     /* As a heuristic to avoid indefinite writer starvation,
      * block if the thread that momentarily appears to be head
      * of queue, if one exists, is a waiting writer.  This is
      * only a probabilistic effect since a new reader will not
      * block if there is a waiting writer behind other enabled
      * readers that have not yet drained from the queue.
      */
     return apparentlyFirstQueuedIsExclusive();
 }

 final boolean apparentlyFirstQueuedIsExclusive() {
     Node h, s;
     return (h = head) != null &&
         (s = h.next)  != null &&
         !s.isShared()         &&
         s.thread != null;
 }
```

在非公平锁的实现中, 只要同步状态队列中有写线程正在等待的话, 就应该阻塞读线程, 不让其获取同步状态. 这样做是为了防止写线程出现饥饿现象.

> 公平读锁

公平读锁和非公平读锁的实现也是基本一样, 不一样的就是``readerShouldBlock()``实现不同.

```java
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

为了保持公平性, 每次都是让同步队列中的队头线程获取锁, 因为队头线程等待的时间最长.

> 释放读锁

忽略一些其他的操作的话, 读锁的释放逻辑跟写锁的差不多, 都是``CAS``来更新同步状态, 不同的是, 读锁是共享锁, 必须通过循环``CAS``来保证线程安全.

> ### 小结

``ReentrantReadWriteLock``内部维护两个锁, 一个是写锁, 另外一个是读锁. 通过将读写锁分离, 在读多写少的情况下, 更够提高程序的并发程度, 因此, ``ReentrantReadWriteLock``的伸缩性要好于``ReentrantLock``. 在读多写少的情况下, 应该使用``ReentrantReadWriteLock``更为合适.

``ReentrantReadWriteLock``只支持锁降级, 不支持锁升级. 因为锁升级有可能会出现条件竞争. 由于读锁是可以被多个线程持有的, 如果进行锁升级的话, 当写线程改变共享变量的状态时, 其他读线程有可能感知不到.
