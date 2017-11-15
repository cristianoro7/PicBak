---
layout: post
title: AbstractQueuedSynchronizer剖析
data: 2017/10/23
subtitle: ""
description: ""
tags:
  - Java并发包#AQS
categories:
  - Java
---

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fktbn58ux2j30m80b47l5.jpg)

# AbstractQueuedSynchronizer剖析

在介绍AbstractQueuedSynchronizer(下面称AQS)前, 我们先来看看一个不安全的锁, 然后引出构建安全锁需要处理哪些情况.

```c
typedef struct lock_t {
    int flag;
} lock_t;

void init(lock_t *mutex) {
    mutex->flag = 0; //0表示锁空闲, 1表示锁被占有
}

void lock(lock_t *mutex) {
    while(mutex->flag == 1) {
        //自旋等待
    }
    mutex->flag = 1; //获得锁
}

void unlock(lock_t *mutex) {
    mutex->flag = 0;
}
```

``flag``字段是一个状态字段, 0表示锁空闲, 1表示锁被占有. 初始化时, ``flag``被初始化为0, 表示锁是空闲的. 在``lock(lock_t*)``中, 会循环检查``flag``标记, 如果是为1的话, 表示锁已经被占有, 于是就一直自旋等待. 直到``flag``被设置为0, 也就是``unlock(lock_t*)``操作.

但是这个锁, 是不能保证正确性的. 因为 ``mutex->flag == 1``和 ``mutex->flag = 1``不是原子操作, 执行这两句的过程中, 有可能被中断.

而且, 这个锁的性能也不怎么好, 如果锁已经被占有的话, 它只会一直循环, 这样就白白浪费了CPU时间片.

如果能将``mutex->flag == 1;``和 ``mutex->flag = 1``作为一个原子操作的话, 那么就能保证锁的正确性. 换句话说, 只要把检查和更新锁的状态字段的操作作为一个原子操作的话, 就不会出现问题. 所以现代处理器普遍都提供了``campare and swap``原句来解决这个问题.

至于性能的问题: 与其让它一直自旋等待, 不如让出时间片, 等锁空闲的时候再调度或者自旋等待一段时间, 超过这个时间后还没获得锁的话, 就放弃时间片.

现在总结一下解决这两个问题需要处理的情况:

* 需要管理``flag``字段的同步状态, 利用一些同步原句来更新和管理同步状态.

* 为了提高锁的性能, 我们需要让获取锁失败的线程挂起或者等待一段时间后才挂起, 防止浪费时间片. 所以, 我们需要一个数据结构来管理记录这些获取锁失败的线程, 并且在当锁空闲的时候, 负责唤醒挂起的线程, 以便他们进行获取锁的操作.

> ## AQS的使命

经过上面的简单介绍, 我们知道构建一个安全并且性能高的锁需要处理下面的情况

* 管理同步状态

* 管理获取锁失败的线程, 并且负责挂起和唤醒获取锁失败的线程.

``AQS``的使命就是来完成上面的任务, 以便让各种类型的锁只关注自己本身的特性. 换句话说: ``AQS``是并发包中的基本骨架, 并发包中的各种锁都是基于``AQS``, ``AQS``为其他锁将所有的脏活和累活(管理同步状态, 将获取锁失败的线程排队,挂起和唤醒.)都解决掉, 让其他的锁只关注自己的特性.

> ## AQS设计

``AQS``是基于模板设计模式来实现的. 它将公用的特性自己实现, 对于具体的子类特性, ``AQS``提供了一些方法作为模板, 子类只需要实现对应的模板方法来构建就行了.

```java
protected boolean tryAcquire(int arg) { //独占式获取同步状态
      throw new UnsupportedOperationException();
  }

protected boolean tryRelease(int arg) { //独占式释放同步状态
      throw new UnsupportedOperationException();
}

protected int tryAcquireShared(int arg) { //共享式获取同步状态
      throw new UnsupportedOperationException();
}

protected boolean tryReleaseShared(int arg) { //共享式释放同步状态
    throw new UnsupportedOperationException();
}
```

