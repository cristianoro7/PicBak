---
title: Picasso中值得学习的技巧
data: 2017/1/30
tags:
  - 图片加载框架#Picasso
categories:
  - Android
---
#### Picasso中的线程池
* Picasso中的线程池主要是对对网络状态进行了监听,并且包装了一个FutureTask实现请求的优先级比较。

##### 分析:

* 首先在Picasso中,定义了一个优先级枚举类型:

```java
/**
   * The priority of a request.
   *
   * @see RequestCreator#priority(Priority)
   */
  public enum Priority {
    LOW,
    NORMAL,
    HIGH
  }

```

<!-- more -->

* 上面的priority就是请求优先级的类型。
* 有了优先级,我们就可以在创建Request的时候这只设置优先等级
```java
/**
   * Set the priority of this request.
   * <p>
   * This will affect the order in which the requests execute but does not guarantee it.
   * By default, all requests have {@link Priority#NORMAL} priority, except for
   * {@link #fetch()} requests, which have {@link Priority#LOW} priority by default.
   */
  public RequestCreator priority(Priority priority) {
    data.priority(priority);
    return this;
  }
```
* 通过阅读上面的代码,我们知道可以这样来设置请求的优先等级
```java
Picasso.with(this)
                .load(url)
                .priority(Picasso.Priority.HIGH)
                .into(mImageView);
```
* 需要主要的是,虽然设置了优先等级,但是并不是绝对的。
```java
hunter.future = service.submit(hunter);
```
* 可以看到,BitmapHunter被提交到了Picasso的线程池,我们进去其中的sumbit(BitmapHunter)方法中看看
```java
@Override
  public Future<?> submit(Runnable task) {
    PicassoFutureTask ftask = new PicassoFutureTask((BitmapHunter) task);
    execute(ftask);
    return ftask;
  }
```
* 传进来的BitmapHunter被包装成了一个PicassoFutureTask,在PicassoFutureTask中重写compareTo(PicassoFutureTask other)实现优先级的比较:
```java
@Override
    public int compareTo(PicassoFutureTask other) {
      Picasso.Priority p1 = hunter.getPriority();
      Picasso.Priority p2 = other.hunter.getPriority();
      /* High-priority requests are "lesser" so they are sorted to the front.*/
      /* Equal priorities are sorted by sequence number to provide FIFO ordering.*/
      return (p1 == p2 ? hunter.sequence - other.hunter.sequence : p2.ordinal() - p1.ordinal());
    }
```

* 我们再看看线程池中对网络状态的监听:
```java
void adjustThreadCount(NetworkInfo info) {
    if (info == null || !info.isConnectedOrConnecting()) {
      setThreadCount(DEFAULT_THREAD_COUNT);
      return;
    }
    switch (info.getType()) {
      case ConnectivityManager.TYPE_WIFI:
      case ConnectivityManager.TYPE_WIMAX:
      case ConnectivityManager.TYPE_ETHERNET:
        setThreadCount(4);
        break;
      case ConnectivityManager.TYPE_MOBILE:
        switch (info.getSubtype()) {
          case TelephonyManager.NETWORK_TYPE_LTE:  /* 4G*/
          case TelephonyManager.NETWORK_TYPE_HSPAP:
          case TelephonyManager.NETWORK_TYPE_EHRPD:
            setThreadCount(3);
            break;
          case TelephonyManager.NETWORK_TYPE_UMTS: /*3G*/
          case TelephonyManager.NETWORK_TYPE_CDMA:
          case TelephonyManager.NETWORK_TYPE_EVDO_0:
          case TelephonyManager.NETWORK_TYPE_EVDO_A:
          case TelephonyManager.NETWORK_TYPE_EVDO_B:
            setThreadCount(2);
            break;
          case TelephonyManager.NETWORK_TYPE_GPRS: /* 2G*/
          case TelephonyManager.NETWORK_TYPE_EDGE:
            setThreadCount(1);
            break;
          default:
            setThreadCount(DEFAULT_THREAD_COUNT);
        }
        break;
      default:
        setThreadCount(DEFAULT_THREAD_COUNT);
    }
  }
```
* adjustThreadCount方法中根据传进来的NetworkInfo来判断当前手机网络的状态,分别在4G,3G,2G的情况下调整线程数
* 传进来的网络状态是在创建分发器的时候注册的广播:
```java
 this.scansNetworkChanges = hasPermission(context, Manifest.permission.ACCESS_NETWORK_STATE);
    this.receiver = new NetworkBroadcastReceiver(this);
    receiver.register();
```

