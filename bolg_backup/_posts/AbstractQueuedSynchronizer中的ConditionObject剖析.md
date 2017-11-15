---
layout: post
title: AbstractQueuedSynchronizer中的ConditionObject剖析
data: 2017/10/24
subtitle: ""
description: ""
tags:
  - Java并发包#AQS
categories:
  - Java
---

# AbstractQueuedSynchronizer中的ConditionObject剖析

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fktlv07v17j30jg0dwngp.jpg)

在多线程环境中, 有时候, 一个线程的执行是需要等待一个条件发生后才能执行的. 在经典的生产者和消费者模式中, 如果缓冲区满后, 生产者是不能向缓冲区投放``item``的, 它需要等待一个条件: ``缓冲区不为满的状态``. 同理, 如果缓冲区为空时, 消费者是不能消费``item``的, 它需要等待一个条件: ``缓冲区不为空``. 一个线程需要等待一定的条件发生, 这个条件往往是别的线程触发的, 这就是经典的``等待/唤醒``模式.

在``JDK1.5``之前要实现这种模式的话, 只能够借助``synchronized``关键字和``Object``的对象锁来实现. 在``1.5``之后, 可以利用基于``AQS``实现的锁和``AQS``内部的``ConditionObject``来实现. 下面以``ReentrantLock``为例实现一个等待/唤醒模式

```java
public class ConditionObjectTest {

    private ReentrantLock lock = new ReentrantLock();

    private LinkedBlockingQueue<Integer> queue = new LinkedBlockingQueue<>();

    private Condition isEmpty = lock.newCondition();

    private Condition isFull = lock.newCondition();

    private int count;

    private static final int MAX_LENGTH = 10;

    public void product(int i) {
        ReentrantLock reentrantLock = lock;
        lock.lock();
        if (count >= MAX_LENGTH) {
            try {
                isEmpty.await(); //等待一个缓冲区不为满的条件
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        queue.add(i);
        count++;
        isFull.signal(); //通知缓冲区已经不为空
        reentrantLock.unlock();
    }

    public void consume() {
        ReentrantLock reentrantLock = lock;
        lock.lock();
        if (queue.isEmpty()) {
            try {
                isFull.await(); //等待缓冲区不为空的条件
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        int i = queue.peek(); //消费
        count--;
        isEmpty.signal(); //通知缓冲区不为满
        reentrantLock.unlock();
    }
}
```

在这个例子中定义了两个条件, 一个是缓冲区为空的条件, 另外一个是缓冲区为满的条件. 当队列达到最大长度时, 也就是缓冲区满了, 此时生产者调用了``isEmpty.await()``来等待一个缓冲区不为满的条件, 因此线程会暂时被挂起. 这个条件是由消费者消费了一个``item``后调用``isEmpty.signal()``是触发的. 触发了这个条件后, 会唤醒处于等待的生产者线程, 使它从``isEmpty.await()``中返回. 至于当缓冲区为满时的情况原理是一样的, 这里不多分析. 下面主要分析``AQS``内部怎么实现等待通知模式的.

> ## ConditionObject

一般而言, 线程是否需要等待一个条件的判断, 这个判断往往是访问一个共享变量, 在前面的例子中, 这个共享变量是``缓冲区``. 因此, 每次等待一个条件或者触发一个条件时, 都必须先获得锁. 这也解释为什么``ConditionObject``会作为``AQS``的内部类.

> ## ConditionObject的等待/通知方法

``ConditionObject``中的等待方法支持的类型跟``AQS``中一样, 都支持不可中断, 可中断, 超时三种类型.

> #### 等待

* ``awaitUninterruptibly()``: 不可中断的等待一个条件

* ``await()``: 响应中断的等待一个条件

* ``awaitNanos(long nanosTimeout)``: 超时等待一个条件, 如果超过指定的等待时间的话, 会直接返回. 超时等待还有两个重载方法, 这里只列出一个.