``tryXXX``是``AQS``提供给子类实现的模板方法, 这些模板对应两种获取同步状态的模式: 独占式模式和共享式模式. 每种模式对应有获取和释放方法. 对于共享式, 同个时刻可以有多个线程获取同步状态, 至于获取的规则由子类去实现. 子类实现这种模式一般是共享锁. 至于独占式, 同个时刻只有一个线程可以获取, 获取的规则也是由子类去重写对应的模板方法实现. 子类实现这种模式一般是独占锁.

> ### AQS中的同步队列

对于获取锁失败的线程, ``AQS``需要用一个数据结构来追踪记录他们. 这个数据结构是一个双向链表. 也可以看做是一个FIFO的``同步队列``.

```java
static final class Node {

    static final Node SHARED = new Node(); //标记当前模式为共享式

    static final Node EXCLUSIVE = null; //标记当前模式为独占式

    static final int CANCELLED =  1; //当前节点已经被取消

    static final int SIGNAL    = -1; //标记后继节点需要被唤醒

    static final int CONDITION = -2; //标记当前节点处于等待队列中(注意不是同步队列)

    static final int PROPAGATE = -3; //用于共享模式中, 表示后继节点获取同步状态可以无条件传递下去.

    volatile int waitStatus;

    volatile Node prev; //前继节点

    volatile Node next; //后继节点

    volatile Thread thread; //获取锁失败的线程

    Node nextWaiter; //等待队列中的当前节点的下个节点
```

如果线程获取锁失败时, ``AQS``会将其包装成一个``Node``节点并且入队在同步队列中. ``Node``节点和``Thread``引用通过``volatile``来保证每次读取的值是最新的. 需要注意的是: ``nextWaiter``是``等待队列``中的后继节点引用, 这里的``等待队列``和``同步队列``不是一样的.

```java
private transient volatile Node head; //队列头结点

private transient volatile Node tail; //队列尾节点
```

每次出队时, 是从``head``节点出队, 入队时. 是从``tail``节点插入. 头尾节点也是被``volatile``修饰来保证他们在多线程环境下的可见性.

> ### AQS中的状态管理

```java
private volatile int state;

protected final int getState() {
   return state;
}

protected final void setState(int newState) {
   state = newState;
}

protected final boolean compareAndSetState(int expect, int update) {
   // See below for intrinsics setup to support this
   return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

对于同步状态的管理, ``AQS``采用一个``int``变量来表示, 并且该变量也是由``volatile``来修饰, 因此``getState``和``setState``这两个方法不用加锁. 这里需要注意的是: ``volatile``能保证可见性, 但是不能保证原子性. 它只能保证单操作的原子性, 也就是更新时不依赖本身的状态或者是其他变量的值. 如果需要原子更新的话, 应该使用``compareAndSetState(int expect, int update)``, 该方法能够保证原子性和可见性.

> ### AQS分类

总得来说, ``AQS``的操作分为两种模式:

* 共享式: 共享式可以细分为: 1. ``不响应中断的共享式获取同步状态``. 2. ``响应中断的共享式获取同步状态``. 3. ``同时响应中断和超时的共享式获取同步状态``.

* 独占式: 独占式跟共享式基本一样, 可以分为: 1. ``不响应中断的独占式获取同步状态``. 2. ``响应中断的独占式获取同步状态``. 3. ``同时响应中断和超时的独占式获取同步状态``.

下面我们通过官方的例子来解析AQS的原理. 如果理解了下面例子的话, 理解并发包中的其他锁自然也不在话下.

> ### 独占式锁

```java
public class Mutex implements Lock, java.io.Serializable {

   // Our internal helper class
   private static class Sync extends AbstractQueuedSynchronizer {
     // Reports whether in locked state
     protected boolean isHeldExclusively() {
       return getState() == 1;
     }

     // Acquires the lock if state is zero
     public boolean tryAcquire(int acquires) {
       assert acquires == 1; // Otherwise unused
       if (compareAndSetState(0, 1)) {
         setExclusiveOwnerThread(Thread.currentThread());
         return true;
       }
       return false;
     }

     // Releases the lock by setting state to zero
     protected boolean tryRelease(int releases) {
       assert releases == 1; // Otherwise unused
       if (getState() == 0) throw new IllegalMonitorStateException();
       setExclusiveOwnerThread(null);
       setState(0);
       return true;
     }