#### Picasso中加载适应图片的宽高

* 我们知道,当我们加载图片时,如果图片的大小与我们ImageView的规格不符合的时候,加载出来的图片的效果是很差的。因此,Picasso也提供了方法给我们调用:
```java
Picasso.with(this)
                .load(url)
               .fit()
                .priority(Picasso.Priority.HIGH)
                .into(mImageView);
```
* 我们只要添加fit方法时,就能够实现宽高自适应。接下来我们看看Picasso是怎么实现这个功能。
* 既然是添加了fit()方法,那么我们先进入fit方法:
```java
/** Internal use only. Used by {@link DeferredRequestCreator}. */
  RequestCreator fit() {
    deferred = true;
    return this;
  }
```
* 方法中仅仅只是对deferred = true; 在上一篇的源码分析中,在into(ImageView)中,有deferred出现的身影,我们来看看:
```java
if (deferred) {
      if (data.hasSize()) {
        throw new IllegalStateException("Fit cannot be used with resize.");
      }
      int width = target.getWidth();
      int height = target.getHeight();
      if (width == 0 || height == 0) {
        if (setPlaceholder) {
          setPlaceholder(target, getPlaceholderDrawable());
        }
        picasso.defer(target, new DeferredRequestCreator(this, target, callback));
        return;
      }
      data.resize(width, height);
    }

    Request request = createRequest(started);
    String requestKey = createKey(request);

    if (shouldReadFromMemoryCache(memoryPolicy)) {
      Bitmap bitmap = picasso.quickMemoryCacheCheck(requestKey);
      if (bitmap != null) {
        picasso.cancelRequest(target);
        setBitmap(target, picasso.context, bitmap, MEMORY, noFade, picasso.indicatorsEnabled);
        if (picasso.loggingEnabled) {
          log(OWNER_MAIN, VERB_COMPLETED, request.plainId(), "from " + MEMORY);
        }
        if (callback != null) {
          callback.onSuccess();
        }
        return;
      }
    }

    if (setPlaceholder) {
      setPlaceholder(target, getPlaceholderDrawable());
    }

    Action action =
        new ImageViewAction(picasso, target, request, memoryPolicy, networkPolicy, errorResId,
            errorDrawable, requestKey, tag, callback, noFade);
    picasso.enqueueAndSubmit(action);
  }
```
* 上面的代码节选自 into(ImageView)方法,deferred默认值是false,因此默认情况下是不会进入if语句块的,但是当我们调用了fit()后,deferred被设置为true,所以当我们调用into是时候,会进入if语句块。接着,调用了``picasso.defer(target, new DeferredRequestCreator(this, target, callback));``
* 我们进入DeferredRequestCreator对象:
```java
class DeferredRequestCreator implements ViewTreeObserver.OnPreDrawListener {

  final RequestCreator creator;
  final WeakReference<ImageView> target;
  Callback callback;

  @TestOnly DeferredRequestCreator(RequestCreator creator, ImageView target) {
    this(creator, target, null);
  }

  DeferredRequestCreator(RequestCreator creator, ImageView target, Callback callback) {
    this.creator = creator;
    this.target = new WeakReference<ImageView>(target);
    this.callback = callback;
    target.getViewTreeObserver().addOnPreDrawListener(this);
  }

  @Override public boolean onPreDraw() {
    ImageView target = this.target.get();
    if (target == null) {
      return true;
    }
    ViewTreeObserver vto = target.getViewTreeObserver();
    if (!vto.isAlive()) {
      return true;
    }

    int width = target.getWidth();
    int height = target.getHeight();

    if (width <= 0 || height <= 0) {
      return true;
    }

    vto.removeOnPreDrawListener(this);
    this.creator.unfit().resize(width, height).into(target, callback);
    return true;
  }

  void cancel() {
    callback = null;
    ImageView target = this.target.get();
    if (target == null) {
      return;
    }
    ViewTreeObserver vto = target.getViewTreeObserver();
    if (!vto.isAlive()) {
      return;
    }
    vto.removeOnPreDrawListener(this);
  }
}

```
* DeferredRequestCreator实现了ViewTreeObserver.OnPreDrawListener接口,拿到ImageView宽高后,再接调用``this.creator.unfit().resize(width, height).into(target, callback);``
* unfit()方法中仅仅对deferred设置为false,当再次调用into的方法时,也就不会进入if语句块,接着调用了resize(width, height)将得到ImageView的宽高设置到请求中,最后调用into进行重新加载。

