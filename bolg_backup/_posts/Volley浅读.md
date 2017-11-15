---
title: Volley浅读
data: 2016/10/30
tags:
  - Android#网络框架
categories:
  - Android
---

#### Velloy浅谈
<!-- more -->
* 我们都知道，一般要使用Volley，先要创建该请求队列对象，我们并不是直接调用它的构造方法，而是调用Volley.newRequestQueue(COntext)来创建请求队列,那么我们下面来看看该方法“：

------------
```java

    /** Default on-disk cache directory. */
    private static final String DEFAULT_CACHE_DIR = "volley";

    /**
     * Creates a default instance of the worker pool and calls {@link RequestQueue#start()} on it.
     *
     * @param context A {@link Context} to use for creating the cache dir.
     * @param stack An {@link HttpStack} to use for the network, or null for default.
     * @return A started {@link RequestQueue} instance.
     */
    public static RequestQueue newRequestQueue(Context context, HttpStack stack) {
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();

            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }

        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network = new BasicNetwork(stack);

        RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        queue.start();

        return queue;
    }

    /**
     * Creates a default instance of the worker pool and calls {@link RequestQueue#start()} on it.
     *
     * @param context A {@link Context} to use for creating the cache dir.
     * @return A started {@link RequestQueue} instance.
     */
    public static RequestQueue newRequestQueue(Context context) {
        return newRequestQueue(context, null);
    }

```

* 从上面的代码可以看出，最终调用的是该重载方法newRequestQueue（Context, HttpStack）.我们来看看该方法中的代码：

```java
 if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network = new BasicNetwork(stack);

        RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        queue.start();

        return queue;
```

* 在if语句中判断SDK版本是否大于9，是的话实例化HurlStack（该类为HttpStack的具体实现类，该类访问网络是基于HttpURLConnection来实现的）
* 如果SDK版本小于9的话，实例化HttpClientStack（该类也是HttpStack的具体实现类，不过它的网络访问实现是基于HttpClient来实现的）
* 上面为什么要通过判断SDK版本判断实例化不同的具体实现类呢？ 这是有关到HttpClient和HttpURLConnection在不同的SDK版本的性能不同。
* 接着实例化BasicNetwork（实现了Network接口）
* 最终实例化ResquestQueue，并且调用start方法。


* 既然实际化的ＲｅｓｔｑｕｅｓｔＱｕｅｕｅ，并且调用了start，那么我们来看看start方法：

```java
/**
     * Starts the dispatchers in this queue.
     */
    public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
```

* 首先实例化了CacheDispatcher对象（该类继承了Thread，实际上就是一个线程），接着调用了start，也就是启动了线程（从名字来看，该线程为缓存线程）
* 再来看看for循环：在里面实例化了networkDispatcher数组（该类也是继承Thread，实际上也是一个线程），再分别调用的数组的start，也就是启动了线程，至于启动多少个线程？ 默认启动的是4个线程（数量可以在BasicNetwork中查看到，这里就不多说），并且从名字看，该线程可以理解为是网络线程。
* 从上面分析我们知道，当我们创建了一个请求队列，在后台默认就启动了5个线程（四个为网络线程，一个为缓存线程）
------------

* 我们创建好队列之后，就可以通过add（）方法来添加请求。我们接下来自然要来看看add：

```java
public Request add(Request request) {
        // Tag the request as belonging to this queue and add it to the set of current requests.
        request.setRequestQueue(this);
        synchronized (mCurrentRequests) {
            mCurrentRequests.add(request);
        }

        // Process requests in the order they are added.
        request.setSequence(getSequenceNumber());
        request.addMarker("add-to-queue");

        // If the request is uncacheable, skip the cache queue and go straight to the network.
        if (!request.shouldCache()) {
            mNetworkQueue.add(request);
            return request;
        }

        // Insert request into stage if there's already a request with the same cache key in flight.
        synchronized (mWaitingRequests) {
            String cacheKey = request.getCacheKey();
            if (mWaitingRequests.containsKey(cacheKey)) {
                // There is already a request in flight. Queue up.
                Queue<Request> stagedRequests = mWaitingRequests.get(cacheKey);
                if (stagedRequests == null) {
                    stagedRequests = new LinkedList<Request>();
                }
                stagedRequests.add(request);
                mWaitingRequests.put(cacheKey, stagedRequests);
                if (VolleyLog.DEBUG) {
                    VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
                }
            } else {
                // Insert 'null' queue for this cacheKey, indicating there is now a request in
                // flight.
                mWaitingRequests.put(cacheKey, null);
                mCacheQueue.add(request);
            }
            return request;
        }
    }
```

