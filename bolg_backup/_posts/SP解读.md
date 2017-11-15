---
layout: post
title: SP解析
data: 2017/5/10
subtitle: ""
description: ""
tags:
  - Android#数据持久化#SP
categories:
  - Android
---

![](http://ww1.sinaimg.cn/large/006VdOYcgy1flhnrmq40dj30m80b4q9b.jpg)

> ## 概述

``SharedPreferences``是``Android``中的数据持久化技术中的一种. 它将``Key-Value``键值对储存在一个``XMl``文件中. 它比较适用储存小量的数据. 比如一些应用的配置信息.

> ## 使用

``SharedPreferences``的被设计成``读写分离``的模式,  ``SharedPreferences``用来读取数据, ``SharedPreferences.Editor``则是用来写数据.

> ### 获取SharedPreferences

关于如何获取``SharedPreferences``, 可以通过下面这三种方法来获取:

```java
SharedPreferences sharedPreferences = Activity.getPreferences(MODE_PRIVATE); //在Activity中调用
SharedPreferences sharedPreferences = PreferenceManager.getDefaultSharedPreferences(context);
SharedPreferences sharedPreferences = context.getSharedPreferences("cr7", MODE_APPEND);
```

前面两种都是第三种的封装. 第三种的函数原型为: ``Context.getSharedPreferences(String name, int mode)``. 其中``name``为文件的名字, ``mode``为操作模式. 操作模式可选有:

```java
public static final int MODE_PRIVATE = 0x0000; //默认的模式, 每次写都会覆盖之前的数据. 并且只允许本应用内或者拥有用一个userId的应用进行读写文件
public static final int MODE_WORLD_READABLE = 0x0001; //允许多个应用进行读, 由于安全性问题, 已被弃用.
public static final int MODE_WORLD_WRITEABLE = 0x0002; //允许多个应用进行写, 由于安全性问题, 已被弃用.
public static final int MODE_APPEND = 0x8000; //如果文件已经存在的话, 写不会覆盖之前的数据, 会直接写到文件的末尾.
public static final int MODE_MULTI_PROCESS = 0x0004; //多进程模式, 该模式不安全, 不建议使用.
```

上面有些是由于安全性的问题被弃用的. 而``MODE_MULTI_PROCESS``用于多进程读写的模式, 但是不能保证可靠性, 有可能会出现脏读现象, 已被官方弃用.

``SharedPreferences``的文件储存在``/data/data/shared_prefs/``. ``Activity.getPreferences(MODE_PRIVATE)``默认的文件名: 调用的``Activity``的名字, 比如在``SpActivity``中调用, 得到的文件名字为: ``SpActivity.xml``. 至于调用``PreferenceManager.getDefaultSharedPreferences(context)``得到的文件名字为: ``包名+ _preferences.xml``.

获取完``SharedPreferences``后, 我们就可以调用其中的``getXX``来获取数据.

```java
sharedPreferences.getString("cr7", ""); //如果不存在cr7这个key的话, 返回默认值""
```

> ### 获取SharedPreferences.Editor

获得``SharedPreferences``实例后, 我们可以调用它的``editor()``方法拿到``Editor``对象, 注意, 每次调用``editor()``方法都是新建一个``Editor``对象. 拿到对象后, 就可以进行写数据了.

```java
editor.putString("cr7", "good and best");
editor.apply();
```

要将数据写入文件中的话, 最后还得调用``apply()``或者``commit()``方法.


> ### 监听Key的对应的Value值的变化

```java
sharedPreferences.registerOnSharedPreferenceChangeListener(new SharedPreferences.OnSharedPreferenceChangeListener() {
    @Override
    public void onSharedPreferenceChanged(SharedPreferences sharedPreferences, String key) {
        Log.d(TAG, "onSharedPreferenceChanged: " + key);
    }
});

sharedPreferences.unregisterOnSharedPreferenceChangeListener(this);
```

如果我们需要监听``Key``对应的``Value``的变化的话, 可以调用``sharedPreferences``中的注册方法. 每当``Key``值对应的``Value``发生变化时, 都会回调这个接口. 但是不用的时候, 记得调用注销接口.

> ## 原理解析

![](http://ww1.sinaimg.cn/large/006VdOYcgy1flhc48le84j312h0hodhv.jpg)

上图为``sharedPreferences``的UML设计图, ``sharedPreferences``和``sharedPreferences.Editor``只是一个接口, 对应的实现由``SharedPreferencesImpl``和``SharedPreferencesImpl.EditorImpl``来负责.

``SharedPreferencesImpl``被设计成单例的形式, 是由``ContextImpl``负责创建和维护. 在``ContextImpl``中有字段:

```java
private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;
```

这个字段是负责维护``SharedPreferences``的. 并且为静态字段,证明同一进程中只有一个实例.

> ### 创建SharedPreferences

``SharedPreferences``的创建是由``ContextImpl``负责的, 对应实现方法为:

```java
public SharedPreferences getSharedPreferences(String name, int mode) {
    // At least one application in the world actually passes in a null
    // name.  This happened to work because when we generated the file name
    // we would stringify it to "null.xml".  Nice.
    if (mPackageInfo.getApplicationInfo().targetSdkVersion <
            Build.VERSION_CODES.KITKAT) {
        if (name == null) {
            name = "null";
        }
    }

    File file;
    synchronized (ContextImpl.class) {
        if (mSharedPrefsPaths == null) {
            mSharedPrefsPaths = new ArrayMap<>();
        }
        file = mSharedPrefsPaths.get(name); //根据路径得到文件
        if (file == null) { //如果没有缓存的话, 创建对应的文件
            file = getSharedPreferencesPath(name); //创建文件
            mSharedPrefsPaths.put(name, file); //添加入缓存
        }
    }
    return getSharedPreferences(file, mode);
}
```

结合注释: 首先根据传入的文件名, 先从缓存中拿, 如果不存在的话, 说明该文件还没有被创建, 因此会创建文件后添加进缓存中. 接着调用``getSharedPreferences(file, mode)``获取``SharedPreferences``实例.

```java
public SharedPreferences getSharedPreferences(File file, int mode) {
    checkMode(mode); //检查模式
    if (getApplicationInfo().targetSdkVersion >= android.os.Build.VERSION_CODES.O) {
        if (isCredentialProtectedStorage()
                && !getSystemService(StorageManager.class).isUserKeyUnlocked(
                        UserHandle.myUserId())
                && !isBuggy()) {
            throw new IllegalStateException("SharedPreferences in credential encrypted "
                    + "storage are not available until after user is unlocked");
        }
    }
    SharedPreferencesImpl sp;
    synchronized (ContextImpl.class) {
        final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
        sp = cache.get(file); //从缓存中获取SP
        if (sp == null) { //不存在的话, 创建实例并添加进缓存.
            sp = new SharedPreferencesImpl(file, mode);
            cache.put(file, sp);
            return sp;
        }
    }
    //设置了MODE_MULTI_PROCESS模式后, 会重新加载文件. 但是这不能解决多进程的脏读现象
    if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
        getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
        // If somebody else (some other process) changed the prefs
        // file behind our back, we reload it.  This has been the
        // historical (if undocumented) behavior.
        sp.startReloadIfChangedUnexpectedly();
    }
    return sp;
}
```

在``getSharedPreferences(File file, int mode)``中的逻辑, 会先调用``checkMode(int)``来检查模式.

```java
private void checkMode(int mode) {
    if (getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.N) {
        if ((mode & MODE_WORLD_READABLE) != 0) {
            throw new SecurityException("MODE_WORLD_READABLE no longer supported");
        }
        if ((mode & MODE_WORLD_WRITEABLE) != 0) {
            throw new SecurityException("MODE_WORLD_WRITEABLE no longer supported");
        }
    }
}
```

在``Android N``后, 出于安全性的考虑, 开始不支持``MODE_WORLD_READABLE``和``MODE_WORLD_WRITEABLE``.

检查完模式后, 接着从缓存中获取该文件对应的``SharedPreferences``, 如果存在的话, 返回. 不存在的话, 创建实例并添加进缓存.

经过上面的步骤就创建并缓存了``SharedPreferences``. 那么我们下面来看看``SharedPreferences``创建的时候内部都做了什么.

> ### SharedPreferencesImpl

```java
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    mBackupFile = makeBackupFile(file); //备份文件, 用于文件读写失败时恢复
    mMode = mode;
    mLoaded = false;
    mMap = null;
    startLoadFromDisk(); //开始从硬盘中加载文件
}
```

在``SharedPreferencesImpl``的构造方法中, 首先备份文件, 用于文件读写失败时的恢复. 最后调用``startLoadFromDisk``从硬盘中加载文件.

```java
private void startLoadFromDisk() {
    synchronized (mLock) {
        mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}
```

先将加载标记``mLoad``设置为``false``表示文件还没有加载完. 然后开一条线程去加载文件.

```java
private void loadFromDisk() {
    synchronized (mLock) {
        if (mLoaded) {
            return;
        }
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }

    // Debugging
    if (mFile.exists() && !mFile.canRead()) {
        Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
    }

    Map map = null;
    StructStat stat = null;
    try {
        stat = Os.stat(mFile.getPath());
        if (mFile.canRead()) {
            BufferedInputStream str = null;
            try {
                str = new BufferedInputStream(
                        new FileInputStream(mFile), 16*1024);
                map = XmlUtils.readMapXml(str); //将文件内容读进Map中
            } catch (Exception e) {
                Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
            } finally {
                IoUtils.closeQuietly(str);
            }
        }
    } catch (ErrnoException e) {
        /* ignore */
    }

    synchronized (mLock) {
        mLoaded = true;
        if (map != null) {
            mMap = map;
            mStatTimestamp = stat.st_mtime;
            mStatSize = stat.st_size;
        } else {
            mMap = new HashMap<>();
        }
        mLock.notifyAll();
    }
}
```

上面的方法主要做了两件事, 1: 从``XML``文件中读出数据, 并存放在``Map``中. 2: 将加载标记设置为``true``表示已经加载完成. 然后调用``notifyAll()``通知正在等待读写的线程.

> ### 利用SharedPreferences读数据

创建完``SharedPreferences``后, 我们就可以拿到它的实例来进行读数据, 下面通过其中的``getString(String key, String defValue)``来分析.

```java
public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}

private void awaitLoadedLocked() {
    if (!mLoaded) {
        // Raise an explicit StrictMode onReadFromDisk for this
        // thread, since the real read will be in a different
        // thread and otherwise ignored by StrictMode.
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    while (!mLoaded) {
        try {
            mLock.wait();
        } catch (InterruptedException unused) {
        }
    }
}
```

每次读数据时, 都会判断文件是否加载完, 没有的话, 会调用``wait()``挂起线程, 直到文件加载完才被唤醒.

前面说过文件的内容都是加载进了``mMap``中, 所以``getXXX``方法获取的数据都是从``mMap``中获取.

> ### Editor写数据

前面分析过, ``SharedPreferences``的读写是分离的. 要进行写数据的话, 我们需要拿到``Editor``对象, 这个对象可以通过``SharedPreferences``中的``editor()``方法拿到:

```java
public Editor edit() {
    // TODO: remove the need to call awaitLoadedLocked() when
    // requesting an editor.  will require some work on the
    // Editor, but then we should be able to do:
    //
    //      context.getSharedPreferences(..).edit().putString(..).apply()
    //
    // ... all without blocking.
    synchronized (mLock) {
        awaitLoadedLocked();
    }

    return new EditorImpl();
}
```

注意: 每次调用这个方法的方法都是新建一个实例, 所以在使用的时候,最好缓存起来, 避免多次调用生成多个对象.

下面也是通过``Editor``中的``putString(String key, String value)``为例子来分析:

```java
public Editor putString(String key, @Nullable String value) {
    synchronized (mLock) {
        mModified.put(key, value);
        return this;
    }
}
```

写数据时, 并没有直接操作``SharedPreferences``中的``mMap``, 而是自己新建一个``mModified``的``Map``对象来记录修改的数据.

从上面可以看出, 调用``putXXX``方法, 数据只是存在内存中, 此时还没有写进磁盘. 需要调用``commit()``或者``apply()``方法.

> ### 提交数据commit()

```java
public boolean commit() {
    long startTime = 0;

    if (DEBUG) {
        startTime = System.currentTimeMillis();
    }

    MemoryCommitResult mcr = commitToMemory(); //将修改的数据写进mMap中

    SharedPreferencesImpl.this.enqueueDiskWrite(
        mcr, null /* sync write on this thread okay */); //写进磁盘
    try {
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    } finally {
        if (DEBUG) {
            Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                    + " committed after " + (System.currentTimeMillis() - startTime)
                    + " ms");
        }
    }
    notifyListeners(mcr);
    return mcr.writeToDiskResult;
}
```

``commit()``方法总体逻辑: 1: 将写的数据同步到``SharedPreferences``中的``mMap``中; 2: 将``mMap``写进磁盘. 3: 回到``Key``监听器.

> #### commitToMemory

```java
private MemoryCommitResult commitToMemory() {
    long memoryStateGeneration;
    List<String> keysModified = null;
    Set<OnSharedPreferenceChangeListener> listeners = null;
    Map<String, Object> mapToWriteToDisk;

    synchronized (SharedPreferencesImpl.this.mLock) {
        // We optimistically don't make a deep copy until
        // a memory commit comes in when we're already
        // writing to disk.
        if (mDiskWritesInFlight > 0) {
            // We can't modify our mMap as a currently
            // in-flight write owns it.  Clone it before
            // modifying it.
            // noinspection unchecked
            mMap = new HashMap<String, Object>(mMap);
        }
        mapToWriteToDisk = mMap;
        mDiskWritesInFlight++;

        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            keysModified = new ArrayList<String>();
            listeners = new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
        }

        synchronized (mLock) {
            boolean changesMade = false;

            if (mClear) { //如果在调用了Editor的clear()方法的话, 会将mMap中的数据先清除
                if (!mMap.isEmpty()) {
                    changesMade = true;
                    mMap.clear();
                }
                mClear = false;
            }

            for (Map.Entry<String, Object> e : mModified.entrySet()) { //将mModified的数据添加到mMap中
                String k = e.getKey();
                Object v = e.getValue();
                // "this" is the magic value for a removal mutation. In addition,
                // setting a value to "null" for a given key is specified to be
                // equivalent to calling remove on that key.
                if (v == this || v == null) {
                    if (!mMap.containsKey(k)) {
                        continue;
                    }
                    mMap.remove(k);
                } else {
                    if (mMap.containsKey(k)) {
                        Object existingValue = mMap.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    mMap.put(k, v);
                }

                changesMade = true;
                if (hasListeners) {
                    keysModified.add(k);
                }
            }

            mModified.clear();

            if (changesMade) {
                mCurrentMemoryStateGeneration++;
            }

            memoryStateGeneration = mCurrentMemoryStateGeneration;
        }
    }
    return new MemoryCommitResult(memoryStateGeneration, keysModified, listeners,
            mapToWriteToDisk);
}
```

首先判断``mClear``字段是否被设置, 是的话, 清除``mMap``中的数据. 接着将``mModified``的数据添加到``mMap``中.

> #### enqueueDiskWrite

```java
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    final boolean isFromSyncCommit = (postWriteRunnable == null); //由于传入的postWriteRunnable为null, 所以该字段为true

    final Runnable writeToDiskRunnable = new Runnable() {
            public void run() {
                synchronized (mWritingToDiskLock) {
                    writeToFile(mcr, isFromSyncCommit); //写入文件
                }
                synchronized (mLock) {
                    mDiskWritesInFlight--;
                }
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };

    // Typical #commit() path with fewer allocations, doing a write on
    // the current thread.
    if (isFromSyncCommit) {
        boolean wasEmpty = false;
        synchronized (mLock) {
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) {
            writeToDiskRunnable.run();
            return;
        }
    }

    QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}
```

由于传入的``postWriteRunnable``为``null``, ``isFromSyncCommit``为``true``. 然后调用``writeToFile(mcr, isFromSyncCommit)``将数据写进磁盘.

```java
private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
    long startTime = 0;
    long existsTime = 0;
    long backupExistsTime = 0;
    long outputStreamCreateTime = 0;
    long writeTime = 0;
    long fsyncTime = 0;
    long setPermTime = 0;
    long fstatTime = 0;
    long deleteTime = 0;

    if (DEBUG) {
        startTime = System.currentTimeMillis();
    }

    boolean fileExists = mFile.exists();

    if (DEBUG) {
        existsTime = System.currentTimeMillis();

        // Might not be set, hence init them to a default value
        backupExistsTime = existsTime;
    }

    // Rename the current file so it may be used as a backup during the next read
    if (fileExists) {
        boolean needsWrite = false;

        // Only need to write if the disk state is older than this commit
        if (mDiskStateGeneration < mcr.memoryStateGeneration) {
            if (isFromSyncCommit) {
                needsWrite = true;
            } else {
                synchronized (mLock) {
                    // No need to persist intermediate states. Just wait for the latest state to
                    // be persisted.
                    if (mCurrentMemoryStateGeneration == mcr.memoryStateGeneration) {
                        needsWrite = true;
                    }
                }
            }
        }

        if (!needsWrite) {
            mcr.setDiskWriteResult(false, true);
            return;
        }

        boolean backupFileExists = mBackupFile.exists();

        if (DEBUG) {
            backupExistsTime = System.currentTimeMillis();
        }

        if (!backupFileExists) {
            if (!mFile.renameTo(mBackupFile)) {
                Log.e(TAG, "Couldn't rename file " + mFile
                      + " to backup file " + mBackupFile);
                mcr.setDiskWriteResult(false, false);
                return;
            }
        } else {
            mFile.delete();
        }
    }

    // Attempt to write the file, delete the backup and return true as atomically as
    // possible.  If any exception occurs, delete the new file; next time we will restore
    // from the backup.
    try {
        FileOutputStream str = createFileOutputStream(mFile);

        if (DEBUG) {
            outputStreamCreateTime = System.currentTimeMillis();
        }

        if (str == null) {
            mcr.setDiskWriteResult(false, false);
            return;
        }
        XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str); //将数据写入文件

        writeTime = System.currentTimeMillis();

        FileUtils.sync(str);

        fsyncTime = System.currentTimeMillis();

        str.close();
        ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);

        if (DEBUG) {
            setPermTime = System.currentTimeMillis();
        }

        try {
            final StructStat stat = Os.stat(mFile.getPath());
            synchronized (mLock) {
                mStatTimestamp = stat.st_mtime;
                mStatSize = stat.st_size;
            }
        } catch (ErrnoException e) {
            // Do nothing
        }

        if (DEBUG) {
            fstatTime = System.currentTimeMillis();
        }

        // Writing was successful, delete the backup file if there is one.
        mBackupFile.delete(); //删除备份文件

        if (DEBUG) {
            deleteTime = System.currentTimeMillis();
        }

        mDiskStateGeneration = mcr.memoryStateGeneration; //更新磁盘状态代数

        mcr.setDiskWriteResult(true, true); //设置结果

        if (DEBUG) {
            Log.d(TAG, "write: " + (existsTime - startTime) + "/"
                    + (backupExistsTime - startTime) + "/"
                    + (outputStreamCreateTime - startTime) + "/"
                    + (writeTime - startTime) + "/"
                    + (fsyncTime - startTime) + "/"
                    + (setPermTime - startTime) + "/"
                    + (fstatTime - startTime) + "/"
                    + (deleteTime - startTime));
        }

        long fsyncDuration = fsyncTime - writeTime;
        mSyncTimes.add(Long.valueOf(fsyncDuration).intValue());
        mNumSync++;

        if (DEBUG || mNumSync % 1024 == 0 || fsyncDuration > MAX_FSYNC_DURATION_MILLIS) {
            mSyncTimes.log(TAG, "Time required to fsync " + mFile + ": ");
        }

        return;
    } catch (XmlPullParserException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    } catch (IOException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    }

    // Clean up an unsuccessfully written file
    if (mFile.exists()) {
        if (!mFile.delete()) {
            Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
        }
    }
    mcr.setDiskWriteResult(false, false);
}
```

这个``writeToFile``方法是``commit()``和``apply()``公用的. 接下来说说``commit()``会走的流程: 首先将当前文件备份, 接着写入磁盘, 最后删除备份文件.

经过前面的分析, 我们可以看出, ``commit()``方法是以同步的方式将数据写入磁盘.

> ### 提交数据apply

```java
public void apply() {
    final long startTime = System.currentTimeMillis();

    final MemoryCommitResult mcr = commitToMemory(); //将数据提交到内存
    final Runnable awaitCommit = new Runnable() {
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }

                if (DEBUG && mcr.wasWritten) {
                    Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                            + " applied after " + (System.currentTimeMillis() - startTime)
                            + " ms");
                }
            }
        };

    QueuedWork.addFinisher(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
            public void run() {
                awaitCommit.run();
                QueuedWork.removeFinisher(awaitCommit);
            }
        };

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable); //写入磁盘

    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    notifyListeners(mcr);
}
```

``apply()``的逻辑跟``commit()``一样, 只不过, 写入磁盘的方式不一样, 下面我们只分析写入磁盘的逻辑:

```java
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    final boolean isFromSyncCommit = (postWriteRunnable == null);

    final Runnable writeToDiskRunnable = new Runnable() {
            public void run() {
                synchronized (mWritingToDiskLock) {
                    writeToFile(mcr, isFromSyncCommit);
                }
                synchronized (mLock) {
                    mDiskWritesInFlight--;
                }
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };

    // Typical #commit() path with fewer allocations, doing a write on
    // the current thread.
    if (isFromSyncCommit) {
        boolean wasEmpty = false;
        synchronized (mLock) {
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) {
            writeToDiskRunnable.run();
            return;
        }
    }

    QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}
```

由于传入的``Runnable``不为``null``, 所以``isFromSyncCommit``为``false``. 最后会调用``QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit)``.

```java
public static void queue(Runnable work, boolean shouldDelay) {
    Handler handler = getHandler();

    synchronized (sLock) {
        sWork.add(work);

        if (shouldDelay && sCanDelay) {
            handler.sendEmptyMessageDelayed(QueuedWorkHandler.MSG_RUN, DELAY);
        } else {
            handler.sendEmptyMessage(QueuedWorkHandler.MSG_RUN);
        }
    }
}
```

``QueueWork``封装了``HandlerThread``和``Handler``. 用来处理``apply()``的异步写入的请求.

在``queue``方法中, 发送了一条延迟消息. 延迟时间为100毫秒.

```java
private static class QueuedWorkHandler extends Handler {
    static final int MSG_RUN = 1;

    QueuedWorkHandler(Looper looper) {
        super(looper);
    }

    public void handleMessage(Message msg) {
        if (msg.what == MSG_RUN) {
            processPendingWork();
        }
    }
}
```

``QueuedWorkHandler``接受到消息时, 调用``processPendingWork()``处理异步写入请求

```java
private static void processPendingWork() {
    long startTime = 0;

    if (DEBUG) {
        startTime = System.currentTimeMillis();
    }

    synchronized (sProcessingWork) {
        LinkedList<Runnable> work;

        synchronized (sLock) {
            work = (LinkedList<Runnable>) sWork.clone();
            sWork.clear();

            // Remove all msg-s as all work will be processed now
            getHandler().removeMessages(QueuedWorkHandler.MSG_RUN);
        }

        if (work.size() > 0) {
            for (Runnable w : work) {
                w.run(); //调用Runnable的run方法
            }

            if (DEBUG) {
                Log.d(LOG_TAG, "processing " + work.size() + " items took " +
                        +(System.currentTimeMillis() - startTime) + " ms");
            }
        }
    }
}
```

在``processPendingWork()``中会循环调用``Runnable``中的``run``方法. 现在我们回到之前的``Runnable``

```java
final Runnable writeToDiskRunnable = new Runnable() {
        public void run() {
            synchronized (mWritingToDiskLock) {
                writeToFile(mcr, isFromSyncCommit); //写入文件
            }
            synchronized (mLock) {
                mDiskWritesInFlight--;
            }
            if (postWriteRunnable != null) {
                postWriteRunnable.run();
            }
        }
    };
```

```java
private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
     long startTime = 0;
     long existsTime = 0;
     long backupExistsTime = 0;
     long outputStreamCreateTime = 0;
     long writeTime = 0;
     long fsyncTime = 0;
     long setPermTime = 0;
     long fstatTime = 0;
     long deleteTime = 0;

     if (DEBUG) {
         startTime = System.currentTimeMillis();
     }

     boolean fileExists = mFile.exists();

     if (DEBUG) {
         existsTime = System.currentTimeMillis();

         // Might not be set, hence init them to a default value
         backupExistsTime = existsTime;
     }

     // Rename the current file so it may be used as a backup during the next read
     if (fileExists) {
         boolean needsWrite = false;

         // Only need to write if the disk state is older than this commit
         if (mDiskStateGeneration < mcr.memoryStateGeneration) {
             if (isFromSyncCommit) {
                 needsWrite = true;
             } else {
                 synchronized (mLock) {
                     // No need to persist intermediate states. Just wait for the latest state to
                     // be persisted.
                     if (mCurrentMemoryStateGeneration == mcr.memoryStateGeneration) { //如果有多次提交的话, 只处理最后一次提交的数据
                         needsWrite = true;
                     }
                 }
             }
         }

         if (!needsWrite) {
             mcr.setDiskWriteResult(false, true);
             return;
         }

         boolean backupFileExists = mBackupFile.exists();

         if (DEBUG) {
             backupExistsTime = System.currentTimeMillis();
         }

         if (!backupFileExists) {
             if (!mFile.renameTo(mBackupFile)) {
                 Log.e(TAG, "Couldn't rename file " + mFile
                       + " to backup file " + mBackupFile);
                 mcr.setDiskWriteResult(false, false);
                 return;
             }
         } else {
             mFile.delete();
         }
     }

     // Attempt to write the file, delete the backup and return true as atomically as
     // possible.  If any exception occurs, delete the new file; next time we will restore
     // from the backup.
     try {
         FileOutputStream str = createFileOutputStream(mFile);

         if (DEBUG) {
             outputStreamCreateTime = System.currentTimeMillis();
         }

         if (str == null) {
             mcr.setDiskWriteResult(false, false);
             return;
         }
         XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);

         writeTime = System.currentTimeMillis();

         FileUtils.sync(str);

         fsyncTime = System.currentTimeMillis();

         str.close();
         ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);

         if (DEBUG) {
             setPermTime = System.currentTimeMillis();
         }

         try {
             final StructStat stat = Os.stat(mFile.getPath());
             synchronized (mLock) {
                 mStatTimestamp = stat.st_mtime;
                 mStatSize = stat.st_size;
             }
         } catch (ErrnoException e) {
             // Do nothing
         }

         if (DEBUG) {
             fstatTime = System.currentTimeMillis();
         }

         // Writing was successful, delete the backup file if there is one.
         mBackupFile.delete();

         if (DEBUG) {
             deleteTime = System.currentTimeMillis();
         }

         mDiskStateGeneration = mcr.memoryStateGeneration;

         mcr.setDiskWriteResult(true, true);

         if (DEBUG) {
             Log.d(TAG, "write: " + (existsTime - startTime) + "/"
                     + (backupExistsTime - startTime) + "/"
                     + (outputStreamCreateTime - startTime) + "/"
                     + (writeTime - startTime) + "/"
                     + (fsyncTime - startTime) + "/"
                     + (setPermTime - startTime) + "/"
                     + (fstatTime - startTime) + "/"
                     + (deleteTime - startTime));
         }

         long fsyncDuration = fsyncTime - writeTime;
         mSyncTimes.add(Long.valueOf(fsyncDuration).intValue());
         mNumSync++;

         if (DEBUG || mNumSync % 1024 == 0 || fsyncDuration > MAX_FSYNC_DURATION_MILLIS) {
             mSyncTimes.log(TAG, "Time required to fsync " + mFile + ": ");
         }

         return;
     } catch (XmlPullParserException e) {
         Log.w(TAG, "writeToFile: Got exception:", e);
     } catch (IOException e) {
         Log.w(TAG, "writeToFile: Got exception:", e);
     }

     // Clean up an unsuccessfully written file
     if (mFile.exists()) {
         if (!mFile.delete()) {
             Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
         }
     }
     mcr.setDiskWriteResult(false, false);
 }
```

``apply()``中将数据写入磁盘跟``commit()``是有区别的. 前者的写入方式是异步写入, 也就是在``QueueWork``所在的线程. 所以当进行多次提交时, 只会将最后一次提交的数据写入磁盘.

> ## 二级缓存

![](http://ww1.sinaimg.cn/large/006VdOYcgy1flhmk9nwkfj30dc09nt95.jpg)

``SharedPreferences``内部对数据的储存是经过内存-硬盘这样的两级缓存. 每次``commit``或者``apply``的时候, 都会先写入内存, 接着再将内存写入硬盘.

``commit()``方法是以同步的方式提交到内存. 具体如下图:

![](http://ww1.sinaimg.cn/large/006VdOYcgy1flhmznz2a7j30el0ahwf0.jpg)

两次``commit``的提交是串行化的, ``commit#2``必须要等待``commit#1``写入磁盘后才能写.

与``commit()``方法不同. ``apply()``是以异步的形式将数据写入磁盘, 也就是说: 写进内存是同步的, 但是内存写进磁盘是异步的, 当进行多次提交时, 后提交的会覆盖前面提交到内存的数据, 最后只有最后一次的提交才会被写进磁盘. 具体如下图:

![](http://ww1.sinaimg.cn/large/006VdOYcgy1flhnaf4orxj30ga0anaak.jpg)

> ## 总结

* ``SharedPreferences``适合储存一些小数据的情景. 如果储存的数据太大的话, 第一次加载时, 有可能会阻塞到主线程.

* ``editor()``每次调用都会生成一个``Editor``实例, 所以在封装的时候要注意, 最好缓存起来, 不然频繁调用的话, 有可能会出现内存抖动.

* ``commit``是在当前线程以同步的方法写入磁盘, 如果在主线程连续多次提交的话, 有可能阻塞主线程.

* ``apply``是先将数据同步写到内存, 然后将内存的数据异步写入磁盘, 并且只会写入最后一次提交的数据, 这样磁盘的读写次数就减少了, 所以``apply``会比``commit``高效.

* ``SharedPreferences``是不支持多进程的, 如果在多进程下读写的话, 可能出现脏读现象. 如果需要在多进程下读写数据的话, 可以借用``ContentProvider``来实现, 数据源设置为``SharedPreferences``即可.
