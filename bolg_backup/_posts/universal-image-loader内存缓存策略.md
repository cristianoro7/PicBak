---
title: universal-image-loader缓存策略
data: 2016/11/26
tags:
  - 图片加载框架#universal-image-loader
categories:
  - Android
---
##### 概述
有看过UIL源码的人都应该知道，库中缓存使用了策略模式来对bitmap进行缓存。这次来单独来分析其中的内存缓存策略
<!-- more -->

##### 内存缓存
接口：
* MemoryCacheAware：底层接口，里面定义了基本的缓存操作方法
* MemoryCache：单纯继承了MemoryCacheAware接口

抽象类：
* BaseMemoryCache：实现了MemoryCahce，运用弱引用包装Bitmap后缓存存在HashMap中。同时也实现了MemoryCacheAware中的接口的方法
* LimitedMemoryCache：这个类继承BaseMemoryCahce，并在类中扩展了一级强引用缓存，也就是说LimitedMemoryCache是拥有双级缓存的（强引用缓存，弱引用缓存）。
 * LimitedMemoryCache会限制内存的缓存大小，当达到缓存达到最大的内存缓存时，会移除缓存，至于移除的策略，库中有提供几个实现策略。

具体实现类：
* WeakMeoryCache：

```java
public class WeakMemoryCache extends BaseMemoryCache {
    @Override
    protected Reference<Bitmap> createReference(Bitmap value) {
        return new WeakReference<Bitmap>(value);
    }
}
```
WeakMemoryCache是继承自BaseMemoryCache，并且只是重写了createReference，所有WeakMemoryCache只有弱引用的缓存。

* LruMemoryCache：

```java
public class LruMemoryCache implements MemoryCache {

    private final LinkedHashMap<String, Bitmap> map;

    private final int maxSize;
    /** Size of this cache in bytes */
    private int size;

    /** @param maxSize Maximum sum of the sizes of the Bitmaps in this cache */
    public LruMemoryCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<String, Bitmap>(0, 0.75f, true);
    }

    /**
     * Returns the Bitmap for {@code key} if it exists in the cache. If a Bitmap was returned, it is moved to the head
     * of the queue. This returns null if a Bitmap is not cached.
     */
    @Override
    public final Bitmap get(String key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        synchronized (this) {
            return map.get(key);
        }
    }

    /** Caches {@code Bitmap} for {@code key}. The Bitmap is moved to the head of the queue. */
    @Override
    public final boolean put(String key, Bitmap value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        synchronized (this) {
            size += sizeOf(key, value);
            Bitmap previous = map.put(key, value);
            if (previous != null) {
                size -= sizeOf(key, previous);
            }
        }

        trimToSize(maxSize);
        return true;
    }

    /**
     * Remove the eldest entries until the total of remaining entries is at or below the requested size.
     *
     * @param maxSize the maximum size of the cache before returning. May be -1 to evict even 0-sized elements.
     */
    private void trimToSize(int maxSize) {
        while (true) {
            String key;
            Bitmap value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName() + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize || map.isEmpty()) {
                    break;
                }

                Map.Entry<String, Bitmap> toEvict = map.entrySet().iterator().next();
                if (toEvict == null) {
                    break;
                }
                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= sizeOf(key, value);
            }
        }
    }

    /** Removes the entry for {@code key} if it exists. */
    @Override
    public final Bitmap remove(String key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        synchronized (this) {
            Bitmap previous = map.remove(key);
            if (previous != null) {
                size -= sizeOf(key, previous);
            }
            return previous;
        }
    }

    @Override
    public Collection<String> keys() {
        synchronized (this) {
            return new HashSet<String>(map.keySet());
        }
    }

    @Override
    public void clear() {
        trimToSize(-1); // -1 will evict 0-sized elements
    }

    /**
     * Returns the size {@code Bitmap} in bytes.
     * <p/>
     * An entry's size must not change while it is in the cache.
     */
    private int sizeOf(String key, Bitmap value) {
        return value.getRowBytes() * value.getHeight();
    }

    @Override
    public synchronized final String toString() {
        return String.format("LruCache[maxSize=%d]", maxSize);
    }
}
```
LruMemoryCache继承自MemoryCache，因此只有强引用的缓存，LruMemoryCache移除缓存的策略是使用最近最少使用算法，即每次通过get得到的Bitmap都会放在队列头，移除的时候就会先从队列尾移除，这样就保证了最少使用的最先被移除

