---
title: Picasso基本用法及源码浅析
data: 2017/1/30
tags:
  - 图片加载框架#Picasso
categories:
  - Android
---

### 基本用法：
#### 加载图片：

* picasso支持从Resource，MediaStore，content，contacts，url 加载图片。
```java
Picasso.with(this)
                .load(uri)
                .into(mImageView);
```
* 以上是最简单的通用加载图片的方法

<!-- more -->

* 添加加载错误以及加载中的展示图片
```java
Picasso.with(this)
                .load(uri)
                .error(R.drawable.error)
                .placeholder(R.drawable.placeholder)
                .into(mImageView);
```
* 如果加载得到的Bitmap的大小不符合我们设置的ImageView的大小，我们可以调用以下代码来调整：
```java
Picasso.with(this)
                .load(uri)
                .fit()
                .into(mImageView);
```
* 与fit()相似的是resize（int，int），两者都可以调整图片的大小，但是fit()只能用在ImageView上。需要注意的是两者不能同时使用。
* 我们还可以制定ImageView的模式
```java
Picasso.with(this)
                .load(R.drawable.cangjinyou)
                .resize(150, 150)
                .centerCrop()
                .into(mImageView);
```
```java
        Picasso.with(this)
                .load(R.drawable.cangjinyou)
                .resize(150, 150)
                .centerInside()
                .into(mImageView);
```
* 需要注意的是centerInside()和centerCrop()不能同时设置。

#### 设置Tag

* 加载图片的同时设置tag来为图片添加生命周期
* 添加tag
```java
Picasso.with(this)
                .load(R.drawable.cangjinyou)
                .fit()
                .centerInside()
                .tag(this)
                .into(mImageView);
```
* 设置tag可以为图片设置生命周期，通常这个tag可以为Activity或者Fragment，设置tag的好处就是可以根据tag的生命周期来管理图片的生命周期。在设置tag的同时需要避免内存泄露，因为当图片正在加载或者暂停加载的时候，Picasso会持有这个tag。
* 设置tag后，我们可以调用Picasso中的cancelTag（Object）,pauseTag（Object）和resumeTag（Object）来管理图片的生命周期。

#### 开启Debug模式和指示模式

* 开启后会在image左上方绘制一个小小小的三角形，三角形的颜色代表图片是从哪里加载得来的，绿色，蓝色，红色分别表示从内存，硬盘，网络加载得来
```java
 public void openDebug() {
        Picasso.with(this)
                .setLoggingEnabled(true);
    }

 public void openIndicator() {
        Picasso.with(this).setIndicatorsEnabled(true);
    }
```

### 源码解读
#### with(Context)创建Picasso全局单例

```java
public static Picasso with(Context context) {
    if (singleton == null) {
      synchronized (Picasso.class) {
        if (singleton == null) {
          singleton = new Builder(context).build();
        }
      }
    }
    return singleton;
  }
```
* 很明显上面的代码逻辑是使用DCL模式创建一个Picasso单例,但是创建对象的方式是使用建造者模式,这样做的好处就是可以让我们自己来配置Picasso。那么怎么配置Picasso?
1.使用Picasso.Builder创建配置;
2.设置配置完的Picasso: setSingletonInstance(Picasso picasso).
* 回到with()中,我们来看看 singleton = new Builder(context).build();中做了哪些初始化操作。
```java
/** Create the {@link Picasso} instance. */
    public Picasso build() {
      Context context = this.context;

      if (downloader == null) {
        downloader = Utils.createDefaultDownloader(context); /*创建下载器*/
      }

      if (cache == null) {
        cache = new LruCache(context); /*创建内存缓存器*/
      }

      if (service == null) {
        service = new PicassoExecutorService(); /*创建Picasso优化后的线程池*/
      }

      if (transformer == null) {
        transformer = RequestTransformer.IDENTITY; /*创建请求转换器*/
      }

      Stats stats = new Stats(cache); /*创建状态管理器*/

      Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats); /*创建分发器*/

      return new Picasso(context, dispatcher, cache, listener, transformer, requestHandlers, stats,
          defaultBitmapConfig, indicatorsEnabled, loggingEnabled); /*创建Picasso实例*/
    }
  }
```
* 从上面注释可以看到,Picasso将每个任务分派给不同的对象去处理,最后再指挥这些一系列的'器'去工作。那么下面,我们先来简单介绍以下他们。

##### 下载器