     // Provides a Condition
     Condition newCondition() { return new ConditionObject(); }

     // Deserializes properly
     private void readObject(ObjectInputStream s)
         throws IOException, ClassNotFoundException {
       s.defaultReadObject();
       setState(0); // reset to unlocked state
     }
   }

   // The sync object does all the hard work. We just forward to it.
   private final Sync sync = new Sync();

   public void lock() {
     sync.acquire(1);
   }

   public boolean tryLock() {
     return sync.tryAcquire(1);
   }

   public void unlock() {
     sync.release(1);
   }

   public Condition newCondition() {
     sync.newCondition();
   }
   public boolean isLocked() {
     return sync.isHeldExclusively();
   }

   public void lockInterruptibly() throws InterruptedException {
     sync.acquireInterruptibly(1);
   }
   public boolean tryLock(long timeout, TimeUnit unit)
       throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
  }
 }
}
```

``Mutex``是一个简单的独占锁, 同一个时刻只允许有一个线程获取锁. ``Sync``是``Mutex``的一个静态内部类, 该类继承了``AQS``, 并且重写了``AQS``提供的模板方法. ``Sync``是``Mutex``一个代理类, ``Mutex``的所有操作都是代理给``Sync``. ``Sync``会调用``AQS``的方法, 而``AQS``又会回调``Sync``实现的模板方法.
这就是``AQS``的设计思路: 实现一个锁时, 往往是设置一个内部静态类, 该类继承``AQS``并且重写其中的模板方法, 最后锁内部的方法都代理给这个内部静态类. 并发包中的其他锁的实现思想也是这样的.

> #### 不响应中断的获取同步状态: acquire(int)

```java
public void lock() {
  sync.acquire(1);
}
```

``lock()``方法为不响应中断获取同步状态, 内部会调用``sync``的``acquire``, 而``acquire``方法是定义在``AQS``里面的. 那么先到``AQS``中看看.

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

首先会调用``tryAcquire(int)``来尝试获取同步状态, 如果获取成功的话, 直接返回. 获取失败的话, 需要入队. 获取的逻辑是在``Sync``重写的``tryAcquire(int)``.

```java
public boolean tryAcquire(int acquires) {
  assert acquires == 1; // Otherwise unused
  if (compareAndSetState(0, 1)) {
    setExclusiveOwnerThread(Thread.currentThread());
    return true;
  }
  return false;
}
```

``Mutex``实现得很简单, 通过``CAS``来更新状态. 如果获取成功的话, 返回``true``. 失败的话, 返回``false``.

如果获取同步状态失败的话, 也就是``tryAcquire``返回``false``的话, ``AQS``会将对应的线程包装成一个``Node``节点, 然后加入同步队列中, 最后挂起线程.

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) { //由于Head和Tail节点采用延迟加载的策略, 因此tail有可能为空
        node.prev = pred;
        if (compareAndSetTail(pred, node)) { //成功的话, 证明入队成功.
            pred.next = node;
            return node;
        }
    }
    enq(node); //失败的话, 说明被其他线程给更新了尾节点, 因此进入enq方法入队.
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // 由于Head和Tail节点采用延迟加载的策略, 因此tail有可能为空
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

``addWaiter``内部首先尝试使用``CAS``来更新尾节点, 也就是插入新的节点. 如果``CAS``返回``true``的话, 证明插入成功, 如果失败的话, 进入``enq``方法.

在``enq``中, 通过一个死循环来保证入队的成功.

入队完成后, 进入``acquireQueued(final Node node, int arg)``方法:

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor(); //拿到当前节点的前驱节点
            if (p == head && tryAcquire(arg)) { //如果当前节点的前驱节点为Head节点的话, 证明该节点处于队列的第二位置(第一是头节点. 仅仅做标记用), 因此它有权获取同步状态.
                setHead(node); //获取成功后, 更新头节点
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //如果p != head或者获取同步状态失败的话, 需要将线程挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

``acquireQueued``的逻辑: 首先拿到当前节点的前驱节点, 如果前驱节点为``head``的话, 证明该节点处于队列的第二个位置, 由于``head``节点仅仅起标记的作用, 因此处于第二个位置的节点逻辑上是处于队头, 它能够竞争同步状态. 如果前驱节点不是``head``节点的话, 需要将线程挂起.

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fkr4pd1fycj30r607zt92.jpg)

现在我们来看看挂起线程的逻辑:

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL) //首先判断是不是已经设置了SIGNAL状态, 是的话, 证明需要被挂起
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) { //节点状态为CANCELLED, 跳过这些取消的节点
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL); //设置节点为SIGNAL, 表明需要被挂起.
    }
    return false;
}
```