* 我们看到一开始，add（Request）就将该请求添加进Set队列中（关于Volley中的4个队列的配合使用，以后会单独拿出来分析一下）
* 再判断该请求能不能缓存，不能的话就跳过缓存队列，直接添加进网络队列
* 接下来，首先拿到请求的key值（也就是其url），判断等待队列中是不是有队列的key值，没有的话将该key值和null添加进等待队列，最后把该请求添加进缓存队列
* 如果有该key值的话通过key值拿出该key值对应的暂存请求队列，如果该队列为空的话，则实例化该队列，接着将含有相同的key（也就是url，相当于同一条请求），将该请求添加进暂存队列，并将该暂存队列添加进等待队列
* 上面的操作中并不是直接将该请求直接添加进缓存队列，而是通过将key值（uel）和暂存队列作为键值对来保存在一个等待队列中，这样的话，当有多条相同的请求添加进来的话，首先第一条会被添加存缓存队列中，接下来相同的请求则不会，而是都保存在暂存队列中。这样保证了相同的请求只会执行一次。

* 上面我们分析了将请求添加队列了，添加就去了就得被执行，那么会在哪里执行，当然是会在网络线程或者缓存线程中执行，那么下面我们分别来分析下这两个线程中run的处理

* 网络线程：

```java
@Override
    public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        Request request;
        while (true) {
            try {
                // Take a request from the queue.
                request = mQueue.take();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }

            try {
                request.addMarker("network-queue-take");

                // If the request was cancelled already, do not perform the
                // network request.
                if (request.isCanceled()) {
                    request.finish("network-discard-cancelled");
                    continue;
                }

                // Tag the request (if API >= 14)
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                    TrafficStats.setThreadStatsTag(request.getTrafficStatsTag());
                }

                // Perform the network request.
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                // If the server returned 304 AND we delivered a response already,
                // we're done -- don't deliver a second identical response.
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

                // Parse the response here on the worker thread.
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");

                // Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();
                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                mDelivery.postError(request, new VolleyError(e));
            }
        }
    }
```

* 在while（true）进入一个循环，首先从请求队列中拿出请求，接着判断该请求是不是有被取消，取消了话就不执行，没有取消的话，调用了mNetwork.performRequest(request)，在performRequest中，volley做了相关的优化操作，这也打算接下来单独拿出来分析; 我们进入performRequest看看：

```java
public interface Network {
    /**
     * Performs the specified request.
     * @param request Request to process
     * @return A {@link NetworkResponse} with data and caching metadata; will never be null
     * @throws VolleyError on errors
     */
    public NetworkResponse performRequest(Request<?> request) throws VolleyError;
}

```

* 发现它是个很熟悉的接口，没错，该接口的具体实现类是BasicNetwork，而该类我们在之前的newRequestQueue中已经实例化了，在该类中重写了performRequest：

