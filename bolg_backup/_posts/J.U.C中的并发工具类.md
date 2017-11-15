---
layout: post
title: J.U.C中的并发工具类
data: 2017/10/29
subtitle: ""
description: ""
tags:
  - Java并发包#工具类
categories:
  - Java
---

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fl1ikq8wyhj30m80b4qi7.jpg)

在``Java``的``J.U.C``包中提供了几个并发编程中非常有用工具类, 例如: ``Semaphore``, ``CountDownLatch``和``CyclicBarrier``. 这次准备来介绍这三个工具类.

> ## Semaphore

``Semaphore``中文意思为: 信号量. 它主要用来维护一组有限的资源. 比如数据库连接, ``Socket``连接. 信号量的使用方式很简单, 在构造函数中, 传入你需要维护的有限资源的数量, 每次需要申请资源时, 调用``acquire()``, 接着做一些业务逻辑, 完成后, 再调用``release()``归还有限资源. 如果有限资源被申请完了后, 还调用``acquire()``的话, 调用的线程会被阻塞, 直到其他线程归还资源后, 才从阻塞中返回.

> ### 原理

``Semaphore``的原理很简单. 基于``AQS``构建的. 它内部维护一个状态计数器, 每次调用申请资源时, 都会递减状态计数器值, 归还资源时, 会增加状态计数器的值. 当状态计数器为负数时, 说明有限资源已经被申请完了, 这时``AQS``会将申请不到资源的线程添加进同步队列中, 并挂起线程, 直到其他线程唤醒.

由于``Semaphore``也是基于``AQS``实现的, 所以它支持不响应中断, 响应中断和响应中断且超时获取资源这三种类型. 同时获取资源的方式支持公平性和非公平性的获取. 默认的实现是非公平的获取方式.

> ### 例子

下面给出一个简单的例子: 数据库连接池

```java
class Main {

    private static DBPool pool = new DBPool(10);

    public static void main(String[] args) {
        doSomeWork();
    }

    private static void doSomeWork() {
        Connection connection = pool.get();
        //访问数据库
        pool.release(connection);
    }
}

public class DBPool {

    private Semaphore semaphore;

    private Connection[] pool;

    private boolean[] connectionFlags;

    public DBPool(int size) {
        this(size, false);
    }

    public DBPool(int size, boolean isFair) {
        semaphore = new Semaphore(size, isFair);
        initPool(size);
    }

    private void initPool(int size) {
        pool = new Connection[size];
        connectionFlags = new boolean[size];
        for (int i = 0; i < size; i++) {
            pool[i] = DBDriver.create();
            connectionFlags[i] = false;
        }
    }

    public synchronized Connection get() {
        semaphore.acquireUninterruptibly();
        return fetchConnection();
    }

    private Connection fetchConnection() {
        for (int i = 0; i < pool.length; i++) {
            if (!connectionFlags[i]) {
                connectionFlags[i] = true;
                return pool[i];
            }
        }
        return null;
    }

    public synchronized void release(Connection connection) {
        for (int i = 0; i < connectionFlags.length; i++) {
            if (pool[i] == connection && connectionFlags[i]) {
                connectionFlags[i] = false;
                semaphore.release();
            }
        }
    }

    static class DBDriver {
        static Connection create() {
            return null; //fake connection
        }
    }
}
```

> ## CountDownLatch

当一个或者多个线程的执行需要等待其他线程执行完后才能执行的话, ``CountDownLatch``能够很好的完成任务.

``CountDownLatch``的使用方式很简单, 只要在构造函数中传入需要等待执行完的线程数, 在需要等待其他线程执行完才执行的线程调用``await()``, 如果此时其他线程还没执行完的话, 该操作会阻塞, 直到每个线程中调用``countDown()``方法来表示自己已经完成了, 最后被阻塞的线程才会从``await()``中返回.

> ### 例子

下面是``CountDownLatch``的一个例子: 工头要带搬砖工去搬砖, 所以工头必须得等他们都上车了才可以开车.

```java
public class CDTest {

    private static CountDownLatch countDownLatch = new CountDownLatch(10); //需要等待10个搬砖工上车

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread(new Worker(countDownLatch)).start();
        }
        try {
            countDownLatch.await(); //等待搬砖工上车完才开车.
            System.out.println("OK, 搬砖工已经上车了, 开车去搬砖");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    static class Worker implements Runnable {

        private CountDownLatch countDownLatch;

        public Worker(CountDownLatch countDownLatch) {
            this.countDownLatch = countDownLatch;
        }

        @Override
        public void run() {
            getOnBus();
            countDownLatch.countDown(); //执行完了.
        }

        void getOnBus() {
            System.out.println("搬砖工上车了!");
        }
    }
}
```

> ### 原理

``CountDownLatch``内部实现的原理很简单. 它基于``AQS``, 通过构造函数传入的``int``数值, 来维护一个计数器.

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

Sync(int count) {
    setState(count);
}
```

传入``count``被设置为``AQS``内部的状态值.

接下来看看``await()``

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

在``await()``中最终会回调``tryAcquireShared(int acquires)``. 该方法实现的逻辑: 如果状态不为0的话, 说明还有其他线程还没执行完, 所以返回-1告诉``AQS``阻塞当前线程. 如果状态为0, 说明全部线程已经执行完, 此时返回1, 告诉``AQS``不用阻塞线程.

当线程被阻塞后, 需要它等待的线程执行完才会被唤醒. 被等待的线程调用``countDown()``表示自己已经完成.

```java
public void countDown() {
    sync.releaseShared(1);
}

protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
}
```

``countDown()``通过循环和``CAS``来更新同步状态, 当最后一个线程也调用了该方法的话, 同步状态为0. 此时被阻塞的线程会得以唤醒.

> ## CyclicBarrier

``CyclicBarrier``允许一组线程互相等待其到达一个同步点的时候才继续执行. 同时``CyclicBarrier``具有重用性, 当等待的一组线程被释放后(成功或者失败), 它能够被继续使用. 这点和``CountDownLatch``不一样, ``CountDownLatch``只能够使用一次.

``CyclicBarrier``遵循``all-or-none``的损坏模式, 相互等待的一组线程如果其中一个线程被中断的或者等待超时的话, 其他线程也将失败. 简单来讲, 要么全部成功, 要么全部失败.

``CyclicBarrier``还支持一组等待的线程被释放后, 执行传入的``Runnable``.

> ### 例子

下面给出一个例子. 游戏玩家都必须等待各自准备好才能开始.

```java
public class CBTest {

    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(5);

    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            new Thread(new Player("玩家-" + i, cyclicBarrier)).start();
        }
        try {
            Thread.currentThread().join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    static class Player implements Runnable {

        private String name;

        private CyclicBarrier cyclicBarrier;

        public Player(String name, CyclicBarrier cyclicBarrier) {
            this.name = name;
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            System.out.println(name + "已经准备好");
            try {
                cyclicBarrier.await(); //等待其他玩家
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
}
```

我们还可以在``CyclicBarrier``的构造函数中传入一个``Runnable``, 当所有线程都到达同步点后, 该任务会被执行, 默认是最后一个到达的线程执行.

```java
public class CBTest2 {

    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(5, new ReadyFinish());

    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            new Thread(new Player("玩家-" + i, cyclicBarrier)).start();
        }
        try {
            Thread.currentThread().join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    static class Player implements Runnable {

        private String name;

        private CyclicBarrier cyclicBarrier;

        public Player(String name, CyclicBarrier cyclicBarrier) {
            this.name = name;
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            System.out.println(name + "已经准备好");
            try {
                cyclicBarrier.await(); //等待其他玩家
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }

    static class ReadyFinish implements Runnable {

        @Override
        public void run() {
            System.out.println("所以玩家已经准备完了, 开始游戏");
        }
    }
}
```

> ### 原理

``CyclicBarrier``内部基于``ReentrantLock``和``Condition``实现的. 到达同步点的线程会被暂时挂在条件队列中等待.

```java
//支持重用性, 表示第几代
private static class Generation {
       boolean broken = false; //是否成功等待所有线程到达同步点
   }

   private final ReentrantLock lock = new ReentrantLock(); //确保线程安全

   private final Condition trip = lock.newCondition(); //到达安全点的线程会被挂在等待队列中

   private final int parties; //互相等待的一组线程

   private final Runnable barrierCommand; //全部线程到达后, 执行的一个任务

   private Generation generation = new Generation();

   private int count; //记录还有多少个线程没到达同步点
```

上面是``CyclicBarrier``中的实例域, 具体讲解看注释.

下面我们来看看线程到达同步点时的逻辑:

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}

private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        final Generation g = generation;

        if (g.broken) //失败
            throw new BrokenBarrierException();

        if (Thread.interrupted()) { //线程被中断了, 失败
            breakBarrier();
            throw new InterruptedException();
        }

        int index = --count; //获得还没到达的线程数
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed)
                    trip.await(); //挂起线程
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos); //挂起线程
            } catch (InterruptedException ie) { //中断失败
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken) //有一个线程失败了, 所有线程也跟着失败
                throw new BrokenBarrierException();

            if (g != generation) //如果当前代对象不相等的话, 证明已经更新代数了.
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```

首先获得还没到达的线程数, 如果不为0的话, 说明还有线程没到达, 这时, 应该阻塞线程. 当最后一个线程到达时, ``index``为0, 此时说明全部线程已经到达, 所以进入唤醒等待线程的逻辑:

```java
if (index == 0) {  // tripped
    boolean ranAction = false;
    try {
        final Runnable command = barrierCommand;
        if (command != null)
            command.run();
        ranAction = true;
        nextGeneration();
        return 0;
    } finally {
        if (!ranAction)
            breakBarrier();
    }
}

private void nextGeneration() {
    trip.signalAll(); //唤醒在条件队列中等待的线程
    //重置CyclicBarrier, 使得它可以被复用
    count = parties;
    generation = new Generation();
}
```

如果构造函数中有传入``Runnable``对象的话, 最后一个线程会执行该``Runnable``对象, 然后进入``nextGeneration()``中唤醒在同步点等待的线程, 最后更新当前年代.

如果等待线程在等待的过程中有被中断或者等待超时的话, 会执行``breakBarrier()``

```java
private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}
```

首先设置当代已经被破坏了, 这样可以让其他线程都失败.然后唤醒在等待队列上等待的线程. 被唤醒的线程会检查``broken``字段, 最后抛出异常.

如果等待的过程中没有超时或者被中断的话, 其他线程从阻塞的方法中返回后, 当前的``generation``是否相等, 不是的话, 说明已经进入下一代了,所以返回一个``index``, 该``index``表示这线程是第几个到达同步点的.

> ## 总结

``Semaphore``用来维护一组有限的资源, 每次申请资源时, 都会递减资源数, 如果资源没了的话, 会阻塞当前线程, 直到有可用的资源为止. 有限的资源可以是: 数据库连接, ``Socket``连接.

``CountDownLatch``适用于 : 当一个或者多个线程的执行需要等待其他线程执行完后才可以执行的场景.

多个线程需要等待彼此到达一个同步点时, 才继续执行, 这种情况下, 可以用``CyclicBarrier``. 而且它具有重用行, 可被多次使用, 这点和``CountDownLatch``不一样, 后者只能被使用一次.