一个线程被挂起前, 必须设置状态为``SIGNAL``, 设置为``SIGNAL``后, 表明前驱节点出队后, 必须唤醒这个节点. 如果节点的状态被设置为``CANCELLED``的话, 说明它不需要被挂起.

所以``shouldParkAfterFailedAcquire(Node pred, Node node)``, 如果线程需要被挂起的话, 它的状态为0(默认状态), 那么它通过``compareAndSetWaitStatus(pred, ws, Node.SIGNAL);``来将状态设置为``SIGNAL``, 表明它需要被挂起, 在下次再调用该方法时, 会执行
```java
if (ws == Node.SIGNAL) {
  return true;
}
```
返回``true``来表示它需要被挂起.

如果线程的状态被设置为``CANCELLED``, 也就是``ws > 0``, 那么会跳过这些取消的节点, 下次循环进入时, 执行上面的逻辑.

当``shouldParkAfterFailedAcquire(Node pred, Node node)``返回``true``时, 会调用``parkAndCheckInterrupt()``来挂起线程.

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

``LockSupport.park(this)``是一个挂起线程的操作.

现在我们再经过一张图来理清``acquire(int)``的逻辑

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fkrwnsxu5ej30oi0mq75m.jpg)

当线程尝试获取同步状态时, 如果成功的话, 直接退出. 如果失败的话, 将线程包装成一个``Node``节点并且加入到队列中. 入队后, 需要判断线程是否需要被挂起. 如果当前节点的前驱节点是``head``节点的话, 尝试获取同步状态, 如果成功的话, 将自己设置为头节点后退出. 如果当前节点的前驱节点不是``head``或者是``head``节点但是获取同步状态失败, 将线程挂起. 被挂起的线程会被前驱节点唤醒, 接着继续竞争同步状态.

> #### 响应中断的获取同步状态: acquireInterruptibly(int arg)

``acquireInterruptibly(int arg)``可以响应线程的中断而退出.

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}

```

首先检查线程的中断状态标记, 如果已经被设置了中断的话, 抛出中断异常来响应. 如果没有被中断的话, 先尝试快速的获取同步状态, 如果成功的话, 直接退出. 失败的话进入``doAcquireInterruptibly(arg);``方法:

```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

``doAcquireInterruptibly(int arg)``方法的逻辑和``acquireQueued(final Node node, int arg)``大致一样. 下面只说说不一样的地方. 如果``parkAndCheckInterrupt()``返回``true``的话, 会抛出中断异常来响应中断.

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

线程被唤醒后会检查中断标记位.

由于``acquireInterruptibly(int arg)``逻辑和``acquire``差不多, 所以这里不多讲.

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fkrxgdyggoj30p30mujss.jpg)

> #### 支持超时的获取同步状态: tryAcquireNanos(int arg, long nanosTimeout)

