---
title: universal-image-loader源码浅读
data: 2016/11/26
tags:
  - 图片加载框架#universal-image-loader
categories:
  - Android
---
##### 概述
* UIL是一款经典的图片加载框架，该类库的设计运用了多种设计模式，使得它的可拓展性增加，比如缓存的策略，如果默认的缓存策略不能够满足你的需求的话，你完全可以自己定制自己的缓存策略。
* 使用者不必关心加载图片时会发生OOM，其中发生的概率还是很小的，因为类库中对图片进行了三级缓存，其中的内存缓存使用了双级缓存（强引用和弱引用）。
* 库中考虑到用户可能会使用ListView，GridView或者RecyclerView来展示图片，因此在库中提供了一个PauseOnScrollListener来控制滑动时是否要加载图片。
* 这次，简单来分析类库的使用流程
<!-- more -->

##### 初始化
* 使用过UIL的都知道，需要先在Application中初始化：

```java
 //创建默认的ImageLoader配置参数  
        ImageLoaderConfiguration configuration = ImageLoaderConfiguration  
                .createDefault(this);  

        //Initialize ImageLoader with configuration.  
        ImageLoader.getInstance().init(configuration);  
```

* 上面的代码是在Application中初始化UIL，ImageLoaderConfiguration.createDefault(this)是创建默认的参数配置
* 点入该方法中：

```java
public static ImageLoaderConfiguration createDefault(Context context) {
        return new Builder(context).build();
    }
```
* 可以看到，在方法中，new了一个Builder，也就是说用建造者模式来初始化默认配置
* 再进入build()中，发现其中会调initEmptyFieldsWithDefaultValues()来初始化默认配置

```java
private void initEmptyFieldsWithDefaultValues() {
            if (taskExecutor == null) {
                taskExecutor = DefaultConfigurationFactory
                        .createExecutor(threadPoolSize, threadPriority, tasksProcessingType);
            } else {
                customExecutor = true;
            }
            if (taskExecutorForCachedImages == null) {
                taskExecutorForCachedImages = DefaultConfigurationFactory
                        .createExecutor(threadPoolSize, threadPriority, tasksProcessingType);
            } else {
                customExecutorForCachedImages = true;
            }
            if (diskCache == null) {
                if (diskCacheFileNameGenerator == null) {
                    diskCacheFileNameGenerator = DefaultConfigurationFactory.createFileNameGenerator();
                }
                diskCache = DefaultConfigurationFactory
                        .createDiskCache(context, diskCacheFileNameGenerator, diskCacheSize, diskCacheFileCount);
            }
            if (memoryCache == null) {
                memoryCache = DefaultConfigurationFactory.createMemoryCache(memoryCacheSize);
            }
            if (denyCacheImageMultipleSizesInMemory) {
                memoryCache = new FuzzyKeyMemoryCache(memoryCache, MemoryCacheUtils.createFuzzyKeyComparator());
            }
            if (downloader == null) {
                downloader = DefaultConfigurationFactory.createImageDownloader(context);
            }
            if (decoder == null) {
                decoder = DefaultConfigurationFactory.createImageDecoder(writeLogs);
            }
            if (defaultDisplayImageOptions == null) {
                defaultDisplayImageOptions = DisplayImageOptions.createSimple();
            }
        }
    }

```
* 该方法内会判断使用者是否有自定义的参数，没有的话，就在默认的参数工厂DefaultConfigurationFactory中创建对应的配置实例
* 至于配置参数是什么，这里就不详细讲

* 初始化完配置参数后，会调用Image Loader的init方法来初始化imageLoader，下面来看看init中做了那些初始化操作：

```java
public synchronized void init(ImageLoaderConfiguration configuration) {
        if (configuration == null) {
            throw new IllegalArgumentException(ERROR_INIT_CONFIG_WITH_NULL);
        }
        if (this.configuration == null) {
            L.d(LOG_INIT_CONFIG);
            engine = new ImageLoaderEngine(configuration);
            this.configuration = configuration;
        } else {
            L.w(WARNING_RE_INIT_CONFIG);
        }
    }
```
* init方法中实例化了ImageLoaderEngine对象，该类的作用主要是用来控制任务的执行