* 我们先来看看下载器的基础接口: Downloader
* 下载器内部提供两个方法,分别是load(Uri uri, int networkPolicy)和shutdown();load(Uri uri, int networkPolicy)方法是从网络上加载图片,返回内部类Response对象,该对象主要是储存着下载来的Bitmap或者字节流。
* 介绍完下载器基础接口,我们回到创建默认下载器的方法内部:
```java
static Downloader createDefaultDownloader(Context context) {
    try {
      Class.forName("com.squareup.okhttp.OkHttpClient"); /*检查是不是存在OkHttpClient,*/
      return OkHttpLoaderCreator.create(context);
    } catch (ClassNotFoundException ignored) {
    }
    return new UrlConnectionDownloader(context);
  }
```
* 首先检查环境中是不是存在Okhttp,存在的话证明已经添加了Okhttp依赖,因而直接使用Okhttp来创建下载器
* 如果不存直接创建UrlConnectionDownloader下载器

##### 内存缓存器
* 内存内存缓存器使用的是Lru,最大的缓存大小是堆内存的最大数的7分之一

##### 线程池
* 线程池中根据网络状态动态调整了线程数量,以及包装了一个FutureTask来进行实现优先级的比较

##### 请求转换器
* Picasso里面并没有进行特殊的请求转换,只是原封不动返回了一个请求

##### 状态管理器
* 对Picasso运转的情况做了记录,便于当发生oom时,打印出Picasso的运行情况

##### 分发器
* 上面几个'器'最终都是在分发器中被其指挥调度

##### Picasso实例
* 分发器指挥调度各个'器'去工作.那么分发器什么时候进行调度? 发送这个命令是就是Picasso, Picasso->Dispatcher->各个'器'

#### 调用load方法创建RequestCreator

* load方法有四个重载方法,用于加载不同的资源。但是最终目的都是创建一个RequestCreator
* 在介绍RequestCreator之前,我们先来看看Request,

##### Request
* 顾名思义,就是一条请求,Picasso中会将要加载的图片包装成一条请求

##### RequestCreator
* 请求的创建任务是交给RequestCreator
* RequestCreator可以在创建完请求后,把该请求设置到目标Target中

#### into
* into方法有五个重载方法,支持将加载得到的图片设置到ImageView,RemoteViews, Target(自定义View)
* 几个重载方法原理都是相同的,这里我只分析ImageView。
```java
public void into(ImageView target, Callback callback) {
    long started = System.nanoTime();
    checkMain(); /*保证into方法是在主线程中被调用的*/

    if (target == null) {
      throw new IllegalArgumentException("Target must not be null.");
    }

    if (!data.hasImage()) { /*如果在load()方法中有传入ResId或者uri的话,是不会进入if语句中的*/
      picasso.cancelRequest(target);
      if (setPlaceholder) {
        setPlaceholder(target, getPlaceholderDrawable());
      }
      return;
    }

    if (deferred) { /*deferred默认值为false,只有再调用了fit方法后deferred才为true,才会进入该方法*/
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
* 方法中的Callback接口可以不传入,如果要传入的话,要注意内存泄露。
* 结合代码中的注释,首先先创建根据当前时间戳来创建一个请求,接着再拼接请求的key
* 拼接完key之后,会判断是不是要从内存缓存中读取Bitmap,有的话,直接从内存拿走最后设置到target中;
* 如果设置了placeholder的话,会显示一张预加载的图片
* 最后实例化一个ImageViewAction并把这个action提交给picasso对象去处理。
* Action是一个对一个请求进行包装,里面对target进行弱引用包装,是的我们不必关心内存泄露.
* 我们接着看看看Picasso对传入的action进行了什么操作:
```java
void enqueueAndSubmit(Action action) {
    Object target = action.getTarget();
    if (target != null && targetToAction.get(target) != action) {
      /* This will also check we are on the main thread.*/
      cancelExistingRequest(target);
      targetToAction.put(target, action);
    }
    submit(action);
  }
```
* 上面逻辑很简单,这里不多说,我们直接看看submit:
```java
void submit(Action action) {
    dispatcher.dispatchSubmit(action);
  }

```
* 前面说过了,Picassso是一个总指挥,既然有action来了,Picasso肯定会给他的直接下属Dispatcher发命令,我们看看dispatcher对Picasso交给它的action怎么分派:
```java
void dispatchSubmit(Action action) {
    handler.sendMessage(handler.obtainMessage(REQUEST_SUBMIT, action));
  }