#### Picasso中,通过设置Tag对图片进行生命周期的管理

* 通过上一篇源码分析知道,我们可以通过设置tag来管理图片的生命周期,接下来我们来分析其中的实现,先点进入tag(Object)来看:
```java
/**
   * Assign a tag to this request. Tags are an easy way to logically associate
   * related requests that can be managed together e.g. paused, resumed,
   * or canceled.
   * <p>
   * You can either use simple {@link String} tags or objects that naturally
   * define the scope of your requests within your app such as a
   * {@link android.content.Context}, an {@link android.app.Activity}, or a
   * {@link android.app.Fragment}.
   *
   * <strong>WARNING:</strong>: Picasso will keep a reference to the tag for
   * as long as this tag is paused and/or has active requests. Look out for
   * potential leaks.

   *
   * @see Picasso#cancelTag(Object)
   * @see Picasso#pauseTag(Object)
   * @see Picasso#resumeTag(Object)
   */
  public RequestCreator tag(Object tag) {
    if (tag == null) {
      throw new IllegalArgumentException("Tag invalid.");
    }
    if (this.tag != null) {
      throw new IllegalStateException("Tag already set.");
    }
    this.tag = tag;
    return this;
  }
```
* 方法中的tag最好是Activity和Fragment,因为Fragment和Activity的生命周期系统自动帮我们管理好了,把tag设置为它们的话,我们只要根据它们的生命周期来调用Picasso#cancelTag(Object)、Picasso#pauseTag(Object)、Picasso#resumeTag(Object)
* 设置玩tag后,当我们要暂停加载时,可以调用Picasso#pauseTag(Object),接着Picasso再调用分发器的``dispatcher.dispatchPauseTag(tag);``,dispatchPauseTag(tag)中又通过Handler发送一条``TAG_PAUSE``的消息,Handler接受到消息后,回调分发器的``dispatcher.performResumeTag(tag);``
```java
void performPauseTag(Object tag) {
    /* Trying to pause a tag that is already paused.*/
    if (!pausedTags.add(tag)) {
      return;
    }

    /* Go through all active hunters and detach/pause the requests*/
    /* that have the paused tag.*/
    for (Iterator<BitmapHunter> it = hunterMap.values().iterator(); it.hasNext();) {
      BitmapHunter hunter = it.next();
      boolean loggingEnabled = hunter.getPicasso().loggingEnabled;

      Action single = hunter.getAction();
      List<Action> joined = hunter.getActions();
      boolean hasMultiple = joined != null && !joined.isEmpty();
      /* Hunter has no requests, bail early.*/
      if (single == null && !hasMultiple) {
        continue;
      }

      if (single != null && single.getTag().equals(tag)) {
        hunter.detach(single);
        pausedActions.put(single.getTarget(), single);
        if (loggingEnabled) {
          log(OWNER_DISPATCHER, VERB_PAUSED, single.request.logId(),
              "because tag '" + tag + "' was paused");
        }
      }

      if (hasMultiple) {
        for (int i = joined.size() - 1; i >= 0; i--) {
          Action action = joined.get(i);
          if (!action.getTag().equals(tag)) {
            continue;
          }

          hunter.detach(action);
          pausedActions.put(action.getTarget(), action);
          if (loggingEnabled) {
            log(OWNER_DISPATCHER, VERB_PAUSED, action.request.logId(),
                "because tag '" + tag + "' was paused");
          }
        }
      }

      /*/ Check if the hunter can be cancelled in case all its requests*/
      /* had the tag being paused here.*/
      if (hunter.cancel()) {
        it.remove();
        if (loggingEnabled) {
          log(OWNER_DISPATCHER, VERB_CANCELED, getLogIdsForHunter(hunter), "all actions paused");
        }
      }
    }
  }

```
* 方法中将Tag缓存在pausedActions这个Map中
* 设置了暂停后,如果请求在加载过程中,会被停止。我们可以看``performSubmit``方法
```java
if (pausedTags.contains(action.getTag())) {
      pausedActions.put(action.getTarget(), action);
      if (action.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_PAUSED, action.request.logId(),
            "because tag '" + action.getTag() + "' is paused");
      }
      return;
    }
```
* 最先判断pausedTags中是否存在Action携带的tag,如果存在的话,把Action缓存到pausedActions中,便于恢复,接着就直接return;从而实现了不加载图片。
* 暂停的Action被缓存到了pausedActions中,因此当我们要恢复的时候,就可以去该map中拿去。
* 恢复tag的调用路程的跟暂停基本是同一个套路,最后``performResumeTag(Object tag)``
```java
void performResumeTag(Object tag) {
    /* Trying to resume a tag that is not paused.*/
    if (!pausedTags.remove(tag)) {
      return;
    }

    List<Action> batch = null;
    for (Iterator<Action> i = pausedActions.values().iterator(); i.hasNext();) {
      Action action = i.next();
      if (action.getTag().equals(tag)) {
        if (batch == null) {
          batch = new ArrayList<Action>();
        }
        batch.add(action);
        i.remove();
      }
    }

    if (batch != null) {
      mainThreadHandler.sendMessage(mainThreadHandler.obtainMessage(REQUEST_BATCH_RESUME, batch));
    }
  }
```
* 通过遍历从``pausedActions``中取出Actions,再发送一条恢复的消息,最后会调用到``resumeAction(Action action)``
```java
void resumeAction(Action action) {
    Bitmap bitmap = null;
    if (shouldReadFromMemoryCache(action.memoryPolicy)) {
      bitmap = quickMemoryCacheCheck(action.getKey());
    }

    if (bitmap != null) {
      /* Resumed action is cached, complete immediately.*/
      deliverAction(bitmap, MEMORY, action);
      if (loggingEnabled) {
        log(OWNER_MAIN, VERB_COMPLETED, action.request.logId(), "from " + MEMORY);
      }
    } else {
      /* Re-submit the action to the executor.*/
      enqueueAndSubmit(action);
      if (loggingEnabled) {
        log(OWNER_MAIN, VERB_RESUMED, action.request.logId());
      }
    }
  }

```
* 首先会尝试从内存缓存中读取Bitmap,不存在的话,再调用``enqueueAndSubmit(action);``去排队加载图片,过程在上一篇中已经分析了,这里不多介绍。
* 值得注意的是,当图片正在加载和暂停加载图片时,Picasso会持有tag,如果我们设置的tag是Activity或者Fragment的话,如果不Activity或者Fragment销毁时,如果不取消tag的话,就会造成内存泄露。
* 总结:
 * 如果我们要设置tag的话,最好就是设置为Activity或者Fragment,因为系统管理了它们的声明周期,我们只要跟随它们的生命周期来调用暂停、恢复、取消操作就行。
 * 在Activity或者Fragment暂停时,应当调用pauseTag,位于前台的时候再resumeTag,当Activity或者Fragment销毁时调用cancelTag释放持有的tag对象等一系列操作。