```java
@Override
    public NetworkResponse performRequest(Request<?> request) throws VolleyError {
        long requestStart = SystemClock.elapsedRealtime();
        while (true) {
            HttpResponse httpResponse = null;
            byte[] responseContents = null;
            Map<String, String> responseHeaders = new HashMap<String, String>();
            try {
                // Gather headers.
                Map<String, String> headers = new HashMap<String, String>();
                addCacheHeaders(headers, request.getCacheEntry());
                httpResponse = mHttpStack.performRequest(request, headers);
                StatusLine statusLine = httpResponse.getStatusLine();
                int statusCode = statusLine.getStatusCode();

                responseHeaders = convertHeaders(httpResponse.getAllHeaders());
                // Handle cache validation.
                if (statusCode == HttpStatus.SC_NOT_MODIFIED) {
                    return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED,
                            request.getCacheEntry().data, responseHeaders, true);
                }

                responseContents = entityToBytes(httpResponse.getEntity());
                // if the request is slow, log it.
                long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
                logSlowRequests(requestLifetime, request, responseContents, statusLine);

                if (statusCode != HttpStatus.SC_OK && statusCode != HttpStatus.SC_NO_CONTENT) {
                    throw new IOException();
                }
                return new NetworkResponse(statusCode, responseContents, responseHeaders, false);
            } catch (SocketTimeoutException e) {
                attemptRetryOnException("socket", request, new TimeoutError());
            } catch (ConnectTimeoutException e) {
                attemptRetryOnException("connection", request, new TimeoutError());
            } catch (MalformedURLException e) {
                throw new RuntimeException("Bad URL " + request.getUrl(), e);
            } catch (IOException e) {
                int statusCode = 0;
                NetworkResponse networkResponse = null;
                if (httpResponse != null) {
                    statusCode = httpResponse.getStatusLine().getStatusCode();
                } else {
                    throw new NoConnectionError(e);
                }
                VolleyLog.e("Unexpected response code %d for %s", statusCode, request.getUrl());
                if (responseContents != null) {
                    networkResponse = new NetworkResponse(statusCode, responseContents,
                            responseHeaders, false);
                    if (statusCode == HttpStatus.SC_UNAUTHORIZED ||
                            statusCode == HttpStatus.SC_FORBIDDEN) {
                        attemptRetryOnException("auth",
                                request, new AuthFailureError(networkResponse));
                    } else {
                        // TODO: Only throw ServerError for 5xx status codes.
                        throw new ServerError(networkResponse);
                    }
                } else {
                    throw new NetworkError(networkResponse);
                }
            }
        }
    }
```

* 该重写方法中代码很长，大多数都是网络请求的细节，我们仔细看会发现该语句：httpResponse = mHttpStack.performRequest(request, headers);
* 该语句也调用了performRequest，类似的，HttpStack也是一个接口，实现类有HurlStack，HttpClientStack，这两个类中分别重写了performRequest，并且返回了HttpResponse
* 返回的HttpResponse在BasicNetwork中封装返回一个NetworkReponse对象，再回到NetworkDisoatcher中，我们看看下面这条语句：

```java
Response<?> response = request.parseNetworkResponse(networkResponse);
```

* 返回的NetworkReponse被request的parseNetworkReponse转为一个Response对应。Request是一个抽象类，该抽象类的具体实现类有JsonObjectRequest，StringRequest，，这几个都是Volley提供给我们的。在这些类中，重写了parseNetworkReponse方法，我们用StringRequest来举例。

```java
public class StringRequest extends Request<String> {
    private final Listener<String> mListener;

    /**
     * Creates a new request with the given method.
     *
     * @param method the request {@link Method} to use
     * @param url URL to fetch the string at
     * @param listener Listener to receive the String response
     * @param errorListener Error listener, or null to ignore errors
     */
    public StringRequest(int method, String url, Listener<String> listener,
            ErrorListener errorListener) {
        super(method, url, errorListener);
        mListener = listener;
    }

    /**
     * Creates a new GET request.
     *
     * @param url URL to fetch the string at
     * @param listener Listener to receive the String response
     * @param errorListener Error listener, or null to ignore errors
     */
    public StringRequest(String url, Listener<String> listener, ErrorListener errorListener) {
        this(Method.GET, url, listener, errorListener);
    }

    @Override
    protected void deliverResponse(String response) {
        mListener.onResponse(response);
    }

    @Override
    protected Response<String> parseNetworkResponse(NetworkResponse response) {
        String parsed;
        try {
            parsed = new String(response.data, HttpHeaderParser.parseCharset(response.headers));
        } catch (UnsupportedEncodingException e) {
            parsed = new String(response.data);
        }
        return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response));
    }
}
```

* 该类代码并不多，主要是重写了parseNetworkResponse，并且返回一个Response对象，并且重写deliverResponse方法，将该Response回调。如果我们要定制自己的Request（比如GsonRequest，JackSonRequest）的话，我们也可以通过继承Request并且重写两个方法

＊　好，我们接着继续回到ＮｅｔｗｏｒｋＤｉｓｐａｔｃｈｅｒ中

```java
mDelivery.postResponse(request, response);
```