```
* 我们看到,在dispatchSubmit中,通过Handler发送一个REQUEST_SUBMIT消息,该Handler是Dispatcher的内部类
```java
@Override public void handleMessage(final Message msg) {
      switch (msg.what) {
        case REQUEST_SUBMIT: {
          Action action = (Action) msg.obj;
          dispatcher.performSubmit(action);
          break;
        }
```
* 接收到消息时,又调用了performSubmit(action)
```java
void performSubmit(Action action, boolean dismissFailed) {
    if (pausedTags.contains(action.getTag())) { /*判断action对应的tag是不是被设置为暂停了, 如果设置了暂停,直接去取消加载图片*/
      pausedActions.put(action.getTarget(), action);
      if (action.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_PAUSED, action.request.logId(),
            "because tag '" + action.getTag() + "' is paused");
      }
      return;
    }

    BitmapHunter hunter = hunterMap.get(action.getKey());
    if (hunter != null) {
      hunter.attach(action);
      return;
    }

    if (service.isShutdown()) {
      if (action.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_IGNORED, action.request.logId(), "because shut down");
      }
      return;
    }

    hunter = forRequest(action.getPicasso(), this, cache, stats, action);
    hunter.future = service.submit(hunter);
    hunterMap.put(action.getKey(), hunter);
    if (dismissFailed) {
      failedActions.remove(action.getTarget());
    }

    if (action.getPicasso().loggingEnabled) {
      log(OWNER_DISPATCHER, VERB_ENQUEUED, action.request.logId());
    }
  }

```
* 结合上面注释,我们直接来看看``hunter = forRequest(action.getPicasso(), this, cache, stats, action);``
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
* 既然有了请求,dispatcher就得来指挥对应的'人'来处理
* 接下来,通过遍历取得RequestHandler来能够处理该请求的RequestHandler来处理
* 那么这些RequestHandler怎么来的?
* 其实这些RequestHandler是创建Picasso的时候,就已经实例化了,我们暂且回到Picasso来看看:
```java
List<RequestHandler> allRequestHandlers =
        new ArrayList<RequestHandler>(builtInHandlers + extraCount);

    /* ResourceRequestHandler needs to be the first in the list to avoid*/
    /* forcing other RequestHandlers to perform null checks on request.uri*/
    /* to cover the (request.resourceId != 0) case.*/
    allRequestHandlers.add(new ResourceRequestHandler(context));
    if (extraRequestHandlers != null) {
      allRequestHandlers.addAll(extraRequestHandlers);
    }

    allRequestHandlers.add(new ContactsPhotoRequestHandler(context));
    allRequestHandlers.add(new MediaStoreRequestHandler(context));
    allRequestHandlers.add(new ContentStreamRequestHandler(context));
    allRequestHandlers.add(new AssetRequestHandler(context));
    allRequestHandlers.add(new FileRequestHandler(context));
    allRequestHandlers.add(new NetworkRequestHandler(dispatcher.downloader, stats));
    requestHandlers = Collections.unmodifiableList(allRequestHandlers);

```
* Picasso默认帮我们创建了7个请求处理器,这些处理器,分别对应响应的不同的请求类型,当然,Picasso也考虑到了这7中可能会不够我们使用,如果是这样的话,我们可以继承RequestHandler,并且调用Picasso.Builder.addRequestHandler(RequestHandler)}来扩展RequestHandler。

##### RequestHandler

* 请求处理器,其中预留了``canHandleRequest(Request data)``和``load(Request request, int networkPolicy)``给具体子类去实现。
* 继承RequestHandler的子类,``canHandleRequest(Request data)``判断该请求是不是在自己的处理范围内,是的话返回true,而load则是加载图片
* RequestHandler内部的Result是由load完的数据进行包装而成的对象。
* 介绍完RequestHandler,我们回到forResquest方法中:在for循环中,找到能处理该请求的RequestHandler并实例化返回一个BitmapHunter,
* 得到BitmapHunter,提交给线程池,得到一个Future,所以此时,我们得去PicassoExecutorService去看看:
```java
private static final class PicassoFutureTask extends FutureTask<BitmapHunter>
      implements Comparable<PicassoFutureTask> {
    private final BitmapHunter hunter;

    public PicassoFutureTask(BitmapHunter hunter) {
      super(hunter, null);
      this.hunter = hunter;
    }

    @Override
    public int compareTo(PicassoFutureTask other) {
      Picasso.Priority p1 = hunter.getPriority();
      Picasso.Priority p2 = other.hunter.getPriority();

      /* High-priority requests are "lesser" so they are sorted to the front.*/
      /* Equal priorities are sorted by sequence number to provide FIFO ordering.*/
      return (p1 == p2 ? hunter.sequence - other.hunter.sequence : p2.ordinal() - p1.ordinal());
    }
  }