#### Picasso中对不同类型的请求的处理

* 我们知道Picasso不仅支持从网络上加载图片,还支持从Drawable,Asset,File等中加载图片。
* 那么Picasso是如何处理?

##### 定义基类RequestHandler

* Picasso中定义了抽象类RequestHandler,也就是请求处理器来。在请求处理器中,定义了canHandleRequest(Request data)方法让子类实现,同时也定义了loadload(Request request, int networkPolicy)让子类实现。
* 请求处理器派生而来的有七种处理器,这里只分析网络请求处理器,主要分析``canHandleRequest(Request data)``
```java
@Override public boolean canHandleRequest(Request data) {
    String scheme = data.uri.getScheme();
    return (SCHEME_HTTP.equals(scheme) || SCHEME_HTTPS.equals(scheme));
  }

```
* 通过判断URI的scheme是不是HTTP或者HTTPS,是的话,返回true,证明,该类型的请求在网络处理器的处理范围内。
* 接下来,我们来看看Picasso是如何进行请求处理器的筛选的,通过上一篇分析,我们知道,请求处理器的筛选是在forRequest方法中:
```java
static BitmapHunter forRequest(Picasso picasso, Dispatcher dispatcher, Cache cache, Stats stats,
      Action action) {
    Request request = action.getRequest();
    List<RequestHandler> requestHandlers = picasso.getRequestHandlers();
    /* Index-based loop to avoid allocating an iterator.*/
    /*noinspection ForLoopReplaceableByForEach*/
    for (int i = 0, count = requestHandlers.size(); i < count; i++) {
      RequestHandler requestHandler = requestHandlers.get(i);
      if (requestHandler.canHandleRequest(request)) {
        return new BitmapHunter(picasso, dispatcher, cache, stats, action, requestHandler);
      }
    }
    return new BitmapHunter(picasso, dispatcher, cache, stats, action, ERRORING_HANDLER);
  }
```
* 代码中的逻辑是这样的:先通过Picasso拿到预先实例化好的请求处理器集合,接着通过遍历来拿到请求处理器,拿到后再调用其``canHandleRequest(request)``方法,从而筛选出能处理这条请求的请求处理器。

