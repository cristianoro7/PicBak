---
title: AsyncTask源码浅析
data: 2016/11/13
tags:
  - 异步框架#AsyncTask
categories:
  - Android
---
##### 源码浅析

* 概述
AsyncTask是一个执行异步任务的小型框架，里面封装了Handler，使得使用者不必关心线程之间的切换，虽然现在执行异步任务都不会用AsyncTask，用得更多的是Bolt Tasks 和RxJava，但是AsyncTask中的设计思想还是很多值得学习的，比如：内部中，串行运行任务时的队列控制，handler的将结果回调回主线程，以及如何取消正在执行的任务等。。
<!-- more -->
* 我们知道，使用AsyncTask的要先实例化该类再调用其中的execute（）方法，那么自然先要来看其中的构造方法都初始化了哪些操作：

```java
 /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     */
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```
* 上面的代码中，分别实例化了WorkRunnable对象和FutureTask对象，其中，WorkRunnable是一个内部类，我们来看看该内部类：
```java
 private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }
```
* 该类实现了Callable接口，其中保存了Params参数，也就是AsyncTask传入的参数

* 我们接下去看看execute（Params)
```java
@MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```
* 该方法返回AsyncTask本身，调用者可以根据需要保持对其的引用
* execute（Params... params）executeOnExecutor(sDefaultExecutor, params)；我们点进该方法中看看
```java
@MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```
* Status是类中的枚举，作用是记录Task的运行状态。onPreExecute()，我们可以在该方法中做一些UI线程的初始化操作
* 接着将传进的参数赋值给mWork.parmas,也就是在构造函数初始化的WorkRunnable对象，保存了传入的参数
* 最后调用了传入的线程池的execute（Runnable）并且返回本身
* 传入的线程池是里面默认的线程池，该线程池是一个串行执行的
* 我们来看看该默认的线程池：
```java
private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
* 内部类中，将传入的Runnable对象添加到mTask队列中，并且在scheduleNext()方法中从mTask队列中拿出头元素，将该元素在后台线程池中执行
* 讲到这里，我们可以看出AsyncTask中的串行执行是通过在默认的线程池中进行队列控制，真正执行的是在后台线程池中执行，并且在执行完再安排下一个任务到后台线程池，这样就巧妙的完成了串行运行
* 我们回来execute（Runnable），传入的参数是mFutureTask，该对象是在之前构造函数实例化的，让我们回到构造函数：
```java
 public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
```
在FutureTask中，将callable对象出传入，现在看看mWorker其中的操作
* 首先设置一个任务被调用的标记
* 接着调用doInBackground(mParams);该方法是我们重写的方法
* 最后将结果传入postResult(Result)中

```java
 private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```
* 方法中的逻主要是通过handler发送一条消息，该消息是包含结果对象的消息
* 我们先来看看内部类AsyncTaskResult<Result>：
```java
@SuppressWarnings({"RawUseOfParameterizedType"})
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
```
* 类中主要是保存了传入的结果以及AsyncTask自身
* 接着来看看InternalHandler()：

```java
 private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
* 该内部类重写的Handler是运行在线程的handler，在handleMessage中接收处理消息

* 分析完了这两个内部类，我们回到postResult中发送消息，消息到达的是InternalHandler的 handleMessage，
* result.mTask.finish(result.mData[0])，该语句调用了finish。

```java
 private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```

* 方法中会先判断任务是不是被取消，如果没有的话，就地调用onPostExecute(result);也就是我们重写的结果方法

* 以上就是AsyncTask的执行流程。publishProgress方法的执行跟postResult差不多，这里不分析

* 关于AsyncTask现在支持并行运行，如果想要并且并行运行的话，可以调用executeOnExecutor(THREAD_POOL_EXECUTOR,
  Params... params)。