> #### 通知

* ``signal()``: 从等待队列中唤醒一个正在等待的线程

* `` signalAll()``: 唤醒等待队列中的全部线程.

> #### ``awaitUninterruptibly()``解读

三种类型的等待方法的实现逻辑跟``AQS``中的获取同步状态的三种类型差不多, 这里只分析``awaitUninterruptibly()``.

```java
public final void awaitUninterruptibly() {
    Node node = addConditionWaiter(); //添加进等待队列, 等待队列不是AQS中的同步队列
    int savedState = fullyRelease(node); //释放同步状态, 相当于释放锁
    boolean interrupted = false;
    while (!isOnSyncQueue(node)) { //判断节点是不是在同步队列中, 也就是等待获取同步状态的队列
        LockSupport.park(this); //不在同步队列的话, 证明已经在等待队列了, 需要等待一个条件, 因此挂起线程, 等待其他线程唤醒
        if (Thread.interrupted())
            interrupted = true;
    }
    if (acquireQueued(node, savedState) || interrupted) //被唤醒后, 重新竞争同步状态, 也就是竞争锁
        selfInterrupt();
}
```

由于每次调用这个方法时, 必定时已经获取了锁的, 所以不用控制同步, 实现起来比较简单. 首先将当前节点添加进等待队列, 接着释放同步状态, 也就是释放锁, 它准备要被挂起了, 挂起前必须释放同步状态, 不然有可能引起死锁. 然后, 判断节点是否存在同步队列中, 如果不存在的话,证明已经被添加进等待队列中, 此时进入``While``循环挂起线程. 接下来执行到这里就停了. 需要等待其他线程触发它等待的条件.

> #### ``signal()``解读

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

每次都是释放等待队列中的第一个节点, 说明等待队列是一个``FIFO``队列. 释放的主要逻辑都在``doSignal(first)``中.

```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

在``doSignal()``中会调用``transferForSignal(first)``将等待队列中的节点移动到同步队列中

```java
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

首先通过``CAS``设置节点的状态, 如果设置失败的话, 说明节点已经被取消. 接着调用``enq(node)``方法, 将节点移动到同步队列中, 然后设置节点的状态为``SIGNAL``.最后唤醒线程. 唤醒后, 在之前的等待方法中, 会被执行.

```java
public final void awaitUninterruptibly() {
    Node node = addConditionWaiter(); //添加进等待队列, 等待队列不是AQS中的同步队列
    int savedState = fullyRelease(node); //释放同步状态, 相当于释放锁
    boolean interrupted = false;
    while (!isOnSyncQueue(node)) { //判断节点是不是在同步队列中, 也就是等待获取同步状态的队列
        LockSupport.park(this); //不在同步队列的话, 证明已经在等待队列了, 需要等待一个条件, 因此挂起线程, 等待其他线程唤醒
        if (Thread.interrupted())
            interrupted = true;
    }
    if (acquireQueued(node, savedState) || interrupted) //被唤醒后, 重新竞争同步状态, 也就是竞争锁
        selfInterrupt();
}
```

由于节点已经被移动到同步队列中, 所以``isOnSyncQueue(node)``会返回``true``跳出循环, 接着调用``acquireQueued(node, savedState)``来竞争同步状态, 也就是重新获得锁. 如果成功的话, 将从``awaitUninterruptibly()``中返回.

对于``signalAll()``, 它通过一个循环, 调用``signal()``来实现唤醒等待队列中的全部线程.

> #### 总结

当一个线程调用等待方法时, 它首先会把自己添加进等待队列中, 接着释放同步状态, 然后被挂起. 直到其他线程调用唤醒的方法, 节点会被移动到同步队列中并且唤醒对应的线程去竞争同步状态, 如果成功的话, 将从等待的方法中返回, 下面是逻辑图:

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fktlspogakj30f20n9q4q.jpg)