* 该语句调用了postResponse，[ResponseDelivery](psi_element://com.android.volley.ResponseDelivery)是一个接口，该接口的具体具体实现类是ExecutorDeliver，我们下面来看看其操作：

```java
/**
 * Delivers responses and errors.
 */
public class ExecutorDelivery implements ResponseDelivery {
    /** Used for posting responses, typically to the main thread. */
    private final Executor mResponsePoster;

    /**
     * Creates a new response delivery interface.
     * @param handler {@link Handler} to post responses on
     */
    public ExecutorDelivery(final Handler handler) {
        // Make an Executor that just wraps the handler.
        mResponsePoster = new Executor() {
            @Override
            public void execute(Runnable command) {
                handler.post(command);
            }
        };
    }

    /**
     * Creates a new response delivery interface, mockable version
     * for testing.
     * @param executor For running delivery tasks
     */
    public ExecutorDelivery(Executor executor) {
        mResponsePoster = executor;
    }

    @Override
    public void postResponse(Request<?> request, Response<?> response) {
        postResponse(request, response, null);
    }

    @Override
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
        request.markDelivered();
        request.addMarker("post-response");
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
    }

    @Override
    public void postError(Request<?> request, VolleyError error) {
        request.addMarker("post-error");
        Response<?> response = Response.error(error);
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, null));
    }

    /**
     * A Runnable used for delivering network responses to a listener on the
     * main thread.
     */
    @SuppressWarnings("rawtypes")
    private class ResponseDeliveryRunnable implements Runnable {
        private final Request mRequest;
        private final Response mResponse;
        private final Runnable mRunnable;

        public ResponseDeliveryRunnable(Request request, Response response, Runnable runnable) {
            mRequest = request;
            mResponse = response;
            mRunnable = runnable;
        }

        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            // If this request has canceled, finish it and don't deliver.
            if (mRequest.isCanceled()) {
                mRequest.finish("canceled-at-delivery");
                return;
            }

            // Deliver a normal response or error, depending.
            if (mResponse.isSuccess()) {
                mRequest.deliverResponse(mResponse.result);
            } else {
                mRequest.deliverError(mResponse.error);
            }

            // If this is an intermediate response, add a marker, otherwise we're done
            // and the request can be finished.
            if (mResponse.intermediate) {
                mRequest.addMarker("intermediate-response");
            } else {
                mRequest.finish("done");
            }

            // If we have been provided a post-delivery runnable, run it.
            if (mRunnable != null) {
                mRunnable.run();
            }
       }
    }
}

```

* 该类中在线程池中包装了一个运行在主线程的Handler
* 我们直接先来看看重写的方法：

```java
 @Override
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
        request.markDelivered();
        request.addMarker("post-response");
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
    }
```

* 在最后一句中，传入了一个重写的Runable对象，在run方法中，首先判断是不是被取消了，使得话就不执行。
* 接着如果Response被标记为success的话，调用mRequest.deliverResponse(mResponse.result); 也就是你我们在StringRequest中重写的方法，而该线程池是在实例化请求队列的时候实例的。
* 分析到这里，Velloy是怎么将数据回调回主线程的？ 我们来看看实例化RequestQueue：

```java
/**
     * Creates the worker pool. Processing will not begin until {@link #start()} is called.
     *
     * @param cache A Cache to use for persisting responses to disk
     * @param network A Network interface for performing HTTP requests
     * @param threadPoolSize Number of network dispatcher threads to create
     */
    public RequestQueue(Cache cache, Network network, int threadPoolSize) {
        this(cache, network, threadPoolSize,
                new ExecutorDelivery(new Handler(Looper.getMainLooper())));
    }
```

* 我们在之前调用newRequestQueue的时候，我们实例化了RequestQueue，最终会调用到上面的重载方法，该方法中在ExecutorDelivery中传入了一个运行在主线程的Handler。那么我们进入ExecutorDelivery的构造函数：

```java
public ExecutorDelivery(final Handler handler) {
        // Make an Executor that just wraps the handler.
        mResponsePoster = new Executor() {
            @Override
            public void execute(Runnable command) {
                handler.post(command);
            }
        };
    }
```

* 在构造函数内，重写了execte（Runnable），通过传入运行在主线程的Handler将runnable post到主线程中的消息队列,这样就将请求的结果回到会主线程处理

-----------

* 上面分析那么多了，我们来总结一个网络线程的工作方式：
* 首先判断请求是不是被取消了，没有的话，接着调用BasicNetwork中的重写的performRequest，在该方法中有调用了HttpStack中的performRequest（该接口的具体实现类是HurlStack，HttpClientStack），在具体的实现类中处理网络细节，接着返回HttpResponse对象，回到BasicNetwork中，处理网络细节，最后返回NetworResponse对象
* 在NetworkDispatcher中得到返回的NetworkResponse对象，接着将之传入Request中的parseNetworkResponse中（该接口的具体实现类是StringRequest之类的），再得到Response对象，最后通过ResponseDeliver（具体实现类是ExecutorDeliver）将Response回调到主线程中处理。

* 缓存线程

```java
@Override
    public void run() {
        if (DEBUG) VolleyLog.v("start new dispatcher");
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

        // Make a blocking call to initialize the cache.
        mCache.initialize();

        while (true) {
            try {
                // Get a request from the cache triage queue, blocking until
                // at least one is available.
                final Request request = mCacheQueue.take();
                request.addMarker("cache-queue-take");

                // If the request has been canceled, don't bother dispatching it.
                if (request.isCanceled()) {
                    request.finish("cache-discard-canceled");
                    continue;
                }

                // Attempt to retrieve this item from cache.
                Cache.Entry entry = mCache.get(request.getCacheKey());
                if (entry == null) {
                    request.addMarker("cache-miss");
                    // Cache miss; send off to the network dispatcher.
                    mNetworkQueue.put(request);
                    continue;
                }

                // If it is completely expired, just send it to the network.
                if (entry.isExpired()) {
                    request.addMarker("cache-hit-expired");
                    request.setCacheEntry(entry);
                    mNetworkQueue.put(request);
                    continue;
                }

                // We have a cache hit; parse its data for delivery back to the request.
                request.addMarker("cache-hit");
                Response<?> response = request.parseNetworkResponse(
                        new NetworkResponse(entry.data, entry.responseHeaders));
                request.addMarker("cache-hit-parsed");

                if (!entry.refreshNeeded()) {
                    // Completely unexpired cache hit. Just deliver the response.
                    mDelivery.postResponse(request, response);
                } else {
                    // Soft-expired cache hit. We can deliver the cached response,
                    // but we need to also send the request to the network for
                    // refreshing.
                    request.addMarker("cache-hit-refresh-needed");
                    request.setCacheEntry(entry);

                    // Mark the response as intermediate.
                    response.intermediate = true;

                    // Post the intermediate response back to the user and have
                    // the delivery then forward the request along to the network.
                    mDelivery.postResponse(request, response, new Runnable() {
                        @Override
                        public void run() {
                            try {
                                mNetworkQueue.put(request);
                            } catch (InterruptedException e) {
                                // Not much we can do about this.
                            }
                        }
                    });
                }

            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }
        }
    }
```

* 我们主要来看看run中。
* run方法中，首先从从队列中拿出缓存请求，接着判断是不是被取消，取消了的话就不处理，然后判断该请求是不是丢失了，是的话就添加进网络队列中，再接着判断该缓存请求是不是过期，过期的话就添加进网络队列。
* 如果都没有丢失和过期的话，下面的逻辑处=处理和网络线程处理大同小异，这里就不分析了

##### 体会：

 * 在源码中，处理缓存线程并不是直接添加进来，而是通过将相同的请求添加进暂存队列中。

 * 平常用接口，都是用来回调，而源码中，通过具体类和接口分离来实现，这样能达到解耦的目的，对以后的扩展也留下了很大的空间（只要实现接口，编写自己的操作，就可以在volley的基础扩展）

 * 各线程的责任分明，分层思想很清晰。将线程分为三种，在处理请求时三种线程配合。首先启用缓存线程，如果丢失或者过期的话，传给网络线程，接着将结果回调给主线程。

* 在上面的分析中并没有涉及到volley中4个队列的配合使用以及各个线程之间的配合，还有在网络访问的时候，volley内部做的优化。所以打算把这些单独拿出来分析.