##### displayImage
* 初始化完成后，我们就可以调用displayImage来展示图片，该方法有多个重载方法，不过最后都会调用到一下的重载方法：

```java
public void displayImage(String uri, ImageAware imageAware, DisplayImageOptions options,
            ImageLoadingListener listener, ImageLoadingProgressListener progressListener) {
        checkConfiguration();
        if (imageAware == null) {
            throw new IllegalArgumentException(ERROR_WRONG_ARGUMENTS);
        }
        if (listener == null) {
            listener = emptyListener;
        }
        if (options == null) {
            options = configuration.defaultDisplayImageOptions;
        }

        if (TextUtils.isEmpty(uri)) {
            engine.cancelDisplayTaskFor(imageAware);
            listener.onLoadingStarted(uri, imageAware.getWrappedView());
            if (options.shouldShowImageForEmptyUri()) {
                imageAware.setImageDrawable(options.getImageForEmptyUri(configuration.resources));
            } else {
                imageAware.setImageDrawable(null);
            }
            listener.onLoadingComplete(uri, imageAware.getWrappedView(), null);
            return;
        }

        ImageSize targetSize = ImageSizeUtils.defineTargetSizeForView(imageAware, configuration.getMaxImageSize());
        String memoryCacheKey = MemoryCacheUtils.generateKey(uri, targetSize);
        engine.prepareDisplayTaskFor(imageAware, memoryCacheKey);

        listener.onLoadingStarted(uri, imageAware.getWrappedView());

        Bitmap bmp = configuration.memoryCache.get(memoryCacheKey);
        if (bmp != null && !bmp.isRecycled()) {
            L.d(LOG_LOAD_IMAGE_FROM_MEMORY_CACHE, memoryCacheKey);

            if (options.shouldPostProcess()) {
                ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
                        options, listener, progressListener, engine.getLockForUri(uri));
                ProcessAndDisplayImageTask displayTask = new ProcessAndDisplayImageTask(engine, bmp, imageLoadingInfo,
                        defineHandler(options));
                if (options.isSyncLoading()) {
                    displayTask.run();
                } else {
                    engine.submit(displayTask);
                }
            } else {
                options.getDisplayer().display(bmp, imageAware, LoadedFrom.MEMORY_CACHE);
                listener.onLoadingComplete(uri, imageAware.getWrappedView(), bmp);
            }
        } else {
            if (options.shouldShowImageOnLoading()) {
                imageAware.setImageDrawable(options.getImageOnLoading(configuration.resources));
            } else if (options.isResetViewBeforeLoading()) {
                imageAware.setImageDrawable(null);
            }

            ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
                    options, listener, progressListener, engine.getLockForUri(uri));
            LoadAndDisplayImageTask displayTask = new LoadAndDisplayImageTask(engine, imageLoadingInfo,
                    defineHandler(options));
            if (options.isSyncLoading()) {
                displayTask.run();
            } else {
                engine.submit(displayTask);
            }
        }
    }
```

* 在看方法的代码之前，先来看看其中的参数
* uri：图片的路径
* ImageAware：该接口的具体实现类会将ImageView包装成弱引用，这样使得GC能够及时回收ImageView，具体的实现类这里也不做多分析
* DisplayImageOptions：展示图片时的参数配置，比如：默认的加载图片，加载失败展示的图片，是否开启内存缓存，磁盘缓存等
* ImageLoadingListener：加载图片过程中的监听接口，类库中有提供缺省的类，方便我们重写其中的任何一个方法
* ImageLoadingProgressListener：加载的进度监听接口

* OK，解释完上面的参数，我们来看看方法：
* 首先会先调用checkConfiguration来见检查Configuration是否已经初始化，没有的话，抛出异常

##### 传入的uri为空时的操作

* 接着我们来分步剖析该方法，先来看下面的一段代码：

```java
if (TextUtils.isEmpty(uri)) {
            engine.cancelDisplayTaskFor(imageAware);
            listener.onLoadingStarted(uri, imageAware.getWrappedView());
            if (options.shouldShowImageForEmptyUri()) {
                imageAware.setImageDrawable(options.getImageForEmptyUri(configuration.resources));
            } else {
                imageAware.setImageDrawable(null);
            }
            listener.onLoadingComplete(uri, imageAware.getWrappedView(), null);
            return;
        }
```