```
* 将提交的BitmapHunter包装成一个PicassoFutureTask,主要的操作就是进行优先级的比较
* 提交完后,我们自然要去BitmapHunter中,因为BitmapHunter是继承自Runnable对象,提交的BitmapHunter会在其Run函数中执行
```java
 @Override public void run() {
    try {
      updateThreadName(data);

      if (picasso.loggingEnabled) {
        log(OWNER_HUNTER, VERB_EXECUTING, getLogIdsForHunter(this));
      }

      result = hunt();
      if (result == null) {
        dispatcher.dispatchFailed(this);
      } else {
        dispatcher.dispatchComplete(this);
      }
    } catch (Downloader.ResponseException e) {
      if (!e.localCacheOnly || e.responseCode != 504) {
        exception = e;
      }
      dispatcher.dispatchFailed(this);
    } catch (NetworkRequestHandler.ContentLengthException e) {
      exception = e;
      dispatcher.dispatchRetry(this);
    } catch (IOException e) {
      exception = e;
      dispatcher.dispatchRetry(this);
    } catch (OutOfMemoryError e) {
      StringWriter writer = new StringWriter();
      stats.createSnapshot().dump(new PrintWriter(writer));
      exception = new RuntimeException(writer.toString(), e);
      dispatcher.dispatchFailed(this);
    } catch (Exception e) {
      exception = e;
      dispatcher.dispatchFailed(this);
    } finally {
      Thread.currentThread().setName(Utils.THREAD_IDLE_NAME);
    }
  }
```
* 上面代码中主要调用了hunt()方法得到Bitmap对象,现在,我们直接进入该方法看看:
```java
Bitmap hunt() throws IOException {
    Bitmap bitmap = null;
    if (shouldReadFromMemoryCache(memoryPolicy)) {
      bitmap = cache.get(key);
      if (bitmap != null) {
        stats.dispatchCacheHit();
        loadedFrom = MEMORY;
        if (picasso.loggingEnabled) {
          log(OWNER_HUNTER, VERB_DECODED, data.logId(), "from cache");
        }
        return bitmap;
      }
    }

    data.networkPolicy = retryCount == 0 ? NetworkPolicy.OFFLINE.index : networkPolicy;
    RequestHandler.Result result = requestHandler.load(data, networkPolicy);
    if (result != null) {
      loadedFrom = result.getLoadedFrom();
      exifRotation = result.getExifOrientation();
      bitmap = result.getBitmap();

      /* If there was no Bitmap then we need to decode it from the stream.*/
      if (bitmap == null) {
        InputStream is = result.getStream();
        try {
          bitmap = decodeStream(is, data);
        } finally {
          Utils.closeQuietly(is);
        }
      }
    }

    if (bitmap != null) {
      if (picasso.loggingEnabled) {
        log(OWNER_HUNTER, VERB_DECODED, data.logId());
      }

      stats.dispatchBitmapDecoded(bitmap);
      if (data.needsTransformation() || exifRotation != 0) {
        synchronized (DECODE_LOCK) {
          if (data.needsMatrixTransform() || exifRotation != 0) {
            bitmap = transformResult(data, bitmap, exifRotation);
            if (picasso.loggingEnabled) {
              log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId());
            }
          }

          if (data.hasCustomTransformations()) {
            bitmap = applyCustomTransformations(data.transformations, bitmap);
            if (picasso.loggingEnabled) {
              log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId(), "from custom transformations");
            }
          }
        }
        if (bitmap != null) {
          stats.dispatchBitmapTransformed(bitmap);
        }
      }
    }

    return bitmap;
  }

```
* 首先判断内存缓存是不是存在该Bitmap,存在的话返回Bitmap对象
* 不存在的话,进行网络加载,进行网络加载时,会先判断硬盘缓存中是不是存在,不存在的话再从网络上下载
* 如果是从网络上下载的话,会得到字节流,接着解析字节流,最后返回该Bitmap。
* 得到Bitmap后,
```java
if (result == null) {
        dispatcher.dispatchFailed(this);
      } else {
        dispatcher.dispatchComplete(this);
      }