* LimitedAgeMemoryCache：

```java
public class LimitedAgeMemoryCache implements MemoryCache {

    private final MemoryCache cache;

    private final long maxAge;
    private final Map<String, Long> loadingDates = Collections.synchronizedMap(new HashMap<String, Long>());

    /**
     * @param cache  Wrapped memory cache
     * @param maxAge Max object age <b>(in seconds)</b>. If object age will exceed this value then it'll be removed from
     *               cache on next treatment (and therefore be reloaded).
     */
    public LimitedAgeMemoryCache(MemoryCache cache, long maxAge) {
        this.cache = cache;
        this.maxAge = maxAge * 1000; // to milliseconds
    }

    @Override
    public boolean put(String key, Bitmap value) {
        boolean putSuccesfully = cache.put(key, value);
        if (putSuccesfully) {
            loadingDates.put(key, System.currentTimeMillis());
        }
        return putSuccesfully;
    }

    @Override
    public Bitmap get(String key) {
        Long loadingDate = loadingDates.get(key);
        if (loadingDate != null && System.currentTimeMillis() - loadingDate > maxAge) {
            cache.remove(key);
            loadingDates.remove(key);
        }

        return cache.get(key);
    }

    @Override
    public Bitmap remove(String key) {
        loadingDates.remove(key);
        return cache.remove(key);
    }

    @Override
    public Collection<String> keys() {
        return cache.keys();
    }

    @Override
    public void clear() {
        cache.clear();
        loadingDates.clear();
    }
}
```
* LimitedAgeMemoryCache是MemoryCache的装饰类，也就是说LimitedAgeMemoryCache可以装饰实现了MemoryCache接口的类，比如LruMemoryCache。那么装饰的新功能是什么？
* 通过上面源码，可以清楚的得出，LimitedAgeMemoryCache是通过判断Bitmap的缓存时间是否超过在初始化传入时的缓存最大时间。超过的话就从内存中移除。

* FuzzyKeyMemoryCache：

```java
public class FuzzyKeyMemoryCache implements MemoryCache {

    private final MemoryCache cache;
    private final Comparator<String> keyComparator;

    public FuzzyKeyMemoryCache(MemoryCache cache, Comparator<String> keyComparator) {
        this.cache = cache;
        this.keyComparator = keyComparator;
    }

    @Override
    public boolean put(String key, Bitmap value) {
        // Search equal key and remove this entry
        synchronized (cache) {
            String keyToRemove = null;
            for (String cacheKey : cache.keys()) {
                if (keyComparator.compare(key, cacheKey) == 0) {
                    keyToRemove = cacheKey;
                    break;
                }
            }
            if (keyToRemove != null) {
                cache.remove(keyToRemove);
            }
        }
        return cache.put(key, value);
    }

    @Override
    public Bitmap get(String key) {
        return cache.get(key);
    }

    @Override
    public Bitmap remove(String key) {
        return cache.remove(key);
    }

    @Override
    public void clear() {
        cache.clear();
    }

    @Override
    public Collection<String> keys() {
        return cache.keys();
    }
}

```

* FuzzyKeyMemoryCache也是继承自MemoryCache，同时也是MemoryCache的装饰类，装饰的新功能是：
* 当添加Bitmap进缓存时，如果添加进去的key跟之前添加过的key值相同的话，从内存移除之前key值对应的Bitmap，添加新传进来的Bitmap到内存。至于key相同的标准是什么,这个得根据自身的需求在重写Comparator对象并在初始化的时候传入。

* **UsingFreqLimitedMemoryCache**：