* 上面的代码的逻辑是对传入的uri为null时的处理
* 首先调用engine将缓存的imageAware的key值取消掉，接着如果有设置默认展示的图片的话，就展示，没有的话就直接调用回调接口的onLoadingComplte来表示展示完图片

* 下面来看看传入uri不为空的处理：

```java
    ImageSize targetSize = ImageSizeUtils.defineTargetSizeForView(imageAware, configuration.getMaxImageSize());
        String memoryCacheKey = MemoryCacheUtils.generateKey(uri, targetSize);
        engine.prepareDisplayTaskFor(imageAware, memoryCacheKey);

        listener.onLoadingStarted(uri, imageAware.getWrappedView());

        Bitmap bmp = configuration.memoryCache.get(memoryCacheKey);
```
* 先获得Image的大小，接着生成image的缓存key值，最后调用engine.prepareDisplayTaskFor(imageAware, memoryCacheKey);将image的key缓存在map中（在执行完后会被移除出缓存），最后从缓存中取出bitmap。

* 接下来，对该bitmap进行分情况的处理：存在缓存和不存在缓存

##### 从内存中取出的Bitmap为空的操作
* 不存在缓存中，也就是bitmap为null：

```java
if (options.shouldShowImageOnLoading()) {
                imageAware.setImageDrawable(options.getImageOnLoading(configuration.resources));
            } else if (options.isResetViewBeforeLoading()) {
                imageAware.setImageDrawable(null);
            }

            ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
                    options, listener, progressListener, engine.getLockForUri(uri));
            LoadAndDisplayImageTask displayTask = new LoadAndDisplayImageTask(engine, imageLoadingInfo,
                    defineHandler(options));
            if (options.isSyncLoading()) {
                displayTask.run();
            } else {
                engine.submit(displayTask);
            }
```

* 在这种情况下，会先判断是否有设置加载图片过程中要展示的图片和图片在加载的过程中是否有被重新设置，有的话分别做相应的操作
* 接着将展示图片需要用到的配置参数包装成一个ImageLoadingInfo对象
* 再紧接着实例化LoadAndDisplayImageTask，该类的作用主要是从网络上或者文件系统中加载图片
* 最后判断是否需要同步加载，是的话在当前线程中run，否则调用engine中的submit提交一个任务

* 下面我们来分别对同步和异步加载两种情况进行分析：
##### 同步展示图片：

```java
@Override
    public void run() {
        if (waitIfPaused()) return;
        if (delayIfNeed()) return;

        ReentrantLock loadFromUriLock = imageLoadingInfo.loadFromUriLock;
        L.d(LOG_START_DISPLAY_IMAGE_TASK, memoryCacheKey);
        if (loadFromUriLock.isLocked()) {
            L.d(LOG_WAITING_FOR_IMAGE_LOADED, memoryCacheKey);
        }

        loadFromUriLock.lock();
        Bitmap bmp;
        try {
            checkTaskNotActual();

            bmp = configuration.memoryCache.get(memoryCacheKey);
            if (bmp == null || bmp.isRecycled()) {
                bmp = tryLoadBitmap();
                if (bmp == null) return; // listener callback already was fired

                checkTaskNotActual();
                checkTaskInterrupted();

                if (options.shouldPreProcess()) {
                    L.d(LOG_PREPROCESS_IMAGE, memoryCacheKey);
                    bmp = options.getPreProcessor().process(bmp);
                    if (bmp == null) {
                        L.e(ERROR_PRE_PROCESSOR_NULL, memoryCacheKey);
                    }
                }

                if (bmp != null && options.isCacheInMemory()) {
                    L.d(LOG_CACHE_IMAGE_IN_MEMORY, memoryCacheKey);
                    configuration.memoryCache.put(memoryCacheKey, bmp);
                }
            } else {
                loadedFrom = LoadedFrom.MEMORY_CACHE;
                L.d(LOG_GET_IMAGE_FROM_MEMORY_CACHE_AFTER_WAITING, memoryCacheKey);
            }

            if (bmp != null && options.shouldPostProcess()) {
                L.d(LOG_POSTPROCESS_IMAGE, memoryCacheKey);
                bmp = options.getPostProcessor().process(bmp);
                if (bmp == null) {
                    L.e(ERROR_POST_PROCESSOR_NULL, memoryCacheKey);
                }
            }
            checkTaskNotActual();
            checkTaskInterrupted();
        } catch (TaskCancelledException e) {
            fireCancelEvent();
            return;
        } finally {
            loadFromUriLock.unlock();
        }

        DisplayBitmapTask displayBitmapTask = new DisplayBitmapTask(bmp, imageLoadingInfo, engine, loadedFrom);
        runTask(displayBitmapTask, syncLoading, handler, engine);
    }
```
* 首先会做出两种判断：一种是是否暂停加载（库中有一个PauseOnScrollListener来控制是否要暂停加载图片）和是否要延迟进行
* 接下来尝试从内存内存缓存中取出Bitmap，如果Bitmap为null或者被回收了的话调用tryLoadBitmap
* 在tryLoadBitmap中会先尝试从硬盘缓存中取出存放Bitmap流的文件，如果存在的话返回获得的Bitmap，否则尝试从网络上加载图片，加载完成后顺便写进硬盘（如果设置了硬盘缓存的话）
* 再得到Bitmap以后回到run方法中，将bitmap写进内存缓存