```
* 是分发器出场把BitmapHunter对象回调出去
* 回调的套路跟之前一样,通过Handler发送一条加载完成的消息,再会回调加载完成的方法
```java
void performComplete(BitmapHunter hunter) {
    if (shouldWriteToMemoryCache(hunter.getMemoryPolicy())) {
      cache.set(hunter.getKey(), hunter.getResult());
    }
    hunterMap.remove(hunter.getKey());
    batch(hunter);
    if (hunter.getPicasso().loggingEnabled) {
      log(OWNER_DISPATCHER, VERB_BATCHED, getLogIdsForHunter(hunter), "for completion");
    }

  }
```
* 最后会回调上面的方法,主要操作是缓存该Bitmap
* 缓存完Bitmap后,调用batch(hunter)
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
* 方法中,通过Hnadler发送一条消息,我们跟踪一下这条信息,最后会调用以下方法:
```java
 void performBatchComplete() {
    List<BitmapHunter> copy = new ArrayList<BitmapHunter>(batch);
    batch.clear();
    mainThreadHandler.sendMessage(mainThreadHandler.obtainMessage(HUNTER_BATCH_COMPLETE, copy));
    logBatch(copy);
  }
```
* 方法中也是通过Hnadler发送一条消息,而这次发送的消息是发送到运行在主线程的Hnadler,该Handler是实例化Picasso的时候传入Dispatcher的,因而我们自然来到Picasso中:
```java
void complete(BitmapHunter hunter) {
    Action single = hunter.getAction();
    List<Action> joined = hunter.getActions();

    boolean hasMultiple = joined != null && !joined.isEmpty();
    boolean shouldDeliver = single != null || hasMultiple;

    if (!shouldDeliver) {
      return;
    }

    Uri uri = hunter.getData().uri;
    Exception exception = hunter.getException();
    Bitmap result = hunter.getResult();
    LoadedFrom from = hunter.getLoadedFrom();

    if (single != null) {
      deliverAction(result, from, single);
    }

    if (hasMultiple) {
      /*noinspection ForLoopReplaceableByForEach*/
      for (int i = 0, n = joined.size(); i < n; i++) {
        Action join = joined.get(i);
        deliverAction(result, from, join);
      }
    }

    if (listener != null && exception != null) {
      listener.onImageLoadFailed(this, uri, exception);
    }
  }

```
* 通过分析,最终会回调Picasso的complete方法:方法中我们只分析单一的Action,也就是会调用``deliverAction(result, from, single);``
```java
private void deliverAction(Bitmap result, LoadedFrom from, Action action) {
    if (action.isCancelled()) {
      return;
    }
    if (!action.willReplay()) {
      targetToAction.remove(action.getTarget());
    }
    if (result != null) {
      if (from == null) {
        throw new AssertionError("LoadedFrom cannot be null.");
      }
      action.complete(result, from);
      if (loggingEnabled) {
        log(OWNER_MAIN, VERB_COMPLETED, action.request.logId(), "from " + from);
      }
    } else {
      action.error();
      if (loggingEnabled) {
        log(OWNER_MAIN, VERB_ERRORED, action.request.logId());
      }
    }
  }
```
* 上面代码,我们可以看到最后会调用``action.complete(result, from);``,因为我们实例化Action的时候是实例化ImageView,因此,我们来看看ImageViewAction中的complete:
```java
@Override public void complete(Bitmap result, Picasso.LoadedFrom from) {
    if (result == null) {
      throw new AssertionError(
          String.format("Attempted to complete action with no result!\n%s", this));
    }

    ImageView target = this.target.get();
    if (target == null) {
      return;
    }

    Context context = picasso.context;
    boolean indicatorsEnabled = picasso.indicatorsEnabled;
    PicassoDrawable.setBitmap(target, context, result, from, noFade, indicatorsEnabled);

    if (callback != null) {
      callback.onSuccess();
    }
  }
```
* 很明显,上面代码中调用了PicassoDrawable.setBitmap(target, context, result, from, noFade, indicatorsEnabled);设置来Bitmap,也就是说设置到ImageView中,有趣的是,PicassoDrawable是继承自BitmapDrawable的,并重写了onDraw方法,根据是否开启指示器模式,来判断是否要在图片的左上角画一个小小的三角形,其中不同颜色的三角形代表从Image的来源,这里具体不多说了。

### 总结
* 总体来说,Picasso的设计是将不同的任务分派给不同的'器',再有分发器调度,而分发器直接接受Picasso的命令,也就是说整个加载的过程中,Picasso始终存在
* Picasso内部的实现还是有很多地方值得学习,比如Action队列的控制,请求处理器的分类,线程池的优化,图片生命周期的管理等。既然Picasso有这么多值得学习的地方,所以打算专门写一篇来分析其中的技巧。
