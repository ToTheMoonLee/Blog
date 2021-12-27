# Glide源码解析

> 基于Glide4.9.0源码解析



参考：

[源码解析：Glide 4.9之缓存策略](https://juejin.cn/post/6844903975947337742)

[Android 图片加载框架 Glide 4.9.0 (二) 从源码的角度分析 Glide 缓存策略](https://juejin.cn/post/6844903953604280328#heading-7)

[面试官：Glide 做了哪些优化？](https://mp.weixin.qq.com/s/ujW9QNc3awH_am5gFzfOhQ)



### 整体流程概括

首先Glide的缓存可以分为如下几层



| 缓存类型          | 对应类              | 备注                                                         |
| ----------------- | ------------------- | ------------------------------------------------------------ |
| 活动缓存          | ActiveResources     | 如果当前对应的图片资源是从内存缓存中获取的，那么会将这个图片存储到活动资源中。 |
| 内存缓存          | LruResourceCache    | 图片解析完成并最近被加载过，则放入内存中                     |
| 磁盘缓存-处理过的 | DiskLruCacheWrapper | 被解码后的图片写入磁盘文件中                                 |
| 磁盘缓存-原始数据 | DiskLruCacheWrapper | 网络请求成功后将原始数据在磁盘中缓存                         |

获取图片时的执行次序如下：

![Glide缓存流程](/Users/lixiangyue/Personal/blog/Blog/源码解析/media/Glide/Glide缓存流程.jpg)

### 执行步骤

#### 1. 生成缓存key

不管是内存缓存还是磁盘缓存，存储的时候肯定需要一个唯一 key 值，直接找到入口函数，即`Engine.load()`函数，具体如下：



```java
public synchronized <R> LoadStatus load() {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
    // 1. 生成唯一的 key 值，用来查找缓存
    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
            resourceClass, transcodeClass, options);
    // 2. 先从活动缓存中查找，ActiveResources
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
        cb.onResourceReady(active, DataSource.MEMORY_CACHE);
        if (VERBOSE_IS_LOGGABLE) {
            logWithTimeAndKey("Loaded resource from active resources", startTime, key);
        }
        return null;
    }
    // 3. 如果活动缓存中没有，就加载 LRU 内存缓存中的资源数据
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
        cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
        if (VERBOSE_IS_LOGGABLE) {
            logWithTimeAndKey("Loaded resource from cache", startTime, key);
        }
        return null;
    }
  
		...
      
    return new LoadStatus(cb, engineJob);
}

// EngineKeyFactory.java

  EngineKey buildKey(Object model, Key signature, int width, int height,
      Map<Class<?>, Transformation<?>> transformations, Class<?> resourceClass,
      Class<?> transcodeClass, Options options) {
    return new EngineKey(model, signature, width, height, transformations, resourceClass,
        transcodeClass, options);
  }

// EngineKey.java

EngineKey(
      Object model,
      Key signature,
      int width,
      int height,
      Map<Class<?>, Transformation<?>> transformations,
      Class<?> resourceClass,
      Class<?> transcodeClass,
      Options options) {
    this.model = Preconditions.checkNotNull(model);
    this.signature = Preconditions.checkNotNull(signature, "Signature must not be null");
    this.width = width;
    this.height = height;
    this.transformations = Preconditions.checkNotNull(transformations);
    this.resourceClass =
        Preconditions.checkNotNull(resourceClass, "Resource class must not be null");
    this.transcodeClass =
        Preconditions.checkNotNull(transcodeClass, "Transcode class must not be null");
    this.options = Preconditions.checkNotNull(options);
  }

@Override
  public boolean equals(Object o) {
    if (o instanceof EngineKey) {
      EngineKey other = (EngineKey) o;
      return model.equals(other.model)
          && signature.equals(other.signature)
          && height == other.height
          && width == other.width
          && transformations.equals(other.transformations)
          && resourceClass.equals(other.resourceClass)
          && transcodeClass.equals(other.transcodeClass)
          && options.equals(other.options);
    }
    return false;
  }

  @Override
  public int hashCode() {
    if (hashCode == 0) {
      hashCode = model.hashCode();
      hashCode = 31 * hashCode + signature.hashCode();
      hashCode = 31 * hashCode + width;
      hashCode = 31 * hashCode + height;
      hashCode = 31 * hashCode + transformations.hashCode();
      hashCode = 31 * hashCode + resourceClass.hashCode();
      hashCode = 31 * hashCode + transcodeClass.hashCode();
      hashCode = 31 * hashCode + options.hashCode();
    }
    return hashCode;
  }
```

可以看到会根据宽、高、解码器、转码器等等一些列参数，共同生成一个key，并且内部重写了hashCode和equals来保证对象的唯一性



#### 2. 内存缓存

默认情况下我们都是使用内存缓存的，继续看`Engine.load()`之后的代码：

```java
public synchronized <R> LoadStatus load() {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
    // 1. 生成唯一的 key 值，用来查找缓存
    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
            resourceClass, transcodeClass, options);
    // 2. 先从活动缓存中查找，ActiveResources
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
        cb.onResourceReady(active, DataSource.MEMORY_CACHE);
        if (VERBOSE_IS_LOGGABLE) {
            logWithTimeAndKey("Loaded resource from active resources", startTime, key);
        }
        return null;
    }
    // 3. 如果活动缓存中没有，就加载 LRU 内存缓存中的资源数据
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
        cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
        if (VERBOSE_IS_LOGGABLE) {
            logWithTimeAndKey("Loaded resource from cache", startTime, key);
        }
        return null;
    }
  
		...
      
    return new LoadStatus(cb, engineJob);
}

    private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }
        // 通过activeResources的get方法获取资源缓存
        EngineResource<?> active = activeResources.get(key);
        if (active != null) {
          // 如果获取到，就引用计数+1
            active.acquire();
        }

        return active;
    }
// EngineResource.java
    synchronized void acquire() {
        if (isRecycled) {
            throw new IllegalStateException("Cannot acquire a recycled resource");
        }
      // 引用计数+1
        ++acquired;
    }

```

在拿到key之后，会先从活动缓存中拿，如果拿到就引用计数+1，并直接返回，本次加载结束；具体内部实现就是通过`activeResources.get(key)`来获取活动缓存中的资源，具体实现如下：



```java
// ActiveResources.java
synchronized EngineResource<?> get(Key key) {
    //通过 HashMap + WeakReference 的存储结构
    //通过 HashMap 的 get 函数拿到活动缓存的弱引用
    ResourceWeakReference activeRef = activeEngineResources.get(key);
    if (activeRef == null) {
        return null;
    }

    EngineResource<?> active = activeRef.get();
    //如果弱引用中关联的EngineResource对象不存在，即EngineResourse被回收
    if (active == null) {
        //清理弱引用缓存，恢复EngineResource，并保存到LruCache缓存中
        cleanupActiveReference(activeRef);
    }
    return active;
}
```

这里具体实现是使用一个HashMap来存储一个指向正在使用的图片的弱引用，而这个弱引用的实现如下：

```java
static final class ResourceWeakReference extends WeakReference<EngineResource<?>> {
        @SuppressWarnings("WeakerAccess")
        @Synthetic
        final Key key;
        @SuppressWarnings("WeakerAccess")
        @Synthetic
        final boolean isCacheable;

        @Nullable
        @SuppressWarnings("WeakerAccess")
        @Synthetic
        Resource<?> resource;

        @Synthetic
        @SuppressWarnings("WeakerAccess")
        ResourceWeakReference(
                @NonNull Key key,
                @NonNull EngineResource<?> referent,
                @NonNull ReferenceQueue<? super EngineResource<?>> queue,
                boolean isActiveResourceRetentionAllowed) {
            super(referent, queue);
            this.key = Preconditions.checkNotNull(key);
            this.resource =
                    referent.isCacheable() && isActiveResourceRetentionAllowed
                            ? Preconditions.checkNotNull(referent.getResource()) : null;
            isCacheable = referent.isCacheable();
        }

        /**
         * 清除数据
         */
        void reset() {
            resource = null;
            clear();
        }
    }
```
 可以发现这个类其实继承了WeakReference，所以当gc发生时，会回收掉ResourceWeakReference对象关联的EngineResource对象，这个对象封装了我们所要获取的图片资源。另外这个类还保存了图片资源和图片资源的缓存key,这是为了当关联的EngineResource对象被回收时，可以利用保存的图片资源来恢复EngineResource对象，然后保存到LruCache缓存中并根据key值从HashMap中删除掉关联了被回收的EngineResource对象的弱引用对象。可以看下EngineResourse被回收的情况：

```java
void cleanupActiveReference(@NonNull ResourceWeakReference ref) {

    synchronized (listener) {
        synchronized (this) {
            //将该弱引用缓存从HashMap中删除
            activeEngineResources.remove(ref.key);

            if (!ref.isCacheable || ref.resource == null) {
                return;
            }
            //恢复EngineResource，其中这个对象封装了图片资源
            EngineResource<?> newResource =
                    new EngineResource<>(ref.resource, /*isCacheable=*/ true, /*isRecyclable=*/ false);
            newResource.setResourceListener(ref.key, listener);
            //回调，该listener为Engine对象
            listener.onResourceReleased(ref.key, newResource);
        }
    }
}

// Engine.java
    public synchronized void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
        // 清除activeResources中对应的图片资源
        activeResources.deactivate(cacheKey);
        if (resource.isCacheable()) {
            // 如果开启了内存缓存，则将其存储到内存缓存
            cache.put(cacheKey, resource);
        } else {
            resourceRecycler.recycle(resource);
        }
    }
```

如果要是弱引用指向的对象被回收了，则会根据内部的缓存的对象，重新构建一个EngineResource并将其加入到LruCache中。



#### 活动资源的存储

弱引用缓存的存储体现在了两个地方：

- 在主线程展示图片前
- 获取LruCache缓存时

**1. 在主线程展示图片前**： 首先是主线程展示图片前，直接定位代码：

```java
// EngineJob.java

void notifyCallbacksOfResult() {
   ... 
    //内部缓存存储的入口
    //实际上调用的是Engine的onEngineJobComplete
    listener.onEngineJobComplete(this, localKey, localResource);

    for (final ResourceCallbackAndExecutor entry : copy) {
        //回到主线程展示照片
        entry.executor.execute(new CallResourceReady(entry.cb));
    }
    //通知上层删除弱引用缓存数据
    decrementPendingCallbacks();
}
```

这里重点是回调了Engine的onEngineJobComplete来存储弱引用缓存，继续看Engine的方法：

```java
// Engine.java

public synchronized void onEngineJobComplete(
        EngineJob<?> engineJob, Key key, EngineResource<?> resource) {
    // A null resource indicates that the load failed, usually due to an exception.
    if (resource != null) {
        resource.setResourceListener(key, this);

        if (resource.isCacheable()) {
            //如果开启内存缓存的话，将解析后的图片添加到弱引用缓存
            activeResources.activate(key, resource);
        }
    }

    jobs.removeIfCurrent(key, engineJob);
}
```

如果开启了内存缓存，则将解析后的图片添加到弱引用

```Java
/**
 * 存入数据
 */
synchronized void activate(Key key, EngineResource<?> resource) {
    ResourceWeakReference toPut =
            new ResourceWeakReference(
                    key, resource, resourceReferenceQueue, isActiveResourceRetentionAllowed);

    ResourceWeakReference removed = activeEngineResources.put(key, toPut);
    if (removed != null) {
        removed.reset();
    }
}
```

这里可以看到，正在使用的图片会存储到弱引用缓存，而不是LruCache中



**2. 获取LruCache缓存时存储**

获取LruCache的代码如下：

```java
private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
    if (!isMemoryCacheable) {
        return null;
    }
    // 内部会将内存缓存的当前的key删除
    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
        // 如果内存中存在，则计数+1
        cached.acquire();
        // 添加到活动缓存中
        activeResources.activate(key, cached);
    }
    return cached;
}


    private EngineResource<?> getEngineResourceFromCache(Key key) {
        // 通过key将图片从内存缓存中删除，并将其返回，从而放到activeResource中
        Resource<?> cached = cache.remove(key);

        final EngineResource<?> result;
        if (cached == null) {
            result = null;
        } else if (cached instanceof EngineResource) {
            // Save an object allocation if we've cached an EngineResource (the typical case).
            result = (EngineResource<?>) cached;
        } else {
            result = new EngineResource<>(cached, true /*isMemoryCacheable*/, true /*isRecyclable*/);
        }
        return result;
    }
```

获取LruCache缓存跟弱引用缓存的获取操作很相似，首先调用了getEngineResourceFromCache来获取图片资源，然后将EngineResource的引用计数加1，并且还会将获取到的图片资源存储到弱引用缓存中。可以看到是通过cache的remove方法来获取该缓存的。



#### 活动资源的删除

弱引用的删除体现在两处：

* JVM进行GC时
* 弱引用对象引用计数为0时

**1. JVM进行GC时**，可以看如下代码；

```java
synchronized EngineResource<?> get(Key key) {
    //通过 HashMap + WeakReference 的存储结构
    //通过 HashMap 的 get 函数拿到活动缓存的弱引用
    ResourceWeakReference activeRef = activeEngineResources.get(key);
    if (activeRef == null) {
        return null;
    }

    EngineResource<?> active = activeRef.get();
    //如果弱引用中关联的EngineResource对象不存在，即EngineResourse被回收
    if (active == null) {
        //清理弱引用缓存，恢复EngineResource，并保存到LruCache缓存中
        cleanupActiveReference(activeRef);
    }
    return active;
}

    void cleanupActiveReference(@NonNull ResourceWeakReference ref) {
        // Fixes a deadlock where we normally acquire the Engine lock and then the ActiveResources lock
        // but reverse that order in this one particular test. This is definitely a bit of a hack...
        synchronized (listener) {
            synchronized (this) {
                //将该弱引用缓存从HashMap中删除
                activeEngineResources.remove(ref.key);

                if (!ref.isCacheable || ref.resource == null) {
                    return;
                }
                //恢复EngineResource，其中这个对象封装了图片资源
                EngineResource<?> newResource =
                        new EngineResource<>(ref.resource, /*isCacheable=*/ true, /*isRecyclable=*/ false);
                newResource.setResourceListener(ref.key, listener);
                //回调，该listener为Engine对象
                listener.onResourceReleased(ref.key, newResource);
            }
        }
    }

    public synchronized void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
        // 清除activeResources中对应的图片资源
        activeResources.deactivate(cacheKey);
        if (resource.isCacheable()) {
            // 如果开启了内存缓存，则将其存储到内存缓存
            cache.put(cacheKey, resource);
        } else {
            resourceRecycler.recycle(resource);
        }
    }
```

可以看到，如果JVM进行GC时，将活动资源的缓存清除了，则弱引用拿到的资源为null，从而调用cleanupActiveReference方法，而在该方法中，会通过弱引用对象中存储的图片资源，来创建一个EngineResource对象，并将其存入LruCache中



**2. 弱引用对象引用计数为0时**

直接定位到最关键的代码，EngineResource#release

```java
void release() {
    // To avoid deadlock, always acquire the listener lock before our lock so that the locking
    // scheme is consistent (Engine -> EngineResource). Violating this order leads to deadlock
    // (b/123646037).
    synchronized (listener) {
        synchronized (this) {
            if (acquired <= 0) {
                throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
            }
            //这里每次调用一次 release 内部引用计数法就会减一，到没有引用也就是为 0 的时候 ，就会通知上层
            if (--acquired == 0) {
                // 该listener为Engine
                listener.onResourceReleased(key, this);
            }
        }
    }
}

    public synchronized void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
        // 清除activeResources中对应的图片资源
        activeResources.deactivate(cacheKey);
        if (resource.isCacheable()) {
            // 如果开启了内存缓存，则将其存储到内存缓存
            cache.put(cacheKey, resource);
        } else {
            resourceRecycler.recycle(resource);
        }
    }
```

这里使用引用计数法，当引用计数为0时，则代表已经不再使用这个对象了，则会回调Engine的onResourceReleased，而在onResourceReleased回调中，会将弱引用删除，并将其添加到LruCache缓存中。

**总结**：

1. 正在使用的图片会加入到弱引用的缓存
2. 不再使用的图片会加入到LruCache缓存
3. 弱引用存入发生在主线程加载图片前、LruCache获取时
4. 弱引用删除发生在JVM的GC、引用计数为0



#### LruCache缓存

**获取**

从Engine的load方法可以看出，当弱引用中拿不到时，才会调用loadFromCache获取LruCache中的资源，直接看代码：

```java
private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
    if (!isMemoryCacheable) {
        return null;
    }
    // 内部会将内存缓存的当前的key删除
    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
        // 如果内存中存在，则计数+1
        cached.acquire();
        // 添加到活动缓存中
        activeResources.activate(key, cached);
    }
    return cached;
}

    private EngineResource<?> getEngineResourceFromCache(Key key) {
        // 通过key将图片从内存缓存中删除，并将其返回，从而放到activeResource中
        Resource<?> cached = cache.remove(key);

        final EngineResource<?> result;
        if (cached == null) {
            result = null;
        } else if (cached instanceof EngineResource) {
            // Save an object allocation if we've cached an EngineResource (the typical case).
            result = (EngineResource<?>) cached;
        } else {
            result = new EngineResource<>(cached, true /*isMemoryCacheable*/, true /*isRecyclable*/);
        }
        return result;
    }
```



这里其实已经分析过了，这里从LruCache中，通过调用remove来拿到缓存，并且将其保存到弱引用的缓存



关于LruCache的**存储**和**删除**其实和弱引用的缓存是相反的

流程图如下（这里直接引用掘金的图片）：

![Glide缓存加载流程](/Users/lixiangyue/Personal/blog/Blog/源码解析/media/Glide/Glide缓存加载流程.jpg)



#### 磁盘缓存

磁盘缓存的策略如下：

| 参数                        | 备注                                                         |
| --------------------------- | ------------------------------------------------------------ |
| DiskCacheStrategy.AUTOMATIC | 这是默认的最优缓存策略：<br/>本地：仅存储转换后的图片（RESOURCE）<br/>网络：仅存储原始图片（DATA）。因为网络获取数据比解析磁盘上的数据要昂贵的多 |
| DiskCacheStrategy.NONE      | 不开启磁盘缓存                                               |
| DiskCacheStrategy.RESOURCE  | 缓存转换过后的图片                                           |
| DiskCacheStrategy.DATA      | 缓存原始图片，即原始输入流。它需要经过压缩转换，解析等操作才能最终展示出来 |
| DiskCacheStrategy.ALL       | 既缓存原始图片，又缓存转换后的图片                           |

如果在内存缓存中加载不到，则要开启线程执行DecodeJob的run方法，最终会调用到ByteBufferFileLoader中的loadData方法，具体路径如下：

```java
public void run() {
    ...
    try {
        if (isCancelled) {
            notifyFailed();
            return;
        }
        runWrapped();
    } catch (CallbackException e) {
        ... 
    }
}

    /**
     * 初始化之后第一次运行时 runReason 为 INITIALIZE
     */
    private void runWrapped() {
        Log.d(TAG, "第 " + runWrappedCount + " 次运行 runWrapped()");
        runWrappedCount++;
        switch (runReason) {
            case INITIALIZE:
                Log.d(TAG, "case:INITIALIZE");
                stage = getNextStage(Stage.INITIALIZE);
                Log.d(TAG, "stage:" + stage);
                currentGenerator = getNextGenerator();
                Log.d(TAG, "currentGenerator:" + currentGenerator);
                runGenerators();
                break;
            case SWITCH_TO_SOURCE_SERVICE:
                Log.d(TAG, "case:SWITCH_TO_SOURCE_SERVICE");
                runGenerators();
                break;
            case DECODE_DATA:
                Log.d(TAG, "case:DECODE_DATA");
                decodeFromRetrievedData();
                break;
            default:
                throw new IllegalStateException("Unrecognized run reason: " + runReason);
        }
    }

    private void runGenerators() {
        Log.d(TAG, "runGenerators()");
        currentThread = Thread.currentThread();
        startFetchTime = LogTime.getLogTime();
        boolean isStarted = false;
        while (!isCancelled && currentGenerator != null
                && !(isStarted = currentGenerator.startNext())) {
            Log.d(TAG, "runGenerators()->while");
            stage = getNextStage(stage);
            currentGenerator = getNextGenerator();

            if (stage == Stage.SOURCE) {
                Log.d(TAG, "runGenerators()->while->stage == Stage.SOURCE");
                reschedule();
                return;
            }
        }
        // We've run out of stages and generators, give up.
        if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
            Log.d(TAG, "runGenerators()->(stage == Stage.FINISHED || isCancelled) && !isStarted");
            notifyFailed();
        }

        // Otherwise a generator started a new load and we expect to be called back in
        // onDataFetcherReady.
    }
```

第一次运行的时候是ResourceCacheGenerator，然后会调用他的startNext方法：

```
public boolean startNext() {
    // GlideUrl
    List<Key> sourceIds = helper.getCacheKeys();
    if (sourceIds.isEmpty()) {
        return false;
    }
    List<Class<?>> resourceClasses = helper.getRegisteredResourceClasses();
    if (resourceClasses.isEmpty()) {
        if (File.class.equals(helper.getTranscodeClass())) {
            return false;
        }
        throw new IllegalStateException(
                "Failed to find any load path from " + helper.getModelClass() + " to "
                        + helper.getTranscodeClass());
    }
    while (modelLoaders == null || !hasNextModelLoader()) {
        resourceClassIndex++;
        if (resourceClassIndex >= resourceClasses.size()) {
            sourceIdIndex++;
            if (sourceIdIndex >= sourceIds.size()) {
                return false;
            }
            resourceClassIndex = 0;
        }

        Key sourceId = sourceIds.get(sourceIdIndex);
        Class<?> resourceClass = resourceClasses.get(resourceClassIndex);
        Transformation<?> transformation = helper.getTransformation(resourceClass);
        // PMD.AvoidInstantiatingObjectsInLoops Each iteration is comparatively expensive anyway,
        // we only run until the first one succeeds, the loop runs for only a limited
        // number of iterations on the order of 10-20 in the worst case.

        // 拿到缓存的key
        currentKey =
                new ResourceCacheKey(// NOPMD AvoidInstantiatingObjectsInLoops
                        helper.getArrayPool(),
                        sourceId,
                        helper.getSignature(),
                        helper.getWidth(),
                        helper.getHeight(),
                        transformation,
                        resourceClass,
                        helper.getOptions());
        // 通过key获取磁盘缓存
        cacheFile = helper.getDiskCache().get(currentKey);
        if (cacheFile != null) {
            sourceKey = sourceId;
            modelLoaders = helper.getModelLoaders(cacheFile);
            modelLoaderIndex = 0;
        }
    }

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
        // 获取数据加载器
        ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
        //  为资源缓存文件，构建一个加载器，这是构建出来的是 ByteBufferFileLoader 的内部类 ByteBufferFetcher
        loadData = modelLoader.buildLoadData(cacheFile,
                helper.getWidth(), helper.getHeight(), helper.getOptions());
        if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
            started = true;
            // 利用 ByteBufferFetcher 加载，最后把结果会通过回调给 DecodeJob 的 onDataFetcherReady 函数
            loadData.fetcher.loadData(helper.getPriority(), this);
        }
    }

    return started;
}
```

首先同样会根据传入的图片宽、高、变换等参数，构建一个key，然后根据这个key从磁盘缓存里拿到对应的文件，并最终通过ByteBufferFileLoader内部的ByteBufferFileLoader来获取磁盘缓存，最终其实就是读流的操作：

```java
public void loadData(@NonNull Priority priority,
    @NonNull DataCallback<? super ByteBuffer> callback) {
  ByteBuffer result;
  try {
    result = ByteBufferUtil.fromFile(file);
  } catch (IOException e) {
    if (Log.isLoggable(TAG, Log.DEBUG)) {
      Log.d(TAG, "Failed to obtain ByteBuffer for file", e);
    }
    callback.onLoadFailed(e);
    return;
  }

  callback.onDataReady(result);
}
```

拿到数据之后就是解析、转换、加载等一些列操作了



**存储**

这里我们直接定位到转换完的图片的存储的地方，即DecodeJob的decodeFromRetrievedData方法：

```java
private void decodeFromRetrievedData() {
    Log.d(TAG, "decodeFromRetrievedData()");
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Retrieved data", startFetchTime,
                "data: " + currentData
                        + ", cache key: " + currentSourceKey
                        + ", fetcher: " + currentFetcher);
    }
    Resource<R> resource = null;
    try {
        // 拿到解码之后的资源
        resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
        e.setLoggingDetails(currentAttemptingKey, currentDataSource);
        throwables.add(e);
    }
    if (resource != null) {
      // 通知外界资源获取成功
        notifyEncodeAndRelease(resource, currentDataSource);
    } else {
        runGenerators();
    }
}

    private void notifyEncodeAndRelease(Resource<R> resource, DataSource dataSource) {
        Log.d(TAG, "notifyEncodeAndRelease()");
        ... 
        try {
            if (deferredEncodeManager.hasResourceToEncode()) {
                // 缓存入口
                deferredEncodeManager.encode(diskCacheProvider, options);
            }
        } finally {
            if (lockedResource != null) {
                lockedResource.unlock();
            }
        }
        // Call onEncodeComplete outside the finally block so that it's not called if the encode process
        // throws.
        onEncodeComplete();
    }

        void encode(DiskCacheProvider diskCacheProvider, Options options) {
            GlideTrace.beginSection("DecodeJob.encode");
            try {
                // 通过DiskCache将数据写入缓存
                diskCacheProvider.getDiskCache().put(key,
                        new DataCacheWriter<>(encoder, toEncode, options));
            } finally {
                toEncode.unlock();
                GlideTrace.endSection();
            }
        }
```

整体流程就是在处理完资源后，通过decodeFromRetrievedData方法中的notifyEncodeAndRelease通知外界缓存磁盘数据，在notifyEncodeAndRelease中调用deferredEncodeManager的encode方法进行文件的存储。



**删除**

由于缓存在了磁盘上，故删除不仅仅由代码控制。常见的删除方式如下：

- 用户主动删除手机上的对应文件
- 卸载软件
- 手动调用Glide.get(applicationContext).clearDiskCache()



#### Data缓存（原始图片）

通过上面的分析我们知道原始图片对应的执行者为DataCacheGenerator，故还是会调用DataCacheGenerator的startNext方法来获取磁盘缓存

```
public boolean startNext() {
    while (modelLoaders == null || !hasNextModelLoader()) {
        sourceIdIndex++;
        if (sourceIdIndex >= cacheKeys.size()) {
            return false;
        }

        Key sourceId = cacheKeys.get(sourceIdIndex);
        // PMD.AvoidInstantiatingObjectsInLoops The loop iterates a limited number of times
        // and the actions it performs are much more expensive than a single allocation.
        @SuppressWarnings("PMD.AvoidInstantiatingObjectsInLoops")
        Key originalKey = new DataCacheKey(sourceId, helper.getSignature());
        cacheFile = helper.getDiskCache().get(originalKey);
        if (cacheFile != null) {
            this.sourceKey = sourceId;
            modelLoaders = helper.getModelLoaders(cacheFile);
            modelLoaderIndex = 0;
        }
    }

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
        ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
        loadData =
                modelLoader.buildLoadData(cacheFile, helper.getWidth(), helper.getHeight(),
                        helper.getOptions());
        if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
            started = true;
            loadData.fetcher.loadData(helper.getPriority(), this);
        }
    }
    return started;
}
```

其实这里跟ResourceCacheGenerator的startNext几乎一模一样，不同的地方是构建key的参数不同，因为原始图片的缓存是不需要宽、高、配置等各种参数的。



**存储**

由于是存储原始数据流，所以直接定位到HttpUrlFetcher的loadData中：

```java
public void loadData(@NonNull Priority priority,
                     @NonNull DataCallback<? super InputStream> callback) {
    long startTime = LogTime.getLogTime();
    try {
        InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
        callback.onDataReady(result);
    } catch (IOException e) {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
            Log.d(TAG, "Failed to load data for url", e);
        }
        callback.onLoadFailed(e);
    } finally {
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "Finished http url fetcher fetch in " + LogTime.getElapsedMillis(startTime));
        }
    }
}
```

拿到原始数据流后，会通过callback.onDataReady进行回调，这里回调的是SourceGenerator的onDataReady方法：

```java
    public void onDataReady(Object data) {
        Log.d(TAG, "onDataReady(Object)....");
        DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
        if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
            //将网络获取到的原始数据，赋值给dataToCache
            dataToCache = data;
            // We might be being called back on someone else's thread. Before doing anything, we should
            // reschedule to get back onto Glide's thread.
            //调用DecodeJob的reschedule，用线程池执行任务，实际上就是再次调用SourceGenerator的startNext
            cb.reschedule();
        } else {
            cb.onDataFetcherReady(loadData.sourceKey, data, loadData.fetcher,
                    loadData.fetcher.getDataSource(), originalKey);
        }
    }

   public void reschedule() {
        Log.d(TAG, "reschedule()");
        runReason = RunReason.SWITCH_TO_SOURCE_SERVICE;
        //此时的callback为EngineJob
        callback.reschedule(this);
    }

    public void reschedule(DecodeJob<?> job) {
        // Even if the job is cancelled here, it still needs to be scheduled so that it can clean itself
        // up.
      // 这里又会调用到SourceGenerator的startNext方法
        getActiveSourceExecutor().execute(job);
    }
```

经过一系列的调用，其实最终还是会调用到SourceGenerator的startNext方法

```java
public boolean startNext() {
    //第二次进入
        //现在dataToCache不等于null，为原始图片
        if (dataToCache != null) {
            Object data = dataToCache;
            dataToCache = null;
            //放入缓存 
            cacheData(data);
        }

    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
        return true;
    }
    sourceCacheGenerator = null;

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
        loadData = helper.getLoadData().get(loadDataListIndex++);
        if (loadData != null
                && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
                || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
            started = true;
            //此处为实际发起请求的部分
            loadData.fetcher.loadData(helper.getPriority(), this);
        }
    }
    return started;
}

    /**
     * 将数据缓存至磁盘缓存
     */
    private void cacheData(Object dataToCache) {
        long startTime = LogTime.getLogTime();
        try {
            Encoder<Object> encoder = helper.getSourceEncoder(dataToCache);
            DataCacheWriter<Object> writer =
                    new DataCacheWriter<>(encoder, dataToCache, helper.getOptions());
            originalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());
            helper.getDiskCache().put(originalKey, writer);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                Log.v(TAG, "Finished encoding source to cache"
                        + ", key: " + originalKey
                        + ", data: " + dataToCache
                        + ", encoder: " + encoder
                        + ", duration: " + LogTime.getElapsedMillis(startTime));
            }
        } finally {
            loadData.fetcher.cleanup();
        }

        sourceCacheGenerator =
                new DataCacheGenerator(Collections.singletonList(loadData.sourceKey), helper, this);
    }
```

最终其实会调用cacheData将数据存储到文件，同样是调用DiskLruCacheWrapper的put方法，将数据存入文件



#### 删除

由于原始图片的缓存也属于磁盘缓存，故跟RESOURCE缓存一样删除不仅仅由代码控制，常见删除方式如下：

- 用户主动删除手机上的对应文件
- 卸载软件
- 手动调用Glide.get(applicationContext).clearDiskCache()

总结图（来自掘金）：

![Glide磁盘缓存加载流程](/Users/lixiangyue/Personal/blog/Blog/源码解析/media/Glide/Glide磁盘缓存加载流程.jpg)