* 如果bitmap存在缓存中的话，就直接使用该Bitmap

* 最后：

```java

        DisplayBitmapTask displayBitmapTask = new DisplayBitmapTask(bmp, imageLoadingInfo, engine, loadedFrom);
        runTask(displayBitmapTask, syncLoading, handler, engine);
```
* 包装成一个展示Bitmap的Task，并调用runTask来执行展示Image的Task

* 以上的就是同步时，展示Image的操作，下面我们来分析异步展示图片

##### 异步展示图片：
```java
/** Submits task to execution pool */
    void submit(final LoadAndDisplayImageTask task) {
        taskDistributor.execute(new Runnable() {
            @Override
            public void run() {
                File image = configuration.diskCache.get(task.getLoadingUri());
                boolean isImageCachedOnDisk = image != null && image.exists();
                initExecutorsIfNeed();
                if (isImageCachedOnDisk) {
                    taskExecutorForCachedImages.execute(task);
                } else {
                    taskExecutor.execute(task);
                }
            }
        });
    }
```
* 方法中有三个线程池：一：taskDistributor：任务分派线程池；二：taskExecutorForCachedImages缓存执行线程池；三：taskExecutor普通执行线程池
* 主要逻辑：通过从硬盘缓存中取出存放bitmap流的文件，如果该文件存在的话，在taskExecutorForCachedImages中执行传入的task，否则在普通线程池中执行task，至于怎么执行，我们在刚刚已经分析了。

##### 取出的Bitmap不为空的操作
* 回到displayImage中：
```java
if (bmp != null && !bmp.isRecycled()) {
            L.d(LOG_LOAD_IMAGE_FROM_MEMORY_CACHE, memoryCacheKey);

            if (options.shouldPostProcess()) {
                ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
                        options, listener, progressListener, engine.getLockForUri(uri));
                ProcessAndDisplayImageTask displayTask = new ProcessAndDisplayImageTask(engine, bmp, imageLoadingInfo,
                        defineHandler(options));
                if (options.isSyncLoading()) {
                    displayTask.run();
                } else {
                    engine.submit(displayTask);
                }
            } else {
                options.getDisplayer().display(bmp, imageAware, LoadedFrom.MEMORY_CACHE);
                listener.onLoadingComplete(uri, imageAware.getWrappedView(), bmp);
            }
        }
```
* 上面代码中一开始包装了ProcessAndDisplayImageTask对象，该对象主要是Bitmap存在缓存中的Task对象，接着判断是要异步执行还是同步执行，最后的操作跟前面分析的逻辑差不多，这里也不做分析了

##### 结尾：
* 上面主要分析了常用的一个displayImage而已，并且只是分析了一个调用流程，并没有分析其中的缓存策略还有其他的细节
* 在上面分析的流程中，我们可以清楚得体会ImageLoader中的分层思想。比如：对Bitmap存在内存与否分了两个展示图片的Task，执行不同类型的任务的线程池等。