``tryAcquireNanos(int arg, long nanosTimeout)``方法如果在给定的一个时间内不能够获取锁的话, 会直接返回. 该方法同时也支持中断. 换句话说``tryAcquireNanos(int arg, long nanosTimeout)``是在``acquireInterruptibly(int arg)``的基础上加入超时获取的逻辑.

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
```

首先检查中断标志, 如果``true``的话, 抛出异常来响应. ``false``的话, 首先尝试获取同步状态, 成功的话直接返回. 失败的话, 进入``  doAcquireNanos(arg, nanosTimeout);``方法:

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout; //记录超时的时间
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            nanosTimeout = deadline - System.nanoTime(); //如果nanosTimeout < 0=的话, 证明超时了
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold) // 剩余的超时时间如果小于这个spinForTimeoutThreshold的话, 线程不会被挂起, 而是会自旋
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

``doAcquireNanos(int, long)``的逻辑和`` doAcquireInterruptibly(int arg)``大体相同, 不同的是: ``doAcquireNanos(int, long)``支持超时获取锁. 超时获取的逻辑: 首先计算出超时的时间戳``final long deadline = System.nanoTime() + nanosTimeout;``, 在自旋的过程中, 如果``deadline - System.nanoTime() <= 0``的话,证明已经超时, 所以返回``false``. 如果大于0的话, 说明还没超时. 如果线程挂起的另外一个条件是``nanosTimeout > spinForTimeoutThreshold``. 这样做为因为``spinForTimeoutThreshold = 1000``纳秒, 已经很小了, 如果再进行超时等待的话, 反而会更加不准确. 因此, 如果``nanosTimeout``小于等于``spinForTimeoutThreshold(1000纳秒)``时,将不会使该线程进行超时等待,而是进入快速的自旋过程。

> #### 释放同步状态: release(int arg)

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head; //获取头结点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

``release(int)``为释放同步状态, 它会调用``tryRelease(int)``模板方法, 这个模板方法是由``Mutex``来重写的, 具体代码前面已经贴出来了.
接着它会进入``unparkSuccessor(h);``来唤醒同步队列中的线程:

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

为了防止提前线程被提前唤醒, 首先利用``CAS``将状态更新为默认状态. 一般而言, 需要唤醒的节点为``head``节点的下个节点, 但是为了防止节点已经被取消或者为空, 需要判断一下, 如果是的话, 从队列找到下一个需要释放的节点. 最后才唤醒线程.

> ### 共享锁

共享锁是指允许同个时刻, 允许多个线程获得资源. 共享锁的实现需要``AQS``的四组方法支持

* ``acquireShared(int arg)``: 不支持中断获取同步状态

* ``acquireSharedInterruptibly(int arg)``: 支持中断地获取同步状态

* ``tryAcquireSharedNanos(int arg, long nanosTimeout)``: 超时获取并且支持中断获取同步状态

* ``releaseShared(int arg)``: 释放同步状态

这四组方法的逻辑跟独占模式下的方法的逻辑差不多, 因此, 这里只分析``acquireShared(int arg)``和``releaseShared(int arg)``.

> #### acquireShared(int arg)

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

``tryAcquireShared(arg)``为子类重写的模板方法. 如果方法返回值小于0的话, 说明获取同步状态失败. 因此进入``  doAcquireShared(arg)``进行排队和挂起等操作

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED); //添加进队列
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor(); //获取前驱节点
            if (p == head) {
                int r = tryAcquireShared(arg); //获取同步状态
                if (r >= 0) { //如果同步状态还有剩余的话
                    setHeadAndPropagate(node, r); //唤醒后续的线程
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

当前驱节点为``head``节点的话, 此时会超时获取同步状态, 如果成功并且状态大于等于0的话, 证明同步资源还有剩余, 可以唤醒后面的线程. 因此会调用``setHeadAndPropagate(node, r) ``, 设置头结点并且将唤醒后面的线程.

> #### releaseShared(int arg)

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

共享式的释放和独占式的释放的主要区别在于: 独占式释放的时候, 只有一个线程在执行, 因此不存在竞争条件, 直接唤醒后续线程即可. 但是在共享式中, 由于同个时刻有多个线程在执行, 因此存在条件竞争, ``doReleaseShared()``内部通过循环和``CAS``来保证线程安全.

> ### 总结

``AQS``利用模板设计模式来为其子类屏蔽了同步状态的管理, 同步队列的管理, 线程的挂起和唤醒等操作, 使得子类只需要关注本身获取同步状态的逻辑. ``AQS``内部总体实现分为两种模式:

* 共享式

* 独占式

共享式和独占式都支持不可中断, 可中断, 可中断并且超时获取同步状态.

关于``AQS``, 其内部还有一个``ConditionObject``类, 该类是实现等待/通知模式. 由于篇幅的关系, 打算在下篇中分析.