```java
public class UsingFreqLimitedMemoryCache extends LimitedMemoryCache {
    /**
     * Contains strong references to stored objects (keys) and last object usage date (in milliseconds). If hard cache
     * size will exceed limit then object with the least frequently usage is deleted (but it continue exist at
     * {@link #softMap} and can be collected by GC at any time)
     */
    private final Map<Bitmap, Integer> usingCounts = Collections.synchronizedMap(new HashMap<Bitmap, Integer>());

    public UsingFreqLimitedMemoryCache(int sizeLimit) {
        super(sizeLimit);
    }

    @Override
    public boolean put(String key, Bitmap value) {
        if (super.put(key, value)) {
            usingCounts.put(value, 0);
            return true;
        } else {
            return false;
        }
    }

    @Override
    public Bitmap get(String key) {
        Bitmap value = super.get(key);
        // Increment usage count for value if value is contained in hardCahe
        if (value != null) {
            Integer usageCount = usingCounts.get(value);
            if (usageCount != null) {
                usingCounts.put(value, usageCount + 1);
            }
        }
        return value;
    }

    @Override
    public Bitmap remove(String key) {
        Bitmap value = super.get(key);
        if (value != null) {
            usingCounts.remove(value);
        }
        return super.remove(key);
    }

    @Override
    public void clear() {
        usingCounts.clear();
        super.clear();
    }

    @Override
    protected int getSize(Bitmap value) {
        return value.getRowBytes() * value.getHeight();
    }

    @Override
    protected Bitmap removeNext() {
        Integer minUsageCount = null;
        Bitmap leastUsedValue = null;
        Set<Entry<Bitmap, Integer>> entries = usingCounts.entrySet();
        synchronized (usingCounts) {
            for (Entry<Bitmap, Integer> entry : entries) {
                if (leastUsedValue == null) {
                    leastUsedValue = entry.getKey();
                    minUsageCount = entry.getValue();
                } else {
                    Integer lastValueUsage = entry.getValue();
                    if (lastValueUsage < minUsageCount) {
                        minUsageCount = lastValueUsage;
                        leastUsedValue = entry.getKey();
                    }
                }
            }
        }
        usingCounts.remove(leastUsedValue);
        return leastUsedValue;
    }

    @Override
    protected Reference<Bitmap> createReference(Bitmap value) {
        return new WeakReference<Bitmap>(value);
    }
}

```

* UsingFreqLimitedMemoryCache继承自LimitedMemoryCache，所以他拥有双级缓存用能（强弱引用缓存）。
* UsingFreqLimitedMemoryCache的移除缓存的策略：每次使用get获得Bitmap时都会记录Bitmap使用的次数。要移除时，移除使用最少的次数的Bitmap，移除的只是强引用的缓存，弱引用中的缓存并不一定有被移除

* **LRULimit**

```java
public class LRULimitedMemoryCache extends LimitedMemoryCache {

    private static final int INITIAL_CAPACITY = 10;
    private static final float LOAD_FACTOR = 1.1f;

    /** Cache providing Least-Recently-Used logic */
    private final Map<String, Bitmap> lruCache = Collections.synchronizedMap(new LinkedHashMap<String, Bitmap>(INITIAL_CAPACITY, LOAD_FACTOR, true));

    /** @param maxSize Maximum sum of the sizes of the Bitmaps in this cache */
    public LRULimitedMemoryCache(int maxSize) {
        super(maxSize);
    }

    @Override
    public boolean put(String key, Bitmap value) {
        if (super.put(key, value)) {
            lruCache.put(key, value);
            return true;
        } else {
            return false;
        }
    }

    @Override
    public Bitmap get(String key) {
        lruCache.get(key); // call "get" for LRU logic
        return super.get(key);
    }

    @Override
    public Bitmap remove(String key) {
        lruCache.remove(key);
        return super.remove(key);
    }

    @Override
    public void clear() {
        lruCache.clear();
        super.clear();
    }

    @Override
    protected int getSize(Bitmap value) {
        return value.getRowBytes() * value.getHeight();
    }

    @Override
    protected Bitmap removeNext() {
        Bitmap mostLongUsedValue = null;
        synchronized (lruCache) {
            Iterator<Entry<String, Bitmap>> it = lruCache.entrySet().iterator();
            if (it.hasNext()) {
                Entry<String, Bitmap> entry = it.next();
                mostLongUsedValue = entry.getValue();
                it.remove();
            }
        }
        return mostLongUsedValue;
    }

    @Override
    protected Reference<Bitmap> createReference(Bitmap value) {
        return new WeakReference<Bitmap>(value);
    }
}
```
* LRULimitedMemoryCache继承自LimitedMemoryCache，同样也是具有两级缓存。而移除的策略是lru算法，这里不多分析。

* **LargestLimitedMemoryCache**