#### Picasso中的BitmapHunter队列控制

* Picasso中的是在BitmapBunter将存放在List集合的基础上来控制队列的
* 在BitmapHunter的run方法中会将BitmapHunter通过分发器回调回分发器中,接着还是按照之前的老套路,通过Handler发送消息,最后来到``batch(BitmapHunter)``中
```java
private void batch(BitmapHunter hunter) {
    if (hunter.isCancelled()) {
      return;
    }
    batch.add(hunter);
    if (!handler.hasMessages(HUNTER_DELAY_NEXT_BATCH)) {
      handler.sendEmptyMessageDelayed(HUNTER_DELAY_NEXT_BATCH, BATCH_DELAY);
    }
  }
```
* 方法中将BitmapHunter存放入List,接着通过通过Handler发送消息
* if()语句的判断和句内的发送消息是整个队列的控制精华。为什么这么说?handler.hasMessages(HUNTER_DELAY_NEXT_BATCH)如果返回false的话,证明没有请求在处理,所以直接发送一条延迟消息,如果返回true,那么不会进入if语句之内,也就不会被分发出去,只是被添加到了List集合内,等带下次有BitmapHunter进来是,且没有图片正在加载,才一起发送出去。而且发送的消息是发送延迟消息,这样也给了执行请求的一定的缓冲时间。
* List<BitmapHunter>最终会被发送到Picasso中运行在主线程的Handler
```java
 case HUNTER_BATCH_COMPLETE: {
          @SuppressWarnings("unchecked") List<BitmapHunter> batch = (List<BitmapHunter>) msg.obj;
          /*noinspection ForLoopReplaceableByForEach*/
          for (int i = 0, n = batch.size(); i < n; i++) {
            BitmapHunter hunter = batch.get(i);
            hunter.picasso.complete(hunter);
          }
          break;
        }
```
* 最后通过遍历每个BitmapHunter来设置到Target上。

#### Picasso中的设计
* Picasso中的设计思路是将加载图片这个任务细分出来许多个小任务,分配给对应的对象,比如:请求处理器,内存缓存器,下载器。
* 既然有这么对人干活,Picasso设计了分发器来指挥这些人干活。
* 分发器听谁的指令去指挥别人? 答案是Picasso,Picasso观察大局,有什么情况出现再分发器,分发器听到命令后,指挥对应的人员去工作,对应人员都干好各自的任务后,再告诉Dispatcher,最后Dispatcher汇报给Picasso。
* 整个框架能够井然有序的运行起来的关键是Handler,可以说Handler是整个框架能运行起来的能源。
* Picasso中,将Handler的作用发挥得淋漓尽致。是Handler运用的典例。
