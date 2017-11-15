---
title: Volley--网络优化和缓存
data: 2016/10/30
tags:
  - Android#网络框架
categories:
  - Android
---
#### 概述：
volley的特点都大家很清楚，volley适合数据量小且通信频繁的请求，但是不适合数据量大的请求。volley有这样的特点是由其内部网络优化和缓存所决定的，这次分析其中的原理。
<!-- more -->
#### 网络优化

##### ByteArrayPool
* 首先Volley网络优化跟两个类息息相关，他们分别是ByteArrayPool和PoolingByteArrayOutputStream。接下来首先分析前者。
* 进入该类中，可以看到该类很长的文档注释。主要的意思是为了减少堆内存的压力而在将该类作为一个字节缓冲池。
* 首先看看该类中的成员变量：
```java
  /** The buffer pool, arranged both by last use and by buffer size */
    private List<byte[]> mBuffersByLastUse = new LinkedList<byte[]>();
    private List<byte[]> mBuffersBySize = new ArrayList<byte[]>(64);

    /** The total size of the buffers in the pool */
    private int mCurrentSize = 0;

    /**
     * The maximum aggregate size of the buffers in the pool. Old buffers are discarded to stay
     * under this limit.
     */
    private final int mSizeLimit;
```
* 两个List集合都是存放byte数组，也就是字节缓冲。mCurrentSize是记录当前缓冲池的大小。mSizeLimit是缓冲池的大小。
* 了解完成员变量的作用后，我们来看看其中的核心方法：
```java
/**
     * Returns a buffer from the pool if one is available in the requested size, or allocates a new
     * one if a pooled one is not available.
     *
     * @param len the minimum size, in bytes, of the requested buffer. The returned buffer may be
     *        larger.
     * @return a byte[] buffer is always returned.
     */
    public synchronized byte[] getBuf(int len) {
        for (int i = 0; i < mBuffersBySize.size(); i++) {
            byte[] buf = mBuffersBySize.get(i);
            if (buf.length >= len) {
                mCurrentSize -= buf.length;
                mBuffersBySize.remove(i);
                mBuffersByLastUse.remove(buf);
                return buf;
            }
        }
        return new byte[len];
    }
```
* 该方法的作用是向字节数组缓冲池中取出大于或者等于len长度的字节数组，取完后字节数组缓冲池的大小会减小。如果一直取的话，字节数组池肯定会被取完，所以也就有了向缓冲池中添加字节数组的方法：
```java
 /**
     * Returns a buffer to the pool, throwing away old buffers if the pool would exceed its allotted
     * size.
     *
     * @param buf the buffer to return to the pool.
     */
    public synchronized void returnBuf(byte[] buf) {
        if (buf == null || buf.length > mSizeLimit) {
            return;
        }
        mBuffersByLastUse.add(buf);
        int pos = Collections.binarySearch(mBuffersBySize, buf, BUF_COMPARATOR);
        if (pos < 0) {
            pos = -pos - 1;
        }
        mBuffersBySize.add(pos, buf);
        mCurrentSize += buf.length;
        trim();
    }

    /**
     * Removes buffers from the pool until it is under its size limit.
     */
    private synchronized void trim() {
        while (mCurrentSize > mSizeLimit) {
            byte[] buf = mBuffersByLastUse.remove(0);
            mBuffersBySize.remove(buf);
            mCurrentSize -= buf.length;
        }
    }
```
* 该方法的逻辑是向字节数组缓冲池中添加字节数组，并且当添加后，当前的缓冲池大小大于缓冲池的最大大小时，会控制其大小不超出规定的范围。
* 分析到里，为什么要用两个集合来存放Byte数组？
* 个人理解：
1.首先问题，我觉得在returnBuf中可用得到答案。因为在这个方法的最后，调用了trim（），该方法的作用是确保字节数组缓冲池的大小不超过最大限度。其中的移除思想是移除LastUseList的0号位的byte[]对象，然后也在BufSizeList中移除该Byte[]对象，直到字节数组缓冲池的大小不超过最大限度。
2.从移除LastUseList中的0号位，这使我想到的LinkList的特点，LinkList是用链表来实现的，链表的插入的话是直接插入到链表尾部的，换句话说，越晚插入的在List的越尾部，也就是越新的缓存，移除0号位的缓存是移除比较旧的缓存
3.因此，我觉得LastUseList的作用是来配合BufSiseList移除比较旧的缓存区
--------------
##### PoolingByteArrayOutputStream
接下来，我们来看看和ByteArrayPool配合使PoolingByteArrayOutputStream。
* 该类继承了ByteArrayOutputStream，ByteArrayOutputStream的作用是在将数据源输入到其中的Byte数组中，也就是内存缓冲区。
* 该类的就是通过向字节数组缓冲池拿字节数组和添加字节数组。

##### BasicNetwork
* 在BasicNetwork的performRequest方法中进行网络时，得到httpResponse后，再调用了：
    responseContents = entityToBytes(httpResponse.getEntity());