```java
public class LargestLimitedMemoryCache extends LimitedMemoryCache {
    /**
     * Contains strong references to stored objects (keys) and sizes of the objects. If hard cache
     * size will exceed limit then object with the largest size is deleted (but it continue exist at
     * {@link #softMap} and can be collected by GC at any time)
     */
    private final Map<Bitmap, Integer> valueSizes = Collections.synchronizedMap(new HashMap<Bitmap, Integer>());

    public LargestLimitedMemoryCache(int sizeLimit) {
        super(sizeLimit);
    }

    @Override
    public boolean put(String key, Bitmap value) {
        if (super.put(key, value)) {
            valueSizes.put(value, getSize(value));
            return true;
        } else {
            return false;
        }
    }

    @Override
    public Bitmap remove(String key) {
        Bitmap value = super.get(key);
        if (value != null) {
            valueSizes.remove(value);
        }
        return super.remove(key);
    }

    @Override
    public void clear() {
        valueSizes.clear();
        super.clear();
    }

    @Override
    protected int getSize(Bitmap value) {
        return value.getRowBytes() * value.getHeight();
    }

    @Override
    protected Bitmap removeNext() {
        Integer maxSize = null;
        Bitmap largestValue = null;
        Set<Entry<Bitmap, Integer>> entries = valueSizes.entrySet();
        synchronized (valueSizes) {
            for (Entry<Bitmap, Integer> entry : entries) {
                if (largestValue == null) {
                    largestValue = entry.getKey();
                    maxSize = entry.getValue();
                } else {
                    Integer size = entry.getValue();
                    if (size > maxSize) {
                        maxSize = size;
                        largestValue = entry.getKey();
                    }
                }
            }
        }
        valueSizes.remove(largestValue);
        return largestValue;
    }

    @Override
    protected Reference<Bitmap> createReference(Bitmap value) {
        return new WeakReference<Bitmap>(value);
    }
}

```
LargestLimitedMemoryCache继承自LimitedMemoryCache，拥有双级缓存。移除缓存的策略是移除占内存最大的Bitmap

* **FIFOLimitedMemoryCache**

```java
public class FIFOLimitedMemoryCache extends LimitedMemoryCache {

    private final List<Bitmap> queue = Collections.synchronizedList(new LinkedList<Bitmap>());

    public FIFOLimitedMemoryCache(int sizeLimit) {
        super(sizeLimit);
    }

    @Override
    public boolean put(String key, Bitmap value) {
        if (super.put(key, value)) {
            queue.add(value);
            return true;
        } else {
            return false;
        }
    }

    @Override
    public Bitmap remove(String key) {
        Bitmap value = super.get(key);
        if (value != null) {
            queue.remove(value);
        }
        return super.remove(key);
    }

    @Override
    public void clear() {
        queue.clear();
        super.clear();
    }

    @Override
    protected int getSize(Bitmap value) {
        return value.getRowBytes() * value.getHeight();
    }

    @Override
    protected Bitmap removeNext() {
        return queue.remove(0);
    }

    @Override
    protected Reference<Bitmap> createReference(Bitmap value) {
        return new WeakReference<Bitmap>(value);
    }
}
```

同理也是具有双级缓存功能，移除缓存的策略是移除头部的Bitmap

##### 内存缓存策略总结：
* 类库中提供的三个接口：
 * MemoryCache：实现该接口的类只有强引用的缓存功能
 * BaseMemoryCache：实现该接口的类只有弱引用的缓存功能
 * LimitedMemoryCache：实现该类的拥有两级缓存功能（强弱引用功能）

* 库中针对这三个接口，都给出了具体的实现类，我们可以根据自己的需求来设置缓存策略，默认的是LruMemoryCache
* 关于双级缓存：实现了LimitMemoryCache的类都具有双级缓存，但是为什么要使用双级缓存？
 * 理解：缓存内存的最大值是固定的，超过这个值后，我们就得从内存中移除Bitmap，但是我们有时不是想要移除，只是内存不足了，我们被迫移除了Bitmap，所以增加了一级强引用的缓存，当内存不足 时，首先移除强引用的Bitmap，弱引用并没有被移除。这样下次如果请求相同的Bitmap时，就可以从弱引用的缓存中取出。但是这样做也有缺点，牺牲了内存。
 * 总体说就是双级缓存就是通过牺牲内存来提高响应速度。
* 如果觉得库中提供给我们的策略不符合自己的业务需求的话，可以根据自己的业务需求来定制缓存策略。