* 我们来看看entityToBytes方法：
```java
/** Reads the contents of HttpEntity into a byte[]. */
    private byte[] entityToBytes(HttpEntity entity) throws IOException, ServerError {
        PoolingByteArrayOutputStream bytes =
                new PoolingByteArrayOutputStream(mPool, (int) entity.getContentLength());
        byte[] buffer = null;
        try {
            InputStream in = entity.getContent();
            if (in == null) {
                throw new ServerError();
            }
            buffer = mPool.getBuf(1024);
            int count;
            while ((count = in.read(buffer)) != -1) {
                bytes.write(buffer, 0, count);
            }
            return bytes.toByteArray();
        } finally {
            try {
                // Close the InputStream and release the resources by "consuming the content".
                entity.consumeContent();
            } catch (IOException e) {
                // This can happen if there was an exception above that left the entity in
                // an invalid state.
                VolleyLog.v("Error occured when calling consumingContent");
            }
            mPool.returnBuf(buffer);
            bytes.close();
        }
    }
```
* 该类主要就是进行输入流的读取，但是它的读取不同于一般的读取，它内部的通过实例化PoolingByteArrayOutputStream，将读取的流都读到内部的字节数组，而字节数组的来源是字节数组缓冲池。这样做的好处是：由于不用再频繁给byte[]分配内部，而是通过从回收和利用字节数组，减少了堆的开销，将读取的数据全都读到缓冲中后，再一次性送出去。这样就大大提高了效率。
---------
#### 缓存
上面分析了ByteArrayPool和PoolByteArrayOutputStream的配合使用，
接着来分析缓存。缓存的操作主要是在DiskBaseCache中。
* 现在我需要先来分析该类中的内部类CacheHeaher和CountingInputStream
* 在CacheHeader主要是对响应头进行缓存，其中readHeader和WriteHeader是将响应头读取和写进文件中
* 而CountingInputStream继承于FileInputStream中，主要扩展了记录读取的数据的长度
* 介绍完这两个内部类，我们来切入正题：
* 先看看类中初始化的方法
```java
/**
     * Initializes the DiskBasedCache by scanning for all files currently in the
     * specified root directory. Creates the root directory if necessary.
     */
    @Override
    public synchronized void initialize() {
        if (!mRootDirectory.exists()) {
            if (!mRootDirectory.mkdirs()) {
                VolleyLog.e("Unable to create cache dir %s", mRootDirectory.getAbsolutePath());
            }
            return;
        }

        File[] files = mRootDirectory.listFiles();
        if (files == null) {
            return;
        }
        for (File file : files) {
            FileInputStream fis = null;
            try {
                fis = new FileInputStream(file);
                CacheHeader entry = CacheHeader.readHeader(fis);
                entry.size = file.length();
                putEntry(entry.key, entry);
            } catch (IOException e) {
                if (file != null) {
                   file.delete();
                }
            } finally {
                try {
                    if (fis != null) {
                        fis.close();
                    }
                } catch (IOException ignored) { }
            }
        }
    }
```
* 该方法的逻辑主要是如果读取文件中内容到内存中保存缓存头的map，该方法会在缓存线程总的run方法中的调用
* 接着看将数据写进文件的方法：
```java
/**
     * Puts the entry with the specified key into the cache.
     */
    @Override
    public synchronized void put(String key, Entry entry) {
        pruneIfNeeded(entry.data.length);
        File file = getFileForKey(key);
        try {
            FileOutputStream fos = new FileOutputStream(file);
            CacheHeader e = new CacheHeader(key, entry);
            e.writeHeader(fos);
            fos.write(entry.data);
            fos.close();
            putEntry(key, e);
            return;
        } catch (IOException e) {
        }
        boolean deleted = file.delete();
        if (!deleted) {
            VolleyLog.d("Could not clean up file %s", file.getAbsolutePath());
        }
    }
```
* pruneIfNeeded（int）该方法会判断内存缓存的大小是不是够用，不够的话，就通过遍历删除，直到增加后的内存小于内存缓存最大的90%。
* 接着就是实例化CacheHeader对象，再将缓存头和数据一起写进文件，最后将缓存头添加进map中
* 再来看看从文件中读取的方法
```java
@Override
    public synchronized Entry get(String key) {
        CacheHeader entry = mEntries.get(key);
        // if the entry does not exist, return.
        if (entry == null) {
            return null;
        }

        File file = getFileForKey(key);
        CountingInputStream cis = null;
        try {
            cis = new CountingInputStream(new FileInputStream(file));
            CacheHeader.readHeader(cis); // eat header
            byte[] data = streamToBytes(cis, (int) (file.length() - cis.bytesRead));
            return entry.toCacheEntry(data);
        } catch (IOException e) {
            VolleyLog.d("%s: %s", file.getAbsolutePath(), e.toString());
            remove(key);
            return null;
        } finally {
            if (cis != null) {
                try {
                    cis.close();
                } catch (IOException ioe) {
                    return null;
                }
            }
        }
    }
```
* 先读取数据再和缓存头包装成一个Entry对象返回

------------

* 通过上面的分析，对Volley的高效等特点总算有了一个全新的认识。Volley内部通过使用字节数组缓存池的利用与回收来减少堆内存的频繁分配，以及通过缓存响应头来控制是否从缓存中读取数据，这样适合一些小数据但通信频繁的操作，而至于为什么会不适合用于数据量大的操作主要是由于请求读取的结果都是一次性存在内存中，当数据量大时可能导致内存爆炸。
* Volley总体来说是面向接口编程，再将这些接口功能组合起来，具体的封装交给具体类，这样扩展性会突出
