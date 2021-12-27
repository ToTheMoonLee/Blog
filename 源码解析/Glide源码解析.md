# Glide源码解析

> 基于Glide4.9.0源码解析

### 概述

[Glide github](https://github.com/bumptech/glide)

如今开发最常用的库就是Glide了，图片框架的选择上，一般我们有三个选择Picasso、Glide、Fresco。个人认为，选择Glide的原因第一是，这个框架比Picasso缓存做的好，而且比Picasso加载速度快，同时包体积虽然比Picasso大，但是比Fresco小很多。

### 使用

Glide的使用相当简单，最基本的使用如下：

```
Glide.with(this)
            .load(url)
            .into(imageView)
```

更多详细的用法，直接看Glide的文档即可。下面开始分析Glide的源码结构。



### 问题

1. Glide的整体加载流程
2. 如何替换自定义网络库
3. Glide是如何管理生命周期的
4. Glide是拿到view大小的
5. Glide是如何添加网络监听的



### 最简化的图片加载过程

如果不用Glide这个库，自己写一个最简化的网络图片加载，其实步骤就是：

1. 使用OkHttp或者自己写HttpConnection请求图片的url获取到二进制流（InputStream）
2. 使用BitmapFactory.decodeStream将InputStream解析成Bitmap
3. 加载图片，view.setXXX()

其实Glide最终的过程也是走了这些，只不过首先他做了一系列的优化，比如做了多层的缓存、绑定了Activity的生命周期、监听网络状态、对返回的数据流根据view的大小进行处理和加载等等。同时Glide对这个过程进行了一下抽象，每个过程分别对应多种加载器、解析器处理。比如通过url获取InputStream使用的是HttpGlideUrlLoader类型的数据加载器；而第二个过程会先使用ByteBufferBitmapDecoder将数据转换成InputStream，在调用Downsampler.decode进行转换成Bitmap的过程。

之所以定义多种加载器和解析器，是因为我们获取路径的源是有多种的，比如我们可以通过网络Uri获取（http/https），也可以通过文件Uri（file://），Base64也可以（data:image），其他的还包括从文件获取，asset中获取等等，所以需要多种数据加载器，而最终我们要解析成bitmap或是drawable等，并且还有可能一些图片格式这个库里暂时不支持解析，所以也需要拓展提供解析器。



### Glide对进行图片加载的过程路径



获取数据--->解码数据--->转码数据--->展示到view或者其他地方（Target）

Model -------> Data -------> Resource -------> Transcoded Resource

String -------> InputStream -------> BitmapResource ------> DrawableResource

Model -------> Data 使用 ModelLoader

Data -------> Resource 使用 ResourceDecoder

Resource -------> Transcoded Resource 使用 ResourceTranscoder

将原始数据写入磁盘 使用 Encoder

对原始数据处理过的数据写入磁盘 使用ResourceDecoder

所有以上的这些处理器均在最开始被注册到了`Registry`类，而`Registry`中的`ModelLoaderRegistry`、`ResourceDecoderRegistry`、`TranscoderRegistry`、`EncoderRegistry`、`ResourceEncoderRegistry`分别会讲上面的集中处理器注册缓存。

同时Glide也允许我们使用@GlideModule注解，或者在清单文件中注册的方式，对以上的处理过程进行替换或拓展

### 相关的类

Glide：全局配置的类，Encoder、Decoder、ModelLoader、Pool 等等都在这里设置，此外还提供了创建 RequestManager 的接口（Glide#with() 方法）

GlideBuilder ：是用来创建 Glide 实例的类，其中包含了很多个 get/set 方法，例如设置 BitmapPool、MemoryCache、ArrayPool 等等，最终通过这些设置调用 build 方法构建

RequestManagerRetriever：创建RequestManagerFragment和RequestManager，并将RequestManager设置到RequestManagerFragment中，进而关联生命周期

`RequestManager` ：主要作用**通过生命周期回调启动结束等、监听网络状态、创建请求构造器（RequestBuilder）**

RequestBuilder： 用来构建请求

SingleRequest：一次请求的抽象，定义了开始结束等等

Target： 代表一个**可被 Glide 加载并且具有生命周期的资源**

ImageViewTarget： 通常我们将图片设置到ImageView上对应的就是这个target

ModelLoader：一个数据加载器，将我们传入的人以的Model.class（比如String，inputstream等）转换成可以DataFetcher获取资源的数据模型

LoadData：ModelLoader的内部类，内部有key对象，还有一个DataFetcher，DataFetcher是真正要执行加载资源操作的类（比如网络请求）

DataFetcher：真正执行数据获取的类，如果我们传入的是一个网络请求的url，那么我们使用的就是HttpUrlFetcher

Encoder：用来将数据写入磁盘缓存的

ResourceDecoder：用来将原始数据解析成最终的数据的，网络请求使用的是ByteBufferBitmapDecoder

Engine：Engine 负责管理请求以及活动资源、缓存等。主要关注 load 方法，这个方法主要做了如下几件事：

1. 通过请求构建 Key；
2. 从活动资源中获取资源（详见缓存章节），获取到则返回；
3. 从缓存中获取资源，获取到则直接返回；
4. 判断当前请求是否正在执行，是则直接返回；
5. 构建 EngineJob 与 DecodeJob 并执行。

EngineJob：这个主要用来执行 DecodeJob 以及管理加载完成的回调，各种监听器，没有太多其他的东西。

DecodeJob：负责从缓存或数据源中加载原始数据并通过解码器转换为相应的资源类型（Resource）。DecodeJob 实现了 Runnable 接口，由 EngineJob 将其运行在指定线程池中。



1. `ModelLoaders` to load custom Models (Urls, Uris, arbitrary POJOs) and Data (InputStreams, FileDescriptors).
2. `ResourceDecoders` to decode new Resources (Drawables, Bitmaps) or new types of Data (InputStreams, FileDescriptors).
3. `Encoders` to write Data (InputStreams, FileDescriptors) to Glide’s disk cache.
4. `ResourceTranscoders` to convert Resources (BitmapResource) into other types of Resources (DrawableResource).
5. `ResourceEncoders` to write Resources (BitmapResource, DrawableResource) to Glide’s disk cache.

**Anatomy of a load**
The set of registered components, including both those registered by default in Glide and those registered in Modules are used to define a set of load paths. Each load path is a step by step progression from the the Model provided to `load()` to the Resource type specified by `as()`. A load path consists (roughly) of the following steps:
\1. Model -> Data (handled by `ModelLoader`s) 2. Data -> Resource (handled by `ResourceDecoder`s) 3. Resource -> Transcoded Resource (optional, handled by `ResourceTranscoder`s).

`Encoder`s can write Data to Glide’s disk cache cache before step 2.
`ResourceEncoder`s can write Resource’s to Glide’s disk cache before step 3.
When a request is started, Glide will attempt all available paths from the Model to the requested Resource type. A request will succeed if any load path succeeds. A request will fail only if all available load paths fail.



### 源码分析

首先先看下`Glide.with(this)`

```
public static RequestManager with(@NonNull Context context) {
  return getRetriever(context).get(context);
}

public static RequestManager with(@NonNull Activity activity) {
	return getRetriever(activity).get(activity);
}

public static RequestManager with(@NonNull FragmentActivity activity) {
	return getRetriever(activity).get(activity);
}

public static RequestManager with(@NonNull Fragment fragment) {
	return getRetriever(fragment.getActivity()).get(fragment);
}

public static RequestManager with(@NonNull android.app.Fragment fragment) {
	return getRetriever(fragment.getActivity()).get(fragment);
}
```

`with()`方法有很多重载方法，主要常用的是Activity、Fragment、Context这三种形式的，其中可以看出逻辑基本相同，下面以Activity参数的为例，查看`getRetriever()`做了什么

```
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    return Glide.get(context).getRequestManagerRetriever();
  }
```

这个方法在注释中写明了，所传参数`context`虽然可能是空的，一般只有在生命周期出现问题的时候，才会出现。继续跟进`Glide.get(context)`

```
  public static Glide get(@NonNull Context context) {
    if (glide == null) {
      synchronized (Glide.class) {
        if (glide == null) {
          checkAndInitializeGlide(context);
        }
      }
    }
    return glide;
  }
```

可以看到，这里是要初始化一个Glide的单例对象，继续跟进`checkAndInitializeGlide(context)`

```
  private static void checkAndInitializeGlide(@NonNull Context context) {
    ...
    initializeGlide(context);
    ...
  }
  
  private static void initializeGlide(@NonNull Context context) {
  	// 这里传入了一个GlideBuilder对象，会配置一些默认信息，调用build()方法时会初始化
    initializeGlide(context, new GlideBuilder());
  }

  @SuppressWarnings("deprecation")
  private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
  	// 拿到应用级别的上下文，避免内存泄漏，实际开发时，我们也可以通过这种形式拿到上下文
    Context applicationContext = context.getApplicationContext();
    // 拿到@GlideModule 标识的注解处理器生成的 GeneratedAppGlideModuleImpl、GeneratedAppGlideModuleFactory等
    // @GlideModule可以进行全局配置，通过注解的方式进行了解耦
    // 之后的代码是如果annotationGeneratedModule为空，则使用ManifestParser对menifest文件进行解析
    // 最后再将生成的RequestManagerFactory对象塞入到builder中
    GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();
    List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
      // 如果Impl存在，且允许解析manifest文件
  // 则遍历manifest中的meta-data，解析出所有的GlideModule类
    if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
      manifestModules = new ManifestParser(applicationContext).parse();
    }

// 根据Impl的黑名单，剔除manifest中的GlideModule类
    if (annotationGeneratedModule != null
        && !annotationGeneratedModule.getExcludedModuleClasses().isEmpty()) {
      Set<Class<?>> excludedModuleClasses =
          annotationGeneratedModule.getExcludedModuleClasses();
      Iterator<com.bumptech.glide.module.GlideModule> iterator = manifestModules.iterator();
      while (iterator.hasNext()) {
        com.bumptech.glide.module.GlideModule current = iterator.next();
        if (!excludedModuleClasses.contains(current.getClass())) {
          continue;
        }
        if (Log.isLoggable(TAG, Log.DEBUG)) {
          Log.d(TAG, "AppGlideModule excludes manifest GlideModule: " + current);
        }
        iterator.remove();
      }
    }

    if (Log.isLoggable(TAG, Log.DEBUG)) {
      for (com.bumptech.glide.module.GlideModule glideModule : manifestModules) {
        Log.d(TAG, "Discovered GlideModule from manifest: " + glideModule.getClass());
      }
    }

// 如果Impl存在，那么设置为该类的RequestManagerFactory； 否则，设置为null
    RequestManagerRetriever.RequestManagerFactory factory =
        annotationGeneratedModule != null
            ? annotationGeneratedModule.getRequestManagerFactory() : null;
    builder.setRequestManagerFactory(factory);
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
      module.applyOptions(applicationContext, builder);
    }
    // 写入Impl的配置
  // 也就是说Impl配置的优先级更高，如果有冲突的话
    if (annotationGeneratedModule != null) {
      annotationGeneratedModule.applyOptions(applicationContext, builder);
    }
    // 调用build()方法，传入applicationContext来初始化Glide，其中包含了一系列的默认参数
    Glide glide = builder.build(applicationContext);
    // 依次调用manifest中GlideModule类的registerComponents方法，来替换Glide的默认配置
    // 进行注册组件的操作，可以实现自定义替换一些组件，比如OkHttp
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
      module.registerComponents(applicationContext, glide, glide.registry);
    }
    // 调用Impl中替换Glide配置的方法
    if (annotationGeneratedModule != null) {
      annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
    }
    // 注册内存管理的回调，因为Glide实现了ComponentCallbacks2接口
    applicationContext.registerComponentCallbacks(glide);
    // 保存glide实例到静态变量中
    Glide.glide = glide;
  }
```

如果没有使用过自定义的Module，则可以代码可以简化为：

```
@SuppressWarnings("deprecation")
private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
  Context applicationContext = context.getApplicationContext();
  // 调用GlideBuilder.build方法创建Glide
  Glide glide = builder.build(applicationContext);
  // 注册内存管理的回调，因为Glide实现了ComponentCallbacks2接口
  applicationContext.registerComponentCallbacks(glide);
  // 保存glide实例到静态变量中
  Glide.glide = glide;
}
```

总结一下，这个方法主要做了如下几件事：

1. 拿到@GlideModule注解标志的生成类，通过该生成类拿到RequestManagerFactory
2. 使用RequestManagerFactory添加到GlideBuilder
3. 通过Builder初始化Glide
4. 注册组件回调，比如替换为OkHttp，具体作用见[Glide自定义模块](https://www.jianshu.com/p/a42ecf4af737)



下面继续看下builder的相关参数：

```
Glide build(@NonNull Context context) {
	// 初始化一些线程池
    if (sourceExecutor == null) {
      sourceExecutor = GlideExecutor.newSourceExecutor();
    }

    if (diskCacheExecutor == null) {
      diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
    }

    if (animationExecutor == null) {
      animationExecutor = GlideExecutor.newAnimationExecutor();
    }
	// 根据一些常量和屏幕密度、宽高等参数动态计算cache大小
    if (memorySizeCalculator == null) {
      memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
    }
	// 网络监听工厂类
    if (connectivityMonitorFactory == null) {
      connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
    }
	// bitmap缓存池
    if (bitmapPool == null) {
      int size = memorySizeCalculator.getBitmapPoolSize();
      if (size > 0) {
        bitmapPool = new LruBitmapPool(size);
      } else {
        bitmapPool = new BitmapPoolAdapter();
      }
    }

    if (arrayPool == null) {
      arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
    }
	// Resource的缓存池
    if (memoryCache == null) {
      memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }
		// 硬盘缓存工厂
    if (diskCacheFactory == null) {
      diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }
		
    if (engine == null) {
      engine =
          new Engine(
              memoryCache,
              diskCacheFactory,
              diskCacheExecutor,
              sourceExecutor,
              GlideExecutor.newUnlimitedSourceExecutor(),
              GlideExecutor.newAnimationExecutor(),
              isActiveResourceRetentionAllowed);
    }

    if (defaultRequestListeners == null) {
      defaultRequestListeners = Collections.emptyList();
    } else {
      defaultRequestListeners = Collections.unmodifiableList(defaultRequestListeners);
    }
		// 用来生成RequestManager，使用静态方法，或者从activity或fragment中获取
		// 传入的requestManagerFactory如果设置了就不为null，用用户自定义的，未设置默认为null，则会使用默认的工厂创建RequestManager
    RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory);
		// 初始化Glide
    return new Glide(
        context,
        engine,
        memoryCache,
        bitmapPool,
        arrayPool,
        requestManagerRetriever,
        connectivityMonitorFactory,
        logLevel,
        defaultRequestOptions.lock(),
        defaultTransitionOptions,
        defaultRequestListeners,
        isLoggingRequestOriginsEnabled);
  }
```

这里可以看到`build()`中主要是构建了线程池、缓存池、RequestManager等，最终会用这些生成的对象，来初始化一个Glide对象

```
Glide(
      @NonNull Context context,
      @NonNull Engine engine,
      @NonNull MemoryCache memoryCache,
      @NonNull BitmapPool bitmapPool,
      @NonNull ArrayPool arrayPool,
      @NonNull RequestManagerRetriever requestManagerRetriever,
      @NonNull ConnectivityMonitorFactory connectivityMonitorFactory,
      int logLevel,
      @NonNull RequestOptions defaultRequestOptions,
      @NonNull Map<Class<?>, TransitionOptions<?, ?>> defaultTransitionOptions,
      @NonNull List<RequestListener<Object>> defaultRequestListeners,
      boolean isLoggingRequestOriginsEnabled) {
    this.engine = engine;
    this.bitmapPool = bitmapPool;
    this.arrayPool = arrayPool;
    this.memoryCache = memoryCache;
    this.requestManagerRetriever = requestManagerRetriever;
    this.connectivityMonitorFactory = connectivityMonitorFactory;
		// 初始化解析器
    DecodeFormat decodeFormat = defaultRequestOptions.getOptions().get(Downsampler.DECODE_FORMAT);
    bitmapPreFiller = new BitmapPreFiller(memoryCache, bitmapPool, decodeFormat);

    final Resources resources = context.getResources();
		// 这里会将一些默认的Loader，Decoder，Encoder或者Loader工厂类等添加到注册表，已备之后使用
    registry = new Registry();
    ...
    ContentResolver contentResolver = context.getContentResolver();
		...
		// 初始化Target的工厂
    ImageViewTargetFactory imageViewTargetFactory = new ImageViewTargetFactory();
    // 初始化glideContext
    glideContext =
        new GlideContext(
            context,
            arrayPool,
            registry,
            imageViewTargetFactory,
            defaultRequestOptions,
            defaultTransitionOptions,
            defaultRequestListeners,
            engine,
            isLoggingRequestOriginsEnabled,
            logLevel);
  }
```

可以看到，这里主要是初始化了默认Loader、Decoder、Transcoder等，然后将其添加到注册表，已备之后使用，用户可以自己定制Loader、Decoder等。继续返回去看`Glide.with()`之后的代码`getRetriever(activity).get(activity)`，首先说明下，RequestManager可以用来管理和开始一些请求requests，可以绑定activity和fragment的生命周期

```
getRetriever(activity)实际是返回了requestManagerRetriever，也就是在builder中初始化的那个对象

  public RequestManager get(@NonNull Activity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      // 这里是生命周期绑定的地方，首先拿到FragmentManager
      android.app.FragmentManager fm = activity.getFragmentManager();
      // 然后调用fragmentGet获取RequestManager
      return fragmentGet(
          activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }
```

继续查看`fragmentGet()`的逻辑

```
 private RequestManager fragmentGet(@NonNull Context context,
      @NonNull android.app.FragmentManager fm,
      @Nullable android.app.Fragment parentHint,
      boolean isParentVisible) {
      // 首先要拿到一个RequestManagerFragment，这个类实际继承了Fragment类
    RequestManagerFragment current = getRequestManagerFragment(fm, parentHint, isParentVisible);
    // 尝试从RequestManagerFragment中拿到RequestManager
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      // TODO(b/27524013): Factor out this Glide.get() call.
      Glide glide = Glide.get(context);
      // RequestManager没有获取到的话，就创建一个RequestManager，并将其添加到RequestManagerFragment中
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      current.setRequestManager(requestManager);
    }
    return requestManager;
  }
```

可以看到这里的逻辑是：

1. 首先先通过`getRequestManagerFragment`拿到一个RequestManagerFragment，RequestManagerFragment实际就是一个Fragment；

2. 尝试从RequestManagerFragment拿到一个ReqeustManger，如果没拿到，则使用工厂类初始化一个出来，并讲其设置给RequestManagerFragment

   

继续看下`getRequestManagerFragment()`的逻辑：

```
  private RequestManagerFragment getRequestManagerFragment(
      @NonNull final android.app.FragmentManager fm,
      @Nullable android.app.Fragment parentHint,
      boolean isParentVisible) {
      // 1. 尝试从FragmentManager中通过Tag找到已经创建的RequestManagerFragment
    RequestManagerFragment current = (RequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
      current = pendingRequestManagerFragments.get(fm);
      if (current == null) {
      // 2. 如果没有找到，则创建一个RequestManagerFragment
        current = new RequestManagerFragment();
        current.setParentFragmentHint(parentHint);
        if (isParentVisible) {
          current.getGlideLifecycle().onStart();
        }
        pendingRequestManagerFragments.put(fm, current);
        // 使用FragmentManager将Fragment添加到Activity中
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();
      }
    }
    return current;
  }
```

逻辑相对来说比较简单，能从FragmentManager中找到RequestManagerFragment则直接返回，如果没有就新建一个，并添加到Activity中，此时Fragment就与Activity的生命周期绑定，并返回，接下来看下`factory.build(glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context)`的代码，其实内部核心代码就是new了一个ReqeustManger类

```
RequestManager(
      Glide glide,
      Lifecycle lifecycle,
      RequestManagerTreeNode treeNode,
      RequestTracker requestTracker,
      ConnectivityMonitorFactory factory,
      Context context) {
    this.glide = glide;
    this.lifecycle = lifecycle;
    this.treeNode = treeNode;
    this.requestTracker = requestTracker;
    this.context = context;
	... 
		// 创建网络监听的监视器
	    connectivityMonitor =
        factory.build(
            context.getApplicationContext(),
            new RequestManagerConnectivityListener(requestTracker));
	
    if (Util.isOnBackgroundThread()) {
      mainHandler.post(addSelfToLifecycle);
    } else {
    // 核心就是将this添加到lifecycle中，而这个lifecycle就是RequestManagerFragment.getGlideLifecycle()
      lifecycle.addListener(this);
    }
    lifecycle.addListener(connectivityMonitor);

    defaultRequestListeners =
        new CopyOnWriteArrayList<>(glide.getGlideContext().getDefaultRequestListeners());
    setRequestOptions(glide.getGlideContext().getDefaultRequestOptions());

    glide.registerRequestManager(this);
  }
```

核心就是将RequestManager添加到RequestManagerFragment中的lifecycle中，这里其实就是用到了观察者模式，相当于RequestMangerFragment中有一个观察者的list，也就是lifecycle(ActivityFragmentLifecycle)，这里调用`addListener(this)`，就相当于将RequestManager（也是一个ActivityFragmentLifecycle）添加到了观察者列表，而在RequestManagerFragment的生命周期中，会调用lifecycle的回调方法。

`fragmentGet`方法逻辑的最后，调用`current.setRequestManager(requestManager);`，最终将RequestManager和RequestManagerFragment实现了绑定。

同时会初始化一个DefaultConnectivityMonitor也添加到lifecycle中，在onstart的时候会注册connectivityReceiver，在onstop的时候会注销connectivityReceiver，同时DefaultConnectivityMonitor也会监听网络状态，调用onConnectivityChanged来重新出发请求

总结一下，`with`方法，实际会初始化一系列线程池、缓存池等相关配置，并将请求管理与生命周期绑定，从而不需要我们关注生命周期的问题。



接下来我们看下`load(url)`又做了些什么，load也有很多重载方法，我们这里看String参数的：

```
  public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
  }
  
  public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
  }
  
  public <ResourceType> RequestBuilder<ResourceType> as(
      @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
  }
  
  protected RequestBuilder(
      @NonNull Glide glide,
      RequestManager requestManager,
      Class<TranscodeType> transcodeClass,
      Context context) {
    this.glide = glide;
    this.requestManager = requestManager;
    this.transcodeClass = transcodeClass;
    this.context = context;
    this.transitionOptions = requestManager.getDefaultTransitionOptions(transcodeClass);
    this.glideContext = glide.getGlideContext();

    initRequestListeners(requestManager.getDefaultRequestListeners());
    apply(requestManager.getDefaultRequestOptions());
  }
  
   public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
  }
    private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    this.model = model;
    isModelSet = true;
    return this;
  }
```

这段逻辑最终会根据requestManager中的一些默认配置来创建一个RequestBuilder，而`load()`方法则会将我们传入的string存如到model中，并返回一个RequestBuilder。之后会调用RequestBuilder.load方法：

```
  // RequestBuilder.java
  public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
  }
  // 只是保存了参数Model
    private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    this.model = model;
    isModelSet = true;
    return this;
  }
```



最后看下这个代码逻辑最多的`into`的逻辑，只看下最基本的主流程：

```
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    Util.assertMainThread();
    Preconditions.checkNotNull(view);

    BaseRequestOptions<?> requestOptions = this;
     // 若没有指定transform，isTransformationSet()为false
  // isTransformationAllowed()一般为true，除非主动调用了dontTransform()方法
    if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      // Clone in this method so that if we use this RequestBuilder to load into a View and then
      // into a different target, we don't retain the transformation applied based on the previous
      // View's scale type.
      // 根据ImageView的ScaleType设置不同的down sample和transform选项
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case FIT_CENTER: // 默认是FIT_CENTER
        case FIT_START:
        case FIT_END:
          requestOptions = requestOptions.clone().optionalFitCenter();
          break;
        case FIT_XY:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case CENTER:
        case MATRIX:
        default:
          // Do nothing.
      }
    }
 	// 调用重载方法
    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
  }
```

代码逻辑分为两部：第一，使用目标imageview的ScaleType属性，对requestOptions重新配置；第二，调用into方法。在继续往下看之前，先看下FIT_CENTER中的逻辑：

```
@NonNull
@CheckResult
public T optionalFitCenter() {
  return optionalScaleOnlyTransform(DownsampleStrategy.FIT_CENTER, new FitCenter());
}

@NonNull
private T optionalScaleOnlyTransform(
    @NonNull DownsampleStrategy strategy, @NonNull Transformation<Bitmap> transformation) {
  return scaleOnlyTransform(strategy, transformation, false /*isTransformationRequired*/);
}

@SuppressWarnings("unchecked")
@NonNull
private T scaleOnlyTransform(
    @NonNull DownsampleStrategy strategy,
    @NonNull Transformation<Bitmap> transformation,
    boolean isTransformationRequired) {
  BaseRequestOptions<T> result = isTransformationRequired
        ? transform(strategy, transformation) : optionalTransform(strategy, transformation);
  result.isScaleOnlyOrNoTransform = true;
  return (T) result;
}

@SuppressWarnings({"WeakerAccess", "CheckResult"})
@NonNull
final T optionalTransform(@NonNull DownsampleStrategy downsampleStrategy,
    @NonNull Transformation<Bitmap> transformation) {
  // isAutoCloneEnabled默认为false，只有在主动调用了autoClone()方法之后才会为true
  if (isAutoCloneEnabled) {
    return clone().optionalTransform(downsampleStrategy, transformation);
  }

  downsample(downsampleStrategy);
  return transform(transformation, /*isRequired=*/ false);
}

@NonNull
@CheckResult
public T downsample(@NonNull DownsampleStrategy strategy) {
  return set(DownsampleStrategy.OPTION, Preconditions.checkNotNull(strategy));
}

@NonNull
@CheckResult
public <Y> T set(@NonNull Option<Y> option, @NonNull Y value) {
  if (isAutoCloneEnabled) {
    return clone().set(option, value);
  }

  Preconditions.checkNotNull(option);
  Preconditions.checkNotNull(value);
  options.set(option, value);
  return selfOrThrowIfLocked();
}

@NonNull
T transform(
    @NonNull Transformation<Bitmap> transformation, boolean isRequired) {
  if (isAutoCloneEnabled) {
    return clone().transform(transformation, isRequired);
  }

  DrawableTransformation drawableTransformation =
      new DrawableTransformation(transformation, isRequired);
  transform(Bitmap.class, transformation, isRequired);
  transform(Drawable.class, drawableTransformation, isRequired);
  // TODO: remove BitmapDrawable decoder and this transformation.
  // Registering as BitmapDrawable is simply an optimization to avoid some iteration and
  // isAssignableFrom checks when obtaining the transformation later on. It can be removed without
  // affecting the functionality.
  transform(BitmapDrawable.class, drawableTransformation.asBitmapDrawable(), isRequired);
  transform(GifDrawable.class, new GifDrawableTransformation(transformation), isRequired);
  return selfOrThrowIfLocked();
}

@NonNull
<Y> T transform(
    @NonNull Class<Y> resourceClass,
    @NonNull Transformation<Y> transformation,
    boolean isRequired) {
  if (isAutoCloneEnabled) {
    return clone().transform(resourceClass, transformation, isRequired);
  }

  Preconditions.checkNotNull(resourceClass);
  Preconditions.checkNotNull(transformation);
  transformations.put(resourceClass, transformation);
  fields |= TRANSFORMATION;
  isTransformationAllowed = true;
  fields |= TRANSFORMATION_ALLOWED;
  // Always set to false here. Known scale only transformations will call this method and then
  // set isScaleOnlyOrNoTransform to true immediately after.
  isScaleOnlyOrNoTransform = false;
  if (isRequired) {
    fields |= TRANSFORMATION_REQUIRED;
    isTransformationRequired = true;
  }
  return selfOrThrowIfLocked();
}
```

一顿操作下来，就是保存了几个值到`BaseRequestOptions`内部的两个`CachedHashCodeArrayMap`里面，其中键值对以及保存到的位置如下：

| 保存的位置      | K                         | V                                                            |
| :-------------- | ------------------------- | ------------------------------------------------------------ |
| Options.values  | DownsampleStrategy.OPTION | DownsampleStrategy.FitCenter()                               |
| transformations | Bitmap.class              | FitCenter()                                                  |
| transformations | Drawable.class            | DrawableTransformation(FitCenter(), false)                   |
| transformations | BitmapDrawable.class      | DrawableTransformation(FitCenter(), false).asBitmapDrawable() |
| transformations | GifDrawable.class         | GifDrawableTransformation(FitCenter())                       |

将KV保存好了之后，就准备调用最终的`into`方法了，我们看一下入参：

```
into(
    glideContext.buildImageViewTarget(view, transcodeClass),
    /*targetListener=*/ null,
    requestOptions,
    Executors.mainThreadExecutor());
```

第一个参数等于`(ViewTarget<ImageView, Drawable>) new DrawableImageViewTarget(view)`：

```
  // GlideContext.java
  public <X> ViewTarget<ImageView, X> buildImageViewTarget(
      @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
       // imageViewTargetFactory是ImageViewTargetFactory的一个实例
  // transcodeClass在RequestManager.load方法中确定了，就是Drawable.class
    return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
  }
  // ImageViewTargetFactory,其中clazz为Drawable.class
  public <Z> ViewTarget<ImageView, Z> buildTarget(@NonNull ImageView view,
      @NonNull Class<Z> clazz) {
    if (Bitmap.class.equals(clazz)) {
      return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
    } else if (Drawable.class.isAssignableFrom(clazz)) {
     // 返回的是(ViewTarget<ImageView, Drawable>) new DrawableImageViewTarget(view);
      return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
    } else {
      throw new IllegalArgumentException(
          "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
    }
  }
```

其中`Drawable.class.isAssignableFrom(clazz)`代码的意思就是看下Drawable.class是否是clazz的同类或者是父类。

`Executors.mainThreadExecutor()`就是一个使用MainLooper的Handler，在execute Runnable时使用此Handler post出去。

```
/** Posts executions to the main thread. */
  public static Executor mainThreadExecutor() {
    return MAIN_THREAD_EXECUTOR;
  }

  private static final Executor MAIN_THREAD_EXECUTOR =
      new Executor() {
        private final Handler handler = new Handler(Looper.getMainLooper());

        @Override
        public void execute(@NonNull Runnable command) {
          handler.post(command);
        }
      };
```

下面点进去`into`方法看下：

```
// RequestBuilder.java
private <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,
      Executor callbackExecutor) {
    Preconditions.checkNotNull(target);
    if (!isModelSet) {
      throw new IllegalArgumentException("You must call #load() before calling #into()");
    }
		// 这里实际上创建了一个SingleRequest
    Request request = buildRequest(target, targetListener, options, callbackExecutor);
 // 这里会判断需不需要重新开始任务
  // 如果当前request和target上之前的request previous相等
  // 且设置了忽略内存缓存或previous还没有完成
  // 那么会进入if分支，无需进行一些相关设置，这是一个很好的优化
    Request previous = target.getRequest();
    if (request.isEquivalentTo(previous)
        && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
      request.recycle();
      // If the request is completed, beginning again will ensure the result is re-delivered,
      // triggering RequestListeners and Targets. If the request is failed, beginning again will
      // restart the request, giving it another chance to complete. If the request is already
      // running, we can let it continue running without interruption.
      // 如果正在运行，就不管它；如果已经失败了，就重新开始
      if (!Preconditions.checkNotNull(previous).isRunning()) {
        // Use the previous request rather than the new one to allow for optimizations like skipping
        // setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
        // that are done in the individual Request.
        previous.begin();
      }
      return target;
    }
  // 如果不能复用previous
  // 先清除target上之前的Request
    requestManager.clear(target);
    // 将Request作为tag设置到view中
    target.setRequest(request);
    // 真正开始网络图片的加载
    requestManager.track(target, request);

    return target;
  }
```

接下来看下buildRequest的流程：

```

private Request buildRequestRecursive(
      Target<TranscodeType> target,
      @Nullable RequestListener<TranscodeType> targetListener,
      @Nullable RequestCoordinator parentCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight,
      BaseRequestOptions<?> requestOptions,
      Executor callbackExecutor) {

    // Build the ErrorRequestCoordinator first if necessary so we can update parentCoordinator.
    ErrorRequestCoordinator errorRequestCoordinator = null;
    // errorBuilder为null, skip
    // 因此errorRequestCoordinator为null
    if (errorBuilder != null) {
      errorRequestCoordinator = new ErrorRequestCoordinator(parentCoordinator);
      parentCoordinator = errorRequestCoordinator;
    }
		// 获得SingleRequest的地方
    Request mainRequest =
        buildThumbnailRequestRecursive(
            target,
            targetListener,
            parentCoordinator,
            transitionOptions,
            priority,
            overrideWidth,
            overrideHeight,
            requestOptions,
            callbackExecutor);

    if (errorRequestCoordinator == null) {
      return mainRequest;
    }
		...
  }

  private Request buildThumbnailRequestRecursive(
      Target<TranscodeType> target,
      RequestListener<TranscodeType> targetListener,
      @Nullable RequestCoordinator parentCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight,
      BaseRequestOptions<?> requestOptions,
      Executor callbackExecutor) {
    // thumbnail重载方法没有调用过，所以会走最后的else case
    if (thumbnailBuilder != null) {
      ...
    } else if (thumbSizeMultiplier != null) {
      ... 
    } else {
      // Base case: no thumbnail.
      // 这里直接返回一个SingleRequest，且将状态设置成status = Status.PENDING
      return obtainRequest(
          target,
          targetListener,
          requestOptions,
          parentCoordinator,
          transitionOptions,
          priority,
          overrideWidth,
          overrideHeight,
          callbackExecutor);
    }
  }
  
  private Request obtainRequest(
      Target<TranscodeType> target,
      RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> requestOptions,
      RequestCoordinator requestCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight,
      Executor callbackExecutor) {
    return SingleRequest.obtain(
        context,
        glideContext,
        model,
        transcodeClass,
        requestOptions,
        overrideWidth,
        overrideHeight,
        priority,
        target,
        targetListener,
        requestListeners,
        requestCoordinator,
        glideContext.getEngine(),
        transitionOptions.getTransitionFactory(),
        callbackExecutor);
  }
```



接下来看下看下真正开始图片加载的地方：



```
  // RequestManager.java
  synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
    targetTracker.track(target);
    requestTracker.runRequest(request);
  }
```

在这里面，`targetTracker`成员变量在声明的时候直接初始化为`TargetTracker`类的无参数实例，该类的作用是保存所有的Target并向它们转发生命周期事件。

还记得刚开始初始化RequestManager的地方吧，RequestManager会被绑定到Fragment中，进而绑定Activity的生命周期，而RequestManager会在onstart方法中调用targetTracker的onstart，targetTracker的start进而会调用target的onstart方法。代码如下：

```
  public synchronized void onStart() {
    resumeRequests();
    targetTracker.onStart();
  }
```



接下来继续看requestTracker.runRequest(request);

```
// RequestTracker.java

public void runRequest(@NonNull Request request) {
    requests.add(request);
    // isPaused默认为false
    if (!isPaused) {
      request.begin();
    } else {
      request.clear();
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Paused, delaying request");
      }
      pendingRequests.add(request);
    }
  }
```

`isPaused`默认为false，只有调用了`RequestTracker.pauseRequests`或`RequestTracker.pauseAllRequests`后才会为true。因此，下面会执行`request.begin()`方法。上面说到过，这里的request实际上是`SingleRequest`对象，我们看一下它的`SingleRequest#begin()`方法。

```
// SingleRequest.java
public synchronized void begin() {
    assertNotCallingCallbacks();
    stateVerifier.throwIfRecycled();
    startTime = LogTime.getLogTime();
    // 如果model为空，会调用监听器的onLoadFailed处理
  	// 若无法处理，则展示失败时的占位图
    if (model == null) {
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        width = overrideWidth;
        height = overrideHeight;
      }
      // Only log at more verbose log levels if the user has set a fallback drawable, because
      // fallback Drawables indicate the user expects null models occasionally.
      int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
      onLoadFailed(new GlideException("Received null model"), logLevel);
      return;
    }

    if (status == Status.RUNNING) {
      throw new IllegalArgumentException("Cannot restart a running request");
    }

    // If we're restarted after we're complete (usually via something like a notifyDataSetChanged
    // that starts an identical request into the same Target or View), we can simply use the
    // resource and size we retrieved the last time around and skip obtaining a new size, starting a
    // new load etc. This does mean that users who want to restart a load because they expect that
    // the view size has changed will need to explicitly clear the View or Target before starting
    // the new load.
    		// 首次请求是未完成的状态
    if (status == Status.COMPLETE) {
      onResourceReady(resource, DataSource.MEMORY_CACHE);
      return;
    }

    // Restarts for requests that are neither complete nor running can be treated as new requests
    // and can run again from the beginning.
	// 请求没有完成也没有在运行，就当作新请求来对待。此时可以从beginning开始运行
  // 如果指定了overrideWidth和overrideHeight，那么直接调用onSizeReady方法
  // 否则会获取ImageView的宽、高，然后调用onSizeReady方法
  // 在该方法中会创建图片加载的Job并开始执行
    status = Status.WAITING_FOR_SIZE;
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
      onSizeReady(overrideWidth, overrideHeight);
    } else {
      target.getSize(this);
    }
		// 显示加载中的占位符
    if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
        && canNotifyStatusChanged()) {
      target.onLoadStarted(getPlaceholderDrawable());
    }
    if (IS_VERBOSE_LOGGABLE) {
      logV("finished run method in " + LogTime.getElapsedMillis(startTime));
    }
  }
```

首先看下失败的情况

```
private synchronized void onLoadFailed(GlideException e, int maxLogLevel) {
  stateVerifier.throwIfRecycled();
  e.setOrigin(requestOrigin);
  int logLevel = glideContext.getLogLevel();
  if (logLevel <= maxLogLevel) {
    Log.w(GLIDE_TAG, "Load failed for " + model + " with size [" + width + "x" + height + "]", e);
    if (logLevel <= Log.INFO) {
      e.logRootCauses(GLIDE_TAG);
    }
  }

  // 设置状态为Status.FAILED
  loadStatus = null;
  status = Status.FAILED;

  isCallingCallbacks = true;
  try {
    //TODO: what if this is a thumbnail request?
    // 尝试调用各个listener的onLoadFailed回调进行处理
    boolean anyListenerHandledUpdatingTarget = false;
    if (requestListeners != null) {
      for (RequestListener<R> listener : requestListeners) {
        anyListenerHandledUpdatingTarget |=
            listener.onLoadFailed(e, model, target, isFirstReadyResource());
      }
    }
    anyListenerHandledUpdatingTarget |=
        targetListener != null
            && targetListener.onLoadFailed(e, model, target, isFirstReadyResource());

    // 如果没有一个回调能够处理，那么显示失败占位符
    if (!anyListenerHandledUpdatingTarget) {
      setErrorPlaceholder();
    }
  } finally {
    isCallingCallbacks = false;
  }

  // 通知requestCoordinator，此请求失败
  notifyLoadFailed();
}

private void notifyLoadFailed() {
  if (requestCoordinator != null) {
    requestCoordinator.onRequestFailed(this);
  }
}
看一下setErrorPlaceholder中显示失败占位符的逻辑：


private synchronized void setErrorPlaceholder() {
  if (!canNotifyStatusChanged()) {
    return;
  }

  Drawable error = null;
  if (model == null) {
    error = getFallbackDrawable();
  }
  // Either the model isn't null, or there was no fallback drawable set.
  if (error == null) {
    error = getErrorDrawable();
  }
  // The model isn't null, no fallback drawable was set or no error drawable was set.
  if (error == null) {
    error = getPlaceholderDrawable();
  }
  target.onLoadFailed(error);
}

private synchronized void setErrorPlaceholder() {
    if (!canNotifyStatusChanged()) {
      return;
    }

    Drawable error = null;
    if (model == null) {
      error = getFallbackDrawable();
    }
    // Either the model isn't null, or there was no fallback drawable set.
    if (error == null) {
      error = getErrorDrawable();
    }
    // The model isn't null, no fallback drawable was set or no error drawable was set.
    if (error == null) {
      error = getPlaceholderDrawable();
    }
    target.onLoadFailed(error);
  }
  
  // ImageTarget.java
    public void onLoadFailed(@Nullable Drawable errorDrawable) {
    super.onLoadFailed(errorDrawable);
    setResourceInternal(null);
    setDrawable(errorDrawable);
  }
  
    public void setDrawable(Drawable drawable) {
    view.setImageDrawable(drawable);
  }

```

显而易见，当model为null时，失败占位符的显示逻辑如下：

1. 如果设置了fallback，那么显示fallback
2. 否则，如果设置了error，那么显示error
3. 否则，如果设置了placeholder，那么显示placeholder



继续看下没有设置size的情况，也就是调用target.getSize(this);，这个target是DrawableImageViewTarget类型，DrawableImageViewTarget继承子ViewTarget：

```
  // ViewTarget.java
  // 这里的SizeReadyCallback回调传入的是SingleRequest
  public void getSize(@NonNull SizeReadyCallback cb) {
    sizeDeterminer.getSize(cb);
  }
  // ViewTarget#SizeDeterminer.java
  void getSize(@NonNull SizeReadyCallback cb) {
  		// 先从view中拿，拿到则调用回调的onSizeReady
      int currentWidth = getTargetWidth();
      int currentHeight = getTargetHeight();
      if (isViewStateAndSizeValid(currentWidth, currentHeight)) {
        cb.onSizeReady(currentWidth, currentHeight);
        return;
      }

      // We want to notify callbacks in the order they were added and we only expect one or two
      // callbacks to be added a time, so a List is a reasonable choice.
      // 如果拿不到就把这个回调加入到cbs的List中
      if (!cbs.contains(cb)) {
        cbs.add(cb);
      }
      // 通过ViewTreeObserver监听来拿到宽高
      if (layoutListener == null) {
        ViewTreeObserver observer = view.getViewTreeObserver();
        layoutListener = new SizeDeterminerLayoutListener(this);
        observer.addOnPreDrawListener(layoutListener);
      }
    }
    
    // ViewTarget#SizeDeterminerLayoutListener.java中的监听，回调
    public boolean onPreDraw() {
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
          Log.v(TAG, "OnGlobalLayoutListener called attachStateListener=" + this);
        }
        SizeDeterminer sizeDeterminer = sizeDeterminerRef.get();
        if (sizeDeterminer != null) {
          sizeDeterminer.checkCurrentDimens();
        }
        return true;
      }
      
      //  ViewTarget#SizeDeterminer.java
          void checkCurrentDimens() {
      if (cbs.isEmpty()) {
        return;
      }

      int currentWidth = getTargetWidth();
      int currentHeight = getTargetHeight();
      if (!isViewStateAndSizeValid(currentWidth, currentHeight)) {
        return;
      }
			// 最终将宽高回调给SingleRequest.java
      notifyCbs(currentWidth, currentHeight);
      clearCallbacksAndListener();
    }
```



可以看到，这里拿到宽高的方法，如果刚开始拿不到的话，就是使用ViewTreeObserver来获取的。

在看onSizeReady中的代码前，先看下之后这段代码

```
if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
    && canNotifyStatusChanged()) {
  target.onLoadStarted(getPlaceholderDrawable());
}
```

由于执行了onSizeReady之后，status会被赋值为RUNNING，所以会调用`target.onLoadStarted(getPlaceholderDrawable());`来加载placeholder的图片。具体为ImageViewTarget中的，因为BitmapImageViewTarget继承自ImageViewTarget：

```
public void onLoadCleared(@Nullable Drawable placeholder) {
  super.onLoadCleared(placeholder);
  if (animatable != null) {
    animatable.stop();
  }
  setResourceInternal(null);
  setDrawable(placeholder);
}
```



接下来来看下SingleRequest的onSizeReady()方法：

```
public synchronized void onSizeReady(int width, int height) {
  stateVerifier.throwIfRecycled();
  if (IS_VERBOSE_LOGGABLE) {
    logV("Got onSizeReady in " + LogTime.getElapsedMillis(startTime));
  }
  // 在SingleRequest.begin方法中已经将status设置为WAITING_FOR_SIZE状态了，继续往下执行
  if (status != Status.WAITING_FOR_SIZE) {
    return;
  }
  status = Status.RUNNING;

	// 拿到一个合适的缩放尺寸
  float sizeMultiplier = requestOptions.getSizeMultiplier();
  this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
  this.height = maybeApplySizeMultiplier(height, sizeMultiplier);

  if (IS_VERBOSE_LOGGABLE) {
    logV("finished setup for calling load in " + LogTime.getElapsedMillis(startTime));
  }
  
  // 使用各种参数，使用Engine类进行加载
  loadStatus =
      engine.load(
          glideContext,
          model,
          requestOptions.getSignature(),
          this.width,
          this.height,
          requestOptions.getResourceClass(),
          transcodeClass,
          priority,
          requestOptions.getDiskCacheStrategy(),
          requestOptions.getTransformations(),
          requestOptions.isTransformationRequired(),
          requestOptions.isScaleOnlyOrNoTransform(),
          requestOptions.getOptions(),
          requestOptions.isMemoryCacheable(),
          requestOptions.getUseUnlimitedSourceGeneratorsPool(),
          requestOptions.getUseAnimationPool(),
          requestOptions.getOnlyRetrieveFromCache(),
          this,
          callbackExecutor);

  // This is a hack that's only useful for testing right now where loads complete synchronously
  // even though under any executor running on any thread but the main thread, the load would
  // have completed asynchronously.
  if (status != Status.RUNNING) {
    loadStatus = null;
  }
  if (IS_VERBOSE_LOGGABLE) {
    logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
  }
}
```

至此，我们拿到了view的宽高，构建了request，接下来继续看下核心的Engin#load()方法

`Engine`负责开始加载，管理active、cached状态资源的类。先回头看下`GlideBuilder.build`中创建`Glide`时的代码，若没有主动设置engine，会使用下面的参数进行创建：

```
if (sourceExecutor == null) {
  sourceExecutor = GlideExecutor.newSourceExecutor();
}

if (diskCacheExecutor == null) {
  diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
}

if (memoryCache == null) {
  memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
}

if (diskCacheFactory == null) {
  diskCacheFactory = new InternalCacheDiskCacheFactory(context);
}

if (engine == null) {
  engine =
      new Engine(
          memoryCache,
          diskCacheFactory,
          diskCacheExecutor,
          sourceExecutor,
          GlideExecutor.newUnlimitedSourceExecutor(),
          GlideExecutor.newAnimationExecutor(),
          isActiveResourceRetentionAllowed /* 默认为false */);
}
```

可以看到，构建了四个线程池，并将其传入到Engine中。

`Engine.load`方法中会以一些参数作为key，依次从active状态、cached状态和进行中的load里寻找。若没有找到，则会创建对应的job并开始执行。
提供给一个或以上请求且没有被释放的资源被称为active资源。一旦所有的消费者都释放了该资源，该资源就会被放入cache中。如果有请求将资源从cache中取出，它会被重新添加到active资源中。如果一个资源从cache中移除，其本身会被discard，其内部拥有的资源将会回收或者在可能的情况下重用。并没有严格要求消费者一定要释放它们的资源，所以active资源会以弱引用的方式保持。

```
public synchronized <R> LoadStatus load(
    GlideContext glideContext,
    Object model,
    Key signature,
    int width,
    int height,
    Class<?> resourceClass,
    Class<R> transcodeClass,
    Priority priority,
    DiskCacheStrategy diskCacheStrategy,
    Map<Class<?>, Transformation<?>> transformations,
    boolean isTransformationRequired,
    boolean isScaleOnlyOrNoTransform,
    Options options,
    boolean isMemoryCacheable,
    boolean useUnlimitedSourceExecutorPool,
    boolean useAnimationPool,
    boolean onlyRetrieveFromCache,
    ResourceCallback cb,
    Executor callbackExecutor) {
  long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
	// 使用model、宽、高、资源类型等等参数，来创建一个key
  EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
      resourceClass, transcodeClass, options);
	// 从active资源中进行加载，第一次显然取不到
  EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
  if (active != null) {
    cb.onResourceReady(active, DataSource.MEMORY_CACHE);
    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Loaded resource from active resources", startTime, key);
    }
    return null;
  }
	// 从内存cache资源中进行加载，第一次显然取不到
  EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
  if (cached != null) {
    cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Loaded resource from cache", startTime, key);
    }
    return null;
  }
	// 从正在进行的jobs中进行加载，第一次显然取不到
  EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
  if (current != null) {
    current.addCallback(cb, callbackExecutor);
    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Added to existing load", startTime, key);
    }
    return new LoadStatus(cb, current);
  }
  // 构建出一个EngineJob
  EngineJob<R> engineJob =
      engineJobFactory.build(
          key,
          isMemoryCacheable,
          useUnlimitedSourceExecutorPool,
          useAnimationPool,
          onlyRetrieveFromCache);
	// 构建出一个DecodeJob，该类实现了Runnable接口
  DecodeJob<R> decodeJob =
      decodeJobFactory.build(
          glideContext,
          model,
          key,
          signature,
          width,
          height,
          resourceClass,
          transcodeClass,
          priority,
          diskCacheStrategy,
          transformations,
          isTransformationRequired,
          isScaleOnlyOrNoTransform,
          onlyRetrieveFromCache,
          options,
          engineJob);
	// 根据engineJob.onlyRetrieveFromCache的值是否为true
  // 将engineJob保存到onlyCacheJobs或者jobs HashMap中
  jobs.put(key, engineJob);
	// callbackExecutor参数为 com.bumptech.glide.util.Executors#DIRECT_EXECUTOR
	// cb是SingleRequest
	// 最终会将两个参数封装 new ResourceCallbackAndExecutor(cb, executor)
	// 并保存到ResourceCallbacksAndExecutors.callbacksAndExecutors中
  engineJob.addCallback(cb, callbackExecutor);
  engineJob.start(decodeJob);

  if (VERBOSE_IS_LOGGABLE) {
    logWithTimeAndKey("Started new load", startTime, key);
  }
  return new LoadStatus(cb, engineJob);
}
```

上述创建EngineJob和DecodeJob使用了两个工厂类engineJobFactory，decodeJobFactory，里面使用对象池，之后再分析，接下来开始看engineJob.start(decodeJob)方法：

```
  // EngineJob.java 
  public synchronized void start(DecodeJob<R> decodeJob) {
    this.decodeJob = decodeJob;
    // 使用默认缓存策略的话，返回RESOURCE_CACHE
    GlideExecutor executor = decodeJob.willDecodeFromCache()
        ? diskCacheExecutor
        : getActiveSourceExecutor();
    executor.execute(decodeJob);
  }
  
    boolean willDecodeFromCache() {
    Stage firstStage = getNextStage(Stage.INITIALIZE);
    return firstStage == Stage.RESOURCE_CACHE || firstStage == Stage.DATA_CACHE;
  }
  
  // 这个方法之后会多次用到，分别尝试从各个缓存拿图片
  private Stage getNextStage(Stage current) {
    // diskCacheStrategy为DiskCacheStrategy.AUTOMATIC
    // decodeCachedResource()为true
    switch (current) {
      case INITIALIZE:
        return diskCacheStrategy.decodeCachedResource()
            ? Stage.RESOURCE_CACHE : getNextStage(Stage.RESOURCE_CACHE);
      case RESOURCE_CACHE:
        return diskCacheStrategy.decodeCachedData()
            ? Stage.DATA_CACHE : getNextStage(Stage.DATA_CACHE);
      case DATA_CACHE:
        // Skip loading from source if the user opted to only retrieve the resource from cache.
        return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
      case SOURCE:
      case FINISHED:
        return Stage.FINISHED;
      default:
        throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
  }
```

可以看出最终会使用diskCacheExecutor这个线程池来执行decodeJob，在看decodeJob的run()方法之前，先看下各个状态的含义，因为之后可能多次用到。

上面各个阶段的含义是：

```
private enum Stage {
  /** The initial stage. */
  // 初始阶段
  INITIALIZE,
  /** Decode from a cached resource. */
  // 从处理过的资源缓存拿
  RESOURCE_CACHE,
  /** Decode from cached source data. */
  // 从原始数据缓存拿
  DATA_CACHE,
  /** Decode from retrieved source. */
  // 从源数据拿（比如网络获取）
  SOURCE,
  /** Encoding transformed resources after a successful load. */
  // 缓存转换过的数据
  ENCODE,
  /** No more viable stages. */
  // 最终阶段了
  FINISHED,
}
```

接下来，我们可以开始看DecodeJob#run()方法了：

```
public void run() {
    // This should be much more fine grained, but since Java's thread pool implementation silently
    // swallows all otherwise fatal exceptions, this will at least make it obvious to developers
    // that something is failing.
    GlideTrace.beginSectionFormat("DecodeJob#run(model=%s)", model);
    // Methods in the try statement can invalidate currentFetcher, so set a local variable here to
    // ensure that the fetcher is cleaned up either way.
    // 首次执行，currentFetcher目前为null
    DataFetcher<?> localFetcher = currentFetcher;
    try {
      if (isCancelled) {
        notifyFailed();
        return;
      }
      // 重点就是包裹了这句代码
      runWrapped();
    } catch (CallbackException e) {
      // If a callback not controlled by Glide throws an exception, we should avoid the Glide
      // specific debug logic below.
      throw e;
    } catch (Throwable t) {
      // Catch Throwable and not Exception to handle OOMs. Throwables are swallowed by our
      // usage of .submit() in GlideExecutor so we're not silently hiding crashes by doing this. We
      // are however ensuring that our callbacks are always notified when a load fails. Without this
      // notification, uncaught throwables never notify the corresponding callbacks, which can cause
      // loads to silently hang forever, a case that's especially bad for users using Futures on
      // background threads.
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(TAG, "DecodeJob threw unexpectedly"
            + ", isCancelled: " + isCancelled
            + ", stage: " + stage, t);
      }
      // When we're encoding we've already notified our callback and it isn't safe to do so again.
      if (stage != Stage.ENCODE) {
        throwables.add(t);
        notifyFailed();
      }
      if (!isCancelled) {
        throw t;
      }
      throw t;
    } finally {
      // Keeping track of the fetcher here and calling cleanup is excessively paranoid, we call
      // close in all cases anyway.
      if (localFetcher != null) {
        localFetcher.cleanup();
      }
      GlideTrace.endSection();
    }
  }
```

真正执行的就是`runWrapped()`方法：

```
private void runWrapped() {
  switch (runReason) {
  // runReason在DecodeJob.init方法中被初始化为INITIALIZE
    case INITIALIZE:
    	// getNextStage在上面已经看过，会返回RESOURCE_CACHE
      stage = getNextStage(Stage.INITIALIZE);
      // 这里会拿到一个ResourceCacheGenerator生成器
      currentGenerator = getNextGenerator();
      runGenerators();
      break;
    case SWITCH_TO_SOURCE_SERVICE:
      runGenerators();
      break;
    case DECODE_DATA:
      decodeFromRetrievedData();
      break;
    default:
      throw new IllegalStateException("Unrecognized run reason: " + runReason);
  }
}

// 返回一个ResourceCacheGenerator
private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
      case RESOURCE_CACHE:
        return new ResourceCacheGenerator(decodeHelper, this);
      case DATA_CACHE:
        return new DataCacheGenerator(decodeHelper, this);
      case SOURCE:
        return new SourceGenerator(decodeHelper, this);
      case FINISHED:
        return null;
      default:
        throw new IllegalStateException("Unrecognized stage: " + stage);
    }
  }
```

第一次会先拿到一个ResourceCacheGenerator，尝试去加载本地磁盘缓存的处理过的文件数据，我们看下runGenerators方法：

```
private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    while (!isCancelled && currentGenerator != null
    		// 重点是这里的startNext方法，如果isStarted为true，那么就跳出循环，如果为false，就执行下一个阶段的Generator
        && !(isStarted = currentGenerator.startNext())) {
      stage = getNextStage(stage);
      currentGenerator = getNextGenerator();

      if (stage == Stage.SOURCE) {
        reschedule();
        return;
      }
    }
    // We've run out of stages and generators, give up.
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
      notifyFailed();
    }

    // Otherwise a generator started a new load and we expect to be called back in
    // onDataFetcherReady.
  }
```

可以看到，在该方法中会依次调用各个状态生成的`DataFetcherGenerator`的`startNext()`尝试fetch数据，直到有某个状态的`DataFetcherGenerator.startNext()`方法可以胜任。若状态抵达到了`Stage.FINISHED`或job被取消，且所有状态的`DataFetcherGenerator.startNext()`都无法满足条件，则调用`SingleRequest.onLoadFailed`进行错误处理。

这里总共有三个`DataFetcherGenerator`，依次是：

1. ResourceCacheGenerator
   获取采样后、transformed后资源文件的缓存文件
2. DataCacheGenerator
   获取原始的没有修改过的资源文件的缓存文件
3. SourceGenerator
   获取原始源数据

在继续查看currentGenerator.startNext()之前，有必要先看下Registry类，该类是用来管理Glide注册进来的用来拓展或替代Glide默认加载、解码、编码逻辑的组件。在Glide创建的时候，绝大多数代码都是对`Registry`的操作。

大致说一下`Registry`类里面各个组件的功能

Glide在创建时，会向`Registry`实例中注入相当多的配置，每个配置都会转发给对应的一个专门的类，这些专门的类有7个。
在这7个类中，除了`DataRewinderRegistry`和`ImageHeaderParserRegistry`外，其他的Registry都会将注入的配置保存到内部的`Entry`类中。`Entry`类的作用就是判断该项配置能够满足条件（`handles`）。
`handles`方法都是以`isAssignableFrom`方法判断，但被判断参数有一些差别。isAssignableFrom上面说过，就是查看调用的对象是否是传入的参数的同类或者父类，比如`Object.class.isAssignableFrom(String.class);`返回true。

在各个Registry中`Entry`类的`hanldes`实现前，先理解一下这里面出现的各种class：

首先看下注入的方法：

```
  public <Model, Data> Registry append(
      @NonNull Class<Model> modelClass, @NonNull Class<Data> dataClass,
      @NonNull ModelLoaderFactory<Model, Data> factory) {
    modelLoaderRegistry.append(modelClass, dataClass, factory);
    return this;
  }
```

真正代码中调用的modelClass和dataClass的注入方式是Registry.append(String.class, InputStream.class, new DataUrlLoader.StreamFactory<String>())

- `modelClass`
  代表原始请求时传入的类型，也即Glide.with(..).load()中被load参数的类型，在本例中就是String.class。
- `dataClass`
  比较原始的数据，根据`ModelLoaderRegistry`中的配置，可以由modelClass获得的dataClass
  同时，一起注入的`ModelLoaderFactory<Model, Data>`是一个可以创建如何从`modelClass`到`dataClass`进行转换的`ModelLoader<Model, Data>`的类的工厂，可以根据`modelClass`获得
- `resourceClass`
  解码后的资源类型，根据`ResourceDecoderRegistry`中的配置，可以由`dataClass`和`resourceClass`获得所有`registeredResourceClasses`
  同时，一起注入的`ResourceDecoder<Data, Resource>`是一个将`dataClass`解码成`resourceClass`的类，可以由有可能到达的`dataClass`以及`registeredResourceClass`获得
- `transcodeClass`
  最终要转换成的数据类型，一般情况下Drawable.class；若Glide加载是指定了asBitmap、asGif、asFile，那么此类型就是Bitmap.class、GifDrawable.class、File.class
  若`registeredResourceClass`不是`transcodeClass`类型，则通过`TranscoderRegistry`注入的`ResourceTranscoder<Resource, T>`将`resourceClass`转为`transcodeClass`



下面是5个在各个Registry中`Entry`类的`hanldes`实现：

```
// MultiModelLoaderFactory.Entry
private static class Entry<Model, Data> {
  private final Class<Model> modelClass;
  @Synthetic final Class<Data> dataClass;
  ...
  public boolean handles(@NonNull Class<?> modelClass, @NonNull Class<?> dataClass) {
    return handles(modelClass) && this.dataClass.isAssignableFrom(dataClass);
  }

  public boolean handles(@NonNull Class<?> modelClass) {
    return this.modelClass.isAssignableFrom(modelClass);
  }
}

// EncoderRegistry.Entry
private static final class Entry<T> {
  private final Class<T> dataClass;
  ...
  boolean handles(@NonNull Class<?> dataClass) {
    return this.dataClass.isAssignableFrom(dataClass);
  }
}

// ResourceEncoderRegistry.Entry
private static final class Entry<T> {
  private final Class<T> resourceClass;
  ...
  @Synthetic
  boolean handles(@NonNull Class<?> resourceClass) {
    return this.resourceClass.isAssignableFrom(resourceClass);
  }
}

// ResourceDecoderRegistry.Entry
private static class Entry<T, R> {
  private final Class<T> dataClass;
  @Synthetic final Class<R> resourceClass;
  ...
  public boolean handles(@NonNull Class<?> dataClass, @NonNull Class<?> resourceClass) {
    return this.dataClass.isAssignableFrom(dataClass) && resourceClass
        .isAssignableFrom(this.resourceClass);
  }
}

// TranscoderRegistry.Entry
private static final class Entry<Z, R> {
  private final Class<Z> fromClass;
  private final Class<R> toClass;
  ...
  public boolean handles(@NonNull Class<?> fromClass, @NonNull Class<?> toClass) {
    return this.fromClass.isAssignableFrom(fromClass) && toClass.isAssignableFrom(this.toClass);
  }
}
```



这7个Registry类作用如下：

1. ModelLoaderRegistry
   构建`modelClass`到`dataClass`的桥梁
   *load custom Models (Urls, Uris, arbitrary POJOs) and Data (InputStreams, FileDescriptors).*
2. ResourceDecoderRegistry
   `dataClass`到`resourceClass`的桥梁
   *to decode new Resources (Drawables, Bitmaps) or new types of Data (InputStreams, FileDescriptors).*
3. EncoderRegistry
   *write Data (InputStreams, FileDescriptors) to Glide’s disk cache*
4. TranscoderRegistry
   构建`resourceClass`到`transcodeClass`的桥梁
   *convert Resources (BitmapResource) into other types of Resources (DrawableResource)*
5. ResourceEncoderRegistry
   *write Resources (BitmapResource, DrawableResource) to Glide’s disk cache.*
6. DataRewinderRegistry
   提供对`ByteBuffer.class`、`InputStream.class`这两种data进行rewind操作的能力
7. ImageHeaderParserRegistry
   提供解析Image头信息的能力



了解完这些之后，继续查看ResourceCacheGenerator#startNext()方法，先过一下外层的逻辑，再看具体逻辑：

```
public boolean startNext() {
		// list中实际最终只保存了GlideUrl对象
    List<Key> sourceIds = helper.getCacheKeys();
    if (sourceIds.isEmpty()) {
      return false;
    }
    // 获得了三个可以到达的registeredResourceClasses
	  // GifDrawable、Bitmap、BitmapDrawable
    List<Class<?>> resourceClasses = helper.getRegisteredResourceClasses();
    if (resourceClasses.isEmpty()) {
      if (File.class.equals(helper.getTranscodeClass())) {
        return false;
      }
      throw new IllegalStateException(
         "Failed to find any load path from " + helper.getModelClass() + " to "
             + helper.getTranscodeClass());
    }
    // 遍历sourceIds中的每一个key、resourceClasses中每一个class，以及其他的一些值组成key
	  // 尝试在磁盘缓存中以key找到缓存文件
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
      cacheFile = helper.getDiskCache().get(currentKey);
      // 如果找到了缓存文件，那么循环条件则会为false，也就退出循环了
      if (cacheFile != null) {
        sourceKey = sourceId;
        modelLoaders = helper.getModelLoaders(cacheFile);
        modelLoaderIndex = 0;
      }
    }
// 找没找到缓存文件，都会执行这里的方法
 // 如果找到了，hasNextModelLoader()方法则会为true，可以执行循环
 // 没有找到缓存文件，则不会进入循环，会直接返回false
    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
      //  在循环中会依次判断某个ModelLoader能不能加载此文件
      loadData = modelLoader.buildLoadData(cacheFile,
          helper.getWidth(), helper.getHeight(), helper.getOptions());
      if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
      // 如果某个ModelLoader可以，那么就调用其fetcher进行加载数据
     // 加载成功或失败会通知自身
        started = true;
        loadData.fetcher.loadData(helper.getPriority(), this);
      }
    }

    return started;
  }
```



接下来看下 helper.getCacheKeys是如何将String.class转换成GlideUrl对象的

```
List<Key> getCacheKeys() {
 // 这里使用了一个标志位，防止在DataCacheGenerator中重复加载
 if (!isCacheKeysSet) {
   isCacheKeysSet = true;
   cacheKeys.clear();
   // 得到可以处理该请求的ModelLoader的LoadData list
   List<LoadData<?>> loadData = getLoadData();
   // 将每一个loadData里的sourceKey以及每一个alternateKeys添加到cacheKeys中
   // 在我们的三步例子中sourceKey为一个GlideUrl，alternateKeys为空
   for (int i = 0, size = loadData.size(); i < size; i++) {
     LoadData<?> data = loadData.get(i);
     if (!cacheKeys.contains(data.sourceKey)) {
       cacheKeys.add(data.sourceKey);
     }
     for (int j = 0; j < data.alternateKeys.size(); j++) {
       if (!cacheKeys.contains(data.alternateKeys.get(j))) {
         cacheKeys.add(data.alternateKeys.get(j));
       }
     }
   }
 }
 return cacheKeys;
}

List<LoadData<?>> getLoadData() {
 // 这里也使用了一个标志位，防止在重复加载
 if (!isLoadDataSet) {
   isLoadDataSet = true;
   loadData.clear();
   // 获得了注册的3个ModelLoader
   List<ModelLoader<Object, ?>> modelLoaders = glideContext.getRegistry().getModelLoaders(model);
   for (int i = 0, size = modelLoaders.size(); i < size; i++) {
     ModelLoader<Object, ?> modelLoader = modelLoaders.get(i);
     // 对每个ModelLoader调用buildLoadData，看看其是否可以满足条件
     // 如果返回不为null，说明是可以处理的，那么添加进来
     LoadData<?> current =
         modelLoader.buildLoadData(model, width, height, options);
     if (current != null) {
       loadData.add(current);
     }
   }
 }
 // 返回可以处理该请求的ModelLoader的LoadData列表
 return loadData;
}
```

可以看到，主体的逻辑是

1. 从Glide初始化的Registry中拿到modelLoaders
2. 查看这些modelLoaders中的modelLoader是否满足条件，满足则加入到loadDatLlist
3. 最后将loadDatLlist中的LoadData对象的sourceKey添加到cacheKeys集合中

上面比较麻烦的代码就是`glideContext.getRegistry().getModelLoaders(model)`获取ModelLoaders的部分，在看这个之前，还是需要返回`Registry`类的相关代码

`Registry`类中提供了很多用来拓展、替换默认组件的方法，根据组件功能的不同，会交给内部很多不同的Registry处理：

```
public class Registry {
 private final ModelLoaderRegistry modelLoaderRegistry;
 private final EncoderRegistry encoderRegistry;
 private final ResourceDecoderRegistry decoderRegistry;
 private final ResourceEncoderRegistry resourceEncoderRegistry;
 private final DataRewinderRegistry dataRewinderRegistry;
 private final TranscoderRegistry transcoderRegistry;
 private final ImageHeaderParserRegistry imageHeaderParserRegistry;
}
```

拿马上要遇到的`modelLoaderRegistry`来说，相关的管理组件的三个方法为：

```
@NonNull
public <Model, Data> Registry append(
   @NonNull Class<Model> modelClass, @NonNull Class<Data> dataClass,
   @NonNull ModelLoaderFactory<Model, Data> factory) {
 modelLoaderRegistry.append(modelClass, dataClass, factory);
 return this;
}

@NonNull
public <Model, Data> Registry prepend(
   @NonNull Class<Model> modelClass, @NonNull Class<Data> dataClass,
   @NonNull ModelLoaderFactory<Model, Data> factory) {
 modelLoaderRegistry.prepend(modelClass, dataClass, factory);
 return this;
}

@NonNull
public <Model, Data> Registry replace(
   @NonNull Class<Model> modelClass,
   @NonNull Class<Data> dataClass,
   @NonNull ModelLoaderFactory<? extends Model, ? extends Data> factory) {
 modelLoaderRegistry.replace(modelClass, dataClass, factory);
 return this;
}
```

上面这三个方法实际上又会交给`MultiModelLoaderFactory`来处理，这是一个代理模式：

```java
public synchronized <Model, Data> void append(
   @NonNull Class<Model> modelClass,
   @NonNull Class<Data> dataClass,
   @NonNull ModelLoaderFactory<? extends Model, ? extends Data> factory) {
 multiModelLoaderFactory.append(modelClass, dataClass, factory);
 cache.clear();
}

public synchronized <Model, Data> void prepend(
   @NonNull Class<Model> modelClass,
   @NonNull Class<Data> dataClass,
   @NonNull ModelLoaderFactory<? extends Model, ? extends Data> factory) {
 multiModelLoaderFactory.prepend(modelClass, dataClass, factory);
 cache.clear();
}

public synchronized <Model, Data> void remove(@NonNull Class<Model> modelClass,
   @NonNull Class<Data> dataClass) {
 tearDown(multiModelLoaderFactory.remove(modelClass, dataClass));
 cache.clear();
}
```

`MultiModelLoaderFactory`中会使用一个`entries`的list来保存所有注入的内容。

前面已经提到过，Glide在构造时会对`Registry`进行大量的操作。因为我们示例是load的`String`类型的Url，也就是说，`modelClass` 为 `String.class`，所以`Registry`中符合的注册项只有4个：

```Java
// model可能是一个base64格式的img，经过处理后可以变成InputStream类型(dataClass)的数据
.append(String.class, InputStream.class, new DataUrlLoader.StreamFactory<String>())
// model可能是一个uri，这种情况下可能性非常多，因为网络图片、assets图片、磁盘图片等等都是一个uri
.append(String.class, InputStream.class, new StringLoader.StreamFactory())
// model可能是一个本地图片、assets图片，所以可以处理成ParcelFileDescriptor.class类型的数据
.append(String.class, ParcelFileDescriptor.class, new StringLoader.FileDescriptorFactory())
// model可能是一个本地图片，所以可以处理成为AssetFileDescriptor.class类型的数据
.append(
   String.class, AssetFileDescriptor.class, new StringLoader.AssetFileDescriptorFactory())
```

了解了这些之后，我们回过头看看`Registry.getModelLoaders`方法干了啥：

```java
// Registry.java
@NonNull
public <Model> List<ModelLoader<Model, ?>> getModelLoaders(@NonNull Model model) {
 List<ModelLoader<Model, ?>> result = modelLoaderRegistry.getModelLoaders(model);
 if (result.isEmpty()) {
   throw new NoModelLoaderAvailableException(model);
 }
 return result;
}
```

这里只是调用了`modelLoaderRegistry.getModelLoaders(model)`，如果返回结果不为空则返回该结果，否则抛出异常。我们继续跟踪一下：

```
// ModelLoaderRegistry.java
// getModelLoaders方法会获取所有声明可以处理String类型的ModelLoader，并调用handles方法过滤掉肯定不能处理的
@NonNull
public <A> List<ModelLoader<A, ?>> getModelLoaders(@NonNull A model) {
 // 返回所有注册过的modelClass为String的ModelLoader，就是上面列出来的四个
 List<ModelLoader<A, ?>> modelLoaders = getModelLoadersForClass(getClass(model));
 int size = modelLoaders.size();
 boolean isEmpty = true;
 List<ModelLoader<A, ?>> filteredLoaders = Collections.emptyList();
 for (int i = 0; i < size; i++) {
   ModelLoader<A, ?> loader = modelLoaders.get(i);
   // 对于每个ModelLoader，看看是否可能处理这种类型的数据
   // 此处会过滤第一个，因为我们传入的url不以data:image开头
   if (loader.handles(model)) {
     if (isEmpty) {
       filteredLoaders = new ArrayList<>(size - i);
       isEmpty = false;
     }
     filteredLoaders.add(loader);
   }
 }
 return filteredLoaders;
}

// 返回所有声明可以处理modelClass为String的ModelLoader
@NonNull
private synchronized <A> List<ModelLoader<A, ?>> getModelLoadersForClass(
   @NonNull Class<A> modelClass) {
 List<ModelLoader<A, ?>> loaders = cache.get(modelClass);
 if (loaders == null) {
   loaders = Collections.unmodifiableList(multiModelLoaderFactory.build(modelClass));
   cache.put(modelClass, loaders);
 }
 return loaders;
}
```

我们看看`multiModelLoaderFactory.build(modelClass)`是如何获取所有声明可以处理modelClass的ModelLoader的：

```
@NonNull
synchronized <Model> List<ModelLoader<Model, ?>> build(@NonNull Class<Model> modelClass) {
 try {
   List<ModelLoader<Model, ?>> loaders = new ArrayList<>();
   // 遍历所有注册进来的entry
   for (Entry<?, ?> entry : entries) {
     // Avoid stack overflow recursively creating model loaders by only creating loaders in
     // recursive requests if they haven't been created earlier in the chain. For example:
     // A Uri loader may translate to another model, which in turn may translate back to a Uri.
     // The original Uri loader won't be provided to the intermediate model loader, although
     // other Uri loaders will be.
     if (alreadyUsedEntries.contains(entry)) {
       continue;
     }
     // 注册过的entry有很多，但是entry.modelClass是modelClass（即String.class）的同类或父类的却只有四个
     if (entry.handles(modelClass)) {
       alreadyUsedEntries.add(entry);
       // 对每一个符合条件的entry调用build接口，获取对应的ModelLoader
       loaders.add(this.<Model, Object>build(entry));
       alreadyUsedEntries.remove(entry);
     }
   }
   return loaders;
 } catch (Throwable t) {
   alreadyUsedEntries.clear();
   throw t;
 }
}

@NonNull
@SuppressWarnings("unchecked")
private <Model, Data> ModelLoader<Model, Data> build(@NonNull Entry<?, ?> entry) {
 return (ModelLoader<Model, Data>) Preconditions.checkNotNull(entry.factory.build(this));
}
```

这里的entry的数据结构以及保存的值我们在上面提到过，下面我们看看`entry.factory.build(this)`创建了四个什么样的ModelLoader：

- `append(String.class, InputStream.class, new DataUrlLoader.StreamFactory<String>())`
  该Factory会创建一个处理data scheme(`data:[mediatype][;base64],encoded_data`, e.g. data:image/gif;base64,R0lGO...lBCBMQiB0UjIQA7)类型数据的`DataUrlLoader`
  **声明可能处理以**`data:image`**开头的model**
- `append(String.class, InputStream.class, new StringLoader.StreamFactory())`
  该Factory能够从String中加载InputStream
  **声明可能处理所有类型的model**
- `.append(String.class, ParcelFileDescriptor.class, new StringLoader.FileDescriptorFactory())`
  该Factory能够从String中加载ParcelFileDescriptor
  **声明可能处理所有类型的model**
- `.append(String.class, AssetFileDescriptor.class, new StringLoader.AssetFileDescriptorFactory())`
  该Factory能够从String中加载AssetFileDescriptor
  **声明可能处理所有类型的model**

此处的四个ModelLoader中，`DataUrlLoader.StreamFactory`的逻辑非常清晰明了，其他三个有点麻烦。因为它们都是创建的`StringLoader`，而`StringLoader`内部也有一个`MultiModelLoader`，在Factory中build `StringLoader`时，会调用`multiFactory.build`创建一个内部的`MultiModelLoader`。
因为String可能指向的数据太多了，所以采取`MultiModelLoader`保存所有可能的`ModelLoader`，在处理时会遍历list找出所有可以处理的。

我们看看`StringLoader.StreamFactory()`的build过程：

```
/**
 * Factory for loading {@link InputStream}s from Strings.
 */
public static class StreamFactory implements ModelLoaderFactory<String, InputStream> {

 @NonNull
 @Override
 public ModelLoader<String, InputStream> build(
     @NonNull MultiModelLoaderFactory multiFactory) {
   return new StringLoader<>(multiFactory.build(Uri.class, InputStream.class));
 }

 @Override
 public void teardown() {
   // Do nothing.
 }
}
```

这里调用了`MultiModelLoaderFactory.build(Class, Class)`方法，该方法的重载方法`MultiModelLoaderFactory.build(Class)`我们在上面遇到过，两个方法有一些差别，不要弄混淆了。

```
@NonNull
public synchronized <Model, Data> ModelLoader<Model, Data> build(@NonNull Class<Model> modelClass /* Uri.class */,
   @NonNull Class<Data> dataClass /* InputStream.class */) {
 try {
   List<ModelLoader<Model, Data>> loaders = new ArrayList<>();
   boolean ignoredAnyEntries = false;
   for (Entry<?, ?> entry : entries) {
     // Avoid stack overflow recursively creating model loaders by only creating loaders in
     // recursive requests if they haven't been created earlier in the chain. For example:
     // A Uri loader may translate to another model, which in turn may translate back to a Uri.
     // The original Uri loader won't be provided to the intermediate model loader, although
     // other Uri loaders will be.
     //
     // 防止递归时重复加载到，造成StackOverflow
     if (alreadyUsedEntries.contains(entry)) {
       ignoredAnyEntries = true;
       continue;
     }
     // ⚡⚡️⚡️ 差别1，这里会检查两个class
     if (entry.handles(modelClass, dataClass)) {
       alreadyUsedEntries.add(entry);
       loaders.add(this.<Model, Data>build(entry));
       alreadyUsedEntries.remove(entry);
     }
   }
   // ⚡⚡️⚡️ 差别2，这里会检查loaders的数量，并做相应的处理
   if (loaders.size() > 1) {
     return factory.build(loaders, throwableListPool);
   } else if (loaders.size() == 1) {
     return loaders.get(0);
   } else {
     // Avoid crashing if recursion results in no loaders available. The assertion is supposed to
     // catch completely unhandled types, recursion may mean a subtype isn't handled somewhere
     // down the stack, which is often ok.
     if (ignoredAnyEntries) {
       return emptyModelLoader();
     } else {
       throw new NoModelLoaderAvailableException(modelClass, dataClass);
     }
   }
 } catch (Throwable t) {
   alreadyUsedEntries.clear();
   throw t;
 }
}
```

流程就是，首先，先匹配出所有dataClass为String.class的modelloader；第二，如果是StringLoader的情况，由于StringLoader内部有一个MultiModelLoader的引用（`new StringLoader<>(multiFactory.build(Uri.class, InputStream.class))`),又会继续匹配Uri.class, InputStream.class的ModelLoader并存入到MultiModelLoader变量中，所以最终创建出来的ModelLoader如下图所示：

​	1. 第一个StringLoader的情况

![StringLoader](/Users/lixiangyue/Personal/blog/Blog/源码解析/media/Glide/StringLoader.jpg)

剩下两个StringLoader的情况就不画图了，描述一下第二个StringLoader里包含了AssetUriLoader和UriLoader

第三个StringLoader包含了UriLoader



上面就是`this.<Model, Data>build(entry)`获得到的4个loader，然后在`modelLoaderRegistry.getModelLoaders(model)`方法中被过滤掉第一个，现在就返回3.6.1节刚开始的`DecodeHelper.getLoadData`方法里面了。

在`DecodeHelper.getLoadData`方法中会遍历每个ModelLoader，并调用其`buildLoadData`方法，如果不为空则加入到数组中。由于此处的3个ModelLoader都是`StringLoader`，我们看看`StringLoader.buildLoadData`方法：

```
@Override
public LoadData<Data> buildLoadData(@NonNull String model, int width, int height,
   @NonNull Options options) {
 // 由于我们传入的model是一个网络图片地址，所以uri肯定是正常的
 Uri uri = parseUri(model);
 // 下面判断uriLoader是否有可能处理
 // 如果没有可能，那么返回null
 // 注意，这里传入的已经是Uri.class了
 if (uri == null || !uriLoader.handles(uri)) {
   return null;
 }
 // 否则调用uriLoader的buildLoadData方法
 return uriLoader.buildLoadData(uri, width, height, options);
}

@Nullable
private static Uri parseUri(String model) {
 Uri uri;
 if (TextUtils.isEmpty(model)) {
   return null;
 // See https://pmd.github.io/pmd-6.0.0/pmd_rules_java_performance.html#simplifystartswith
 } else if (model.charAt(0) == '/') {
   uri = toFileUri(model);
 } else {
   uri = Uri.parse(model);
   String scheme = uri.getScheme();
   if (scheme == null) {
     uri = toFileUri(model);
   }
 }
 return uri;
}

private static Uri toFileUri(String path) {
 return Uri.fromFile(new File(path));
}
```

所以重点就在于`StringLoader.uriLoader`了，该参数我们上面分析递归过程的时候分析到了，是一个`MultiModelLoader`对象：

```
class MultiModelLoader<Model, Data> implements ModelLoader<Model, Data> {

 private final List<ModelLoader<Model, Data>> modelLoaders;
 private final Pool<List<Throwable>> exceptionListPool;

 MultiModelLoader(@NonNull List<ModelLoader<Model, Data>> modelLoaders,
     @NonNull Pool<List<Throwable>> exceptionListPool) {
   this.modelLoaders = modelLoaders;
   this.exceptionListPool = exceptionListPool;
 }

 @Override
 public LoadData<Data> buildLoadData(@NonNull Model model, int width, int height,
     @NonNull Options options) {
   Key sourceKey = null;
   int size = modelLoaders.size();
   List<DataFetcher<Data>> fetchers = new ArrayList<>(size);
   //noinspection ForLoopReplaceableByForEach to improve perf
   for (int i = 0; i < size; i++) {
     ModelLoader<Model, Data> modelLoader = modelLoaders.get(i);
     // 找到Url.class的ModelLoader
     if (modelLoader.handles(model)) {
       LoadData<Data> loadData = modelLoader.buildLoadData(model, width, height, options);
       if (loadData != null) {
         sourceKey = loadData.sourceKey;
         fetchers.add(loadData.fetcher);
       }
     }
   }
   return !fetchers.isEmpty() && sourceKey != null
       ? new LoadData<>(sourceKey, new MultiFetcher<>(fetchers, exceptionListPool)) : null;
 }

 @Override
 public boolean handles(@NonNull Model model) {
   for (ModelLoader<Model, Data> modelLoader : modelLoaders) {
     if (modelLoader.handles(model)) {
       return true;
     }
   }
   return false;
 }
}
```

`MultiModelLoader`类的`handles`方法和`buildLoadData`方法都比较清晰明了。

- `handles`方法是只要内部有一个ModelLoader有可能处理，就返回true。
- `buildLoadData`方法会先调用`hanldes`进行第一次筛选，然后在调用ModelLoader的`buildLoadData`方法，如果不为空则保存起来，最后返回`LoadData<>(sourceKey, new MultiFetcher<>(fetchers, exceptionListPool))`对象。

依次遍历handles方法时，只有HttpUriLoader和UriUriLoader是返回true的，同时会调用**buildLoadData**来初始化**LoadData**，而**LoadData**中初始化了真正执行网络请求的**HttpUrlFetcher**

总之最终会得到一个LoadData，LoadData<>(sourceKey, new MultiFetcher<>(fetchers, exceptionListPool))

最终我们得到结果，list中只有一个key：上面的`LoadData`的`sourceKey`，即`GlideUrl`。

那么我们继续回到ResourceCacheGenerator#startNext()中。

接下来要执行的代码是：

```java
// ResourceCacheGenerator.startNext
List<Class<?>> resourceClasses = helper.getRegisteredResourceClasses();

// DecodeHelper
List<Class<?>> getRegisteredResourceClasses() {
 return glideContext.getRegistry()
     .getRegisteredResourceClasses(model.getClass(), resourceClass, transcodeClass);
}
```

继续回去看Registry

```
@NonNull
public <Model, TResource, Transcode> List<Class<?>> getRegisteredResourceClasses(
   @NonNull Class<Model> modelClass,
   @NonNull Class<TResource> resourceClass,
   @NonNull Class<Transcode> transcodeClass) {
 // modelClass = String.class
 // resourceClass 默认为 Object.class
 // transcodeClass = Drawable.class
 //
 // 首先取缓存
 List<Class<?>> result =
     modelToResourceClassCache.get(modelClass, resourceClass, transcodeClass);

 // 有缓存就返回缓存，没有就加载，然后放入缓存
 if (result == null) {
   result = new ArrayList<>();
   // 从modelLoaderRegistry中寻找所有modelClass为String或父类的entry，并返回其dataClass
   // 此处得到的dataClasses为[InputStream.class, ParcelFileDescriptor.class, AssetFileDescriptor.class]  
   // 因为四个注册项中，前两个dataClass都为InputStream.class，去重时肯定会去掉一个
   List<Class<?>> dataClasses = modelLoaderRegistry.getDataClasses(modelClass);
   for (Class<?> dataClass : dataClasses) {
       // 先看看对应的ResourceDecoderRegistry数据结构吧
       List<? extends Class<?>> registeredResourceClasses =
           decoderRegistry.getResourceClasses(dataClass, resourceClass);
       for (Class<?> registeredResourceClass : registeredResourceClasses) {
         List<Class<Transcode>> registeredTranscodeClasses = transcoderRegistry
             .getTranscodeClasses(registeredResourceClass, transcodeClass);
         if (!registeredTranscodeClasses.isEmpty() && !result.contains(registeredResourceClass)) {
           result.add(registeredResourceClass);
         }
       }
     }
   modelToResourceClassCache.put(
       modelClass, resourceClass, transcodeClass, Collections.unmodifiableList(result));
 }

 return result;
}
```

 获取完dataClasses后，下面要对每一个dataClass调用`decoderRegistry.getResourceClasses`方法。

这里涉及到了`ResourceDecoderRegistry`类，同样该类中的数据也是Glide创建时注入的，我们看一下相关的代码。

```
// Glide
.append(Registry.BUCKET_BITMAP, ByteBuffer.class, Bitmap.class, byteBufferBitmapDecoder)
.append(Registry.BUCKET_BITMAP, InputStream.class, Bitmap.class, streamBitmapDecoder)
.append(Registry.BUCKET_BITMAP, ParcelFileDescriptor.class, Bitmap.class,
parcelFileDescriptorVideoDecoder)
.append(Registry.BUCKET_BITMAP, AssetFileDescriptor.class, Bitmap.class, VideoDecoder.asset(bitmapPool))
.append(Registry.BUCKET_BITMAP, Bitmap.class, Bitmap.class, new UnitBitmapDecoder())
.append(Registry.BUCKET_BITMAP_DRAWABLE, ByteBuffer.class, BitmapDrawable.class, new BitmapDrawableDecoder<>(resources, byteBufferBitmapDecoder))
.append(Registry.BUCKET_BITMAP_DRAWABLE, InputStream.class, BitmapDrawable.class, new BitmapDrawableDecoder<>(resources, streamBitmapDecoder))
.append(Registry.BUCKET_BITMAP_DRAWABLE, ParcelFileDescriptor.class, BitmapDrawable.class, new BitmapDrawableDecoder<>(resources, parcelFileDescriptorVideoDecoder))
.append(Registry.BUCKET_GIF, InputStream.class, GifDrawable.class, new StreamGifDecoder(imageHeaderParsers, byteBufferGifDecoder, arrayPool))
.append(Registry.BUCKET_GIF, ByteBuffer.class, GifDrawable.class, byteBufferGifDecoder)
.append(Registry.BUCKET_BITMAP, GifDecoder.class, Bitmap.class, new GifFrameResourceDecoder(bitmapPool))
.append(Uri.class, Drawable.class, resourceDrawableDecoder)
.append(Uri.class, Bitmap.class, new ResourceBitmapDecoder(resourceDrawableDecoder, bitmapPool))
.append(File.class, File.class, new FileDecoder())
.append(Drawable.class, Drawable.class, new UnitDrawableDecoder())

// Registry
public class Registry {
 public static final String BUCKET_GIF = "Gif";
 public static final String BUCKET_BITMAP = "Bitmap";
 public static final String BUCKET_BITMAP_DRAWABLE = "BitmapDrawable";
 private static final String BUCKET_PREPEND_ALL = "legacy_prepend_all";
 private static final String BUCKET_APPEND_ALL = "legacy_append";

 private final ResourceDecoderRegistry decoderRegistry;

 public Registry() {
   this.decoderRegistry = new ResourceDecoderRegistry();
   setResourceDecoderBucketPriorityList(
       Arrays.asList(BUCKET_GIF, BUCKET_BITMAP, BUCKET_BITMAP_DRAWABLE));
 }

 @NonNull
 public final Registry setResourceDecoderBucketPriorityList(@NonNull List<String> buckets) {
   // See #3296 and https://bugs.openjdk.java.net/browse/JDK-6260652.
   List<String> modifiedBuckets = new ArrayList<>(buckets.size());
   modifiedBuckets.addAll(buckets);
   modifiedBuckets.add(0, BUCKET_PREPEND_ALL);
   modifiedBuckets.add(BUCKET_APPEND_ALL);
   decoderRegistry.setBucketPriorityList(modifiedBuckets);
   return this;
 }

 @NonNull
 public <Data, TResource> Registry append(
     @NonNull Class<Data> dataClass,
     @NonNull Class<TResource> resourceClass,
     @NonNull ResourceDecoder<Data, TResource> decoder) {
   append(BUCKET_APPEND_ALL, dataClass, resourceClass, decoder);
   return this;
 }

 @NonNull
 public <Data, TResource> Registry append(
     @NonNull String bucket,
     @NonNull Class<Data> dataClass,
     @NonNull Class<TResource> resourceClass,
     @NonNull ResourceDecoder<Data, TResource> decoder) {
   decoderRegistry.append(bucket, decoder, dataClass, resourceClass);
   return this;
 }

 @NonNull
 public <Data, TResource> Registry prepend(
     @NonNull String bucket,
     @NonNull Class<Data> dataClass,
     @NonNull Class<TResource> resourceClass,
     @NonNull ResourceDecoder<Data, TResource> decoder) {
   decoderRegistry.prepend(bucket, decoder, dataClass, resourceClass);
   return this;
 }
}

// ResourceDecoderRegistry
public class ResourceDecoderRegistry {
 private final List<String> bucketPriorityList = new ArrayList<>();
 private final Map<String, List<Entry<?, ?>>> decoders = new HashMap<>();

 public synchronized void setBucketPriorityList(@NonNull List<String> buckets) {
   List<String> previousBuckets = new ArrayList<>(bucketPriorityList);
   bucketPriorityList.clear();
   bucketPriorityList.addAll(buckets);
   for (String previousBucket : previousBuckets) {
     if (!buckets.contains(previousBucket)) {
       // Keep any buckets from the previous list that aren't included here, but but them at the
       // end.
       bucketPriorityList.add(previousBucket);
     }
   }
 }

 public synchronized <T, R> void append(@NonNull String bucket,
     @NonNull ResourceDecoder<T, R> decoder,
     @NonNull Class<T> dataClass, @NonNull Class<R> resourceClass) {
   getOrAddEntryList(bucket).add(new Entry<>(dataClass, resourceClass, decoder));
 }

 public synchronized <T, R> void prepend(@NonNull String bucket,
     @NonNull ResourceDecoder<T, R> decoder,
     @NonNull Class<T> dataClass, @NonNull Class<R> resourceClass) {
   getOrAddEntryList(bucket).add(0, new Entry<>(dataClass, resourceClass, decoder));
 }

 @NonNull
 private synchronized List<Entry<?, ?>> getOrAddEntryList(@NonNull String bucket) {
   if (!bucketPriorityList.contains(bucket)) {
     // Add this unspecified bucket as a low priority bucket.
     bucketPriorityList.add(bucket);
   }
   List<Entry<?, ?>> entries = decoders.get(bucket);
   if (entries == null) {
     entries = new ArrayList<>();
     decoders.put(bucket, entries);
   }
   return entries;
 }
}
```

`ResourceDecoderRegistry`内部维持着一个具有优先级 bucket 的 list，优先级顺序由BUCKET在`bucketPriorityList`中的顺序决定。
在`Registry`的构造器中，创建`ResourceDecoderRegistry`后，就调用`setResourceDecoderBucketPriorityList`方法调整了其优先级，优先级别为：
BUCKET_PREPEND_ALL, BUCKET_GIF, BUCKET_BITMAP, BUCKET_BITMAP_DRAWABLE, BUCKET_APPEND_ALL。

`ResourceDecoderRegistry`的`append`方法和`prepend`方法就是向对应的桶中将entry插入到尾部或头部。

了解完了`ResourceDecoderRegistry`之后，我们在回到原来的位置继续看看`decoderRegistry.getResourceClasses`方法干了什么：

```
@NonNull
@SuppressWarnings("unchecked")
public synchronized <T, R> List<Class<R>> getResourceClasses(@NonNull Class<T> dataClass,
   @NonNull Class<R> resourceClass) {
 List<Class<R>> result = new ArrayList<>();
 for (String bucket : bucketPriorityList) {
 	// 根据优先级列表顺序，获取对应的entries
   List<Entry<?, ?>> entries = decoders.get(bucket);
   if (entries == null) {
     continue;
   }
   for (Entry<?, ?> entry : entries) {
     if (entry.handles(dataClass, resourceClass)
         && !result.contains((Class<R>) entry.resourceClass)) {
       result.add((Class<R>) entry.resourceClass);
     }
   }
 }
 return result;
}
```

那么，该方法的作用就是根据传入的dataClass以及resourceClass在桶中依次按顺序查找映射关系，如果可以找到就返回这条映射关系的resourceClass。

这里的入参`dataClass in　[InputStream.class, ParcelFileDescriptor.class, AssetFileDescriptor.class]`, `resourceClass = Object.class`。

上面的方法会运行三次，每一次的结果如下：

1. InputStream.class & Object.class
   注册表中所对应的resourceClass 即`GifDrawable.class`、`Bitmap.class`、`BitmapDrawable.class`

2. ParcelFileDescriptor.class & Object.class
   注册表中所对应的resourceClass 即`Bitmap.class`、`BitmapDrawable.class`

3. AssetFileDescriptor.class & Object.class
   注册表中所对应的resourceClass 即`Bitmap.class`

   

   回到`Registry.getRegisteredResourceClasses`方法中，下面将会对每次运行返回的resourceClasses数组进行遍历，并调用了`transcoderRegistry.getTranscodeClasses`方法：

   ```
   // Registry.getRegisteredResourceClasses
   
   for (Class<?> dataClass : dataClasses) {
    // 我们刚才分析了这个方法，返回值看上面的分析
    List<? extends Class<?>> registeredResourceClasses =
        decoderRegistry.getResourceClasses(dataClass, resourceClass);
        // registeredResourceClasses中的数据为[GifDrawable.class、Bitmap.class、BitmapDrawable.class]
    for (Class<?> registeredResourceClass : registeredResourceClasses) {
      // 现在分析这个方法
      List<Class<Transcode>> registeredTranscodeClasses = transcoderRegistry
          .getTranscodeClasses(registeredResourceClass, transcodeClass);
      if (!registeredTranscodeClasses.isEmpty() && !result.contains(registeredResourceClass)) {
        result.add(registeredResourceClass);
      }
    }
   }
   ```

现在分析一下`transcoderRegistry.getTranscodeClasses`方法。

和上面分析过的两个Registry一样，`TranscoderRegistry`也同样是在Glide构建的时候注册进来的，相关代码如下：

```
// Glide
.register(Bitmap.class, BitmapDrawable.class, new BitmapDrawableTranscoder(resources))
.register(Bitmap.class, byte[].class, bitmapBytesTranscoder)
.register(Drawable.class, byte[].class, new DrawableBytesTranscoder(bitmapPool, bitmapBytesTranscoder, gifDrawableBytesTranscoder))
.register(GifDrawable.class, byte[].class, gifDrawableBytesTranscoder);

// Registry
public class Registry {
 private final TranscoderRegistry transcoderRegistry;

 public Registry() {
   this.transcoderRegistry = new TranscoderRegistry();
 }

 @NonNull
 public <TResource, Transcode> Registry register(
     @NonNull Class<TResource> resourceClass, @NonNull Class<Transcode> transcodeClass,
     @NonNull ResourceTranscoder<TResource, Transcode> transcoder) {
   transcoderRegistry.register(resourceClass, transcodeClass, transcoder);
   return this;
 }
}

public class TranscoderRegistry {
 private final List<Entry<?, ?>> transcoders = new ArrayList<>();

 public synchronized <Z, R> void register(
     @NonNull Class<Z> decodedClass, @NonNull Class<R> transcodedClass,
     @NonNull ResourceTranscoder<Z, R> transcoder) {
   transcoders.add(new Entry<>(decodedClass, transcodedClass, transcoder));
 }
}
```

注册代码很简单，就是保存到list中就完事了。看看`TranscoderRegistry.getTranscodeClasses`方法：

```
@NonNull
public synchronized <Z, R> List<Class<R>> getTranscodeClasses(
   @NonNull Class<Z> resourceClass, @NonNull Class<R> transcodeClass) {
 List<Class<R>> transcodeClasses = new ArrayList<>();
 // GifDrawable -> Drawable is just the UnitTranscoder, as is GifDrawable -> GifDrawable.
 // 🔥路径1
 if (transcodeClass.isAssignableFrom(resourceClass)) {
   transcodeClasses.add(transcodeClass);
   return transcodeClasses;
 }

 // 🔥路径2
 for (Entry<?, ?> entry : transcoders) {
   if (entry.handles(resourceClass, transcodeClass)) {
     transcodeClasses.add(transcodeClass);
   }
 }

 // list添加的都是入参transcodeClass
 return transcodeClasses;
}
```

此方法的作用是根据resourceClass和transcodeClass，从自身或者注册表的“map”中找出`transcodeClass`。

回到`Registry.getRegisteredResourceClasses`方法中的第二层for loop继续分析，下面就是对每个resourceClass得到的registeredTranscodeClasses（transcodeClass为`Drawable.class`）

1. InputStream.class & Object.class
   resourceClass为
2. `GifDrawable.class`
   路径1 允许添加resourceClass
3. `Bitmap.class`
   路径2 走Glide注入代码片段的第2行 允许添加resourceClass
4. `BitmapDrawable.class`
   路径1 允许添加resourceClass
5. ParcelFileDescriptor.class & Object.class
   resourceClass为
6. `Bitmap.class`
   路径2 走Glide注入代码片段的第2行 但resourceClass已经添加过
7. `BitmapDrawable.class`
   路径1 但resourceClass已经添加过
8. AssetFileDescriptor.class & Object.class
   resourceClass为
9. `Bitmap.class`
   路径2 走Glide注入代码片段的第2行 但resourceClass已经添加过

因此，`Registry.getRegisteredResourceClasses`返回了`[GifDrawable.class、Bitmap.class、BitmapDrawable.class]`数组，该数组会经过`DecodeHelper`返回到`ResourceCacheGenerator.startNext`方法的调用中。



继续回到本小节的第一个方法`ResourceCacheGenerator.startNext`方法中。下面要执行的代码是一个while循环，循环结束的标志是找到了缓存文件：

```
// 遍历sourceIds中的每一个key、resourceClasses中每一个class，以及其他的一些值组成key
// 尝试在磁盘缓存中以key找到缓存文件
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
 cacheFile = helper.getDiskCache().get(currentKey);
 // 如果找到了缓存文件，那么循环条件则会为false，也就退出循环了
 if (cacheFile != null) {
   sourceKey = sourceId;
   modelLoaders = helper.getModelLoaders(cacheFile);
   modelLoaderIndex = 0;
 }
}
```

走到这里，由于是初次加载，所以DiskLruCache里面肯定是没有缓存的。

注意这里的Key的组成，在之前我们描述过`ResourceCacheGenerator`的作用：获取采样后、transformed后资源文件的缓存文件。在第3节中我们可以看到，DownsampleStrategy和Transformation保存在了`BaseRequestOptions`里面，前者保存在`BaseRequestOptions.Options`里，后者保存在`transformations`里。在这里这两个参数都作为了缓存文件的Key，这也侧面验证了`ResourceCacheGenerator`的作用。

但在加载代码上加上话`.diskCacheStrategy(DiskCacheStrategy.RESOURCE)`，就可以到了缓存，这样可以接着一次性把后面的方法也分析完，先看看下面的这个方法：

```
modelLoaders = helper.getModelLoaders(cacheFile);
```

内部调用了`Registry.getModelLoaders`方法：

```
List<ModelLoader<File, ?>> getModelLoaders(File file)
   throws Registry.NoModelLoaderAvailableException {
 return glideContext.getRegistry().getModelLoaders(file);
}
```

该方法我们上面具体分析过，稍加回忆后我们可以写出`getModelLoaders(File)`方法的过程：

1. 首先找出注入时以`File.class`为modelClass的注入代码
2. 调用所有注入的`factory.build`方法得到`ModelLoader`
3. 过滤掉不可能处理`model`的`ModelLoader`

这样得到了以下四个ModelLoader

- `.append(File.class, ByteBuffer.class, new ByteBufferFileLoader.Factory())`
- `ByteBufferFileLoader`
- `.append(File.class, InputStream.class, new FileLoader.StreamFactory())`
- `FileLoader`
- `.append(File.class, ParcelFileDescriptor.class, new FileLoader.FileDescriptorFactory())`
- `FileLoader`
- `.append(File.class, File.class, UnitModelLoader.Factory.<File>getInstance())`
- `UnitModelLoader`

所以此时的modelLoaders值为`[ByteBufferFileLoader, FileLoader, FileLoader, UnitModelLoader]`。

接下来会调用每一个ModelLoader尝试加载数据，直到找到第一个可以处理的ModelLoader：

```
loadData = null;
boolean started = false;
while (!started && hasNextModelLoader()) {
 ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
 loadData = modelLoader.buildLoadData(cacheFile,
     helper.getWidth(), helper.getHeight(), helper.getOptions());
 if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
   started = true;
   loadData.fetcher.loadData(helper.getPriority(), this);
 }
}

return started;
```

这四个ModelLoader都会调用`buildLoadData`方法创建`LoadData`对象，该对象重要的成员变量是`DataFetcher`；然后调用`helper.hasLoadPath`根据`resourceClass`参数和`transcodeClass`参数判断是否有路径达到`DataFetcher.getDataClass`，如果有那就调用此fetcher进行loadData，任务执行完毕。

例子中符合条件的`ModelLoader`以及其`fetcher`如下：

- `ByteBufferFileLoader`
  fetcher = `ByteBufferFetcher(file)`
  fetcher.dataClass = `ByteBuffer.class`
- `FileLoader`
  fetcher = `FileFetcher(model, fileOpener)`
  fetcher.dataClass = `InputStream.class`
- `FileLoader`
  fetcher = `FileFetcher(model, fileOpener)`
  fetcher.dataClass = `ParcelFileDescriptor.class`
- `UnitModelLoader`
  fetcher = `UnitFetcher<>(model)`
  fetcher.dataClass = `File.class`

看一下`DecodeHelper.getLoadPath`方法是如何判断路径的：

```
// DecodeHelper
<Data> LoadPath<Data, ?, Transcode> getLoadPath(Class<Data> dataClass) {
 return glideContext.getRegistry().getLoadPath(dataClass, resourceClass, transcodeClass);
}

// Registry
@Nullable
public <Data, TResource, Transcode> LoadPath<Data, TResource, Transcode> getLoadPath(
   @NonNull Class<Data> dataClass, @NonNull Class<TResource> resourceClass,
   @NonNull Class<Transcode> transcodeClass) {
 // 先取缓存
 LoadPath<Data, TResource, Transcode> result =
     loadPathCache.get(dataClass, resourceClass, transcodeClass);
 // 如果取到NO_PATHS_SIGNAL这条LoadPath，那么返回null
 if (loadPathCache.isEmptyLoadPath(result)) {
   return null;
 } else if (result == null) {
   // 取到null，说明还没有获取过
   // 那么先获取decodePaths，在创建LoadPath对象并存入缓存中
   List<DecodePath<Data, TResource, Transcode>> decodePaths =
       getDecodePaths(dataClass, resourceClass, transcodeClass);
   // It's possible there is no way to decode or transcode to the desired types from a given
   // data class.
   if (decodePaths.isEmpty()) {
     result = null;
   } else {
     result =
         new LoadPath<>(
             dataClass, resourceClass, transcodeClass, decodePaths, throwableListPool);
   }
   // 存入缓存
   loadPathCache.put(dataClass, resourceClass, transcodeClass, result);
 }
 return result;
}
```

可以看出，`getLoadPath`的关键就是`getDecodePaths`方法：

```
@NonNull
private <Data, TResource, Transcode> List<DecodePath<Data, TResource, Transcode>> getDecodePaths(
   @NonNull Class<Data> dataClass, @NonNull Class<TResource> resourceClass,
   @NonNull Class<Transcode> transcodeClass) {
 List<DecodePath<Data, TResource, Transcode>> decodePaths = new ArrayList<>();
 // 1
 List<Class<TResource>> registeredResourceClasses =
     decoderRegistry.getResourceClasses(dataClass, resourceClass);

 for (Class<TResource> registeredResourceClass : registeredResourceClasses) {
   // 2
   List<Class<Transcode>> registeredTranscodeClasses =
       transcoderRegistry.getTranscodeClasses(registeredResourceClass, transcodeClass);

   for (Class<Transcode> registeredTranscodeClass : registeredTranscodeClasses) {
     // 3
     List<ResourceDecoder<Data, TResource>> decoders =
         decoderRegistry.getDecoders(dataClass, registeredResourceClass);
     ResourceTranscoder<TResource, Transcode> transcoder =
         transcoderRegistry.get(registeredResourceClass, registeredTranscodeClass);
     // 4
     @SuppressWarnings("PMD.AvoidInstantiatingObjectsInLoops")
     DecodePath<Data, TResource, Transcode> path =
         new DecodePath<>(dataClass, registeredResourceClass, registeredTranscodeClass,
             decoders, transcoder, throwableListPool);
     decodePaths.add(path);
   }
 }
 return decodePaths;
}
```

首先就是`decoderRegistry.getResourceClasses(dataClass, resourceClass)`方法，该方法我们上面分析过，作用就是根据传入的dataClass以及resourceClass在Registry中找映射关系，如果可以找到就返回这条映射关系的resourceClass（该方法中dataClass为传入的fetcher.dataClass，resourceClass值为Object.class）。

然后对于每个获取到的registeredResourceClass，调用`transcoderRegistry.getTranscodeClasses`方法，此方法之前也解析过，其作用是根据resourceClass和transcodeClass，从自身或者注册表的“map”中找出transcodeClass（参数transcodeClass为`Drawable.class`）。

最后，对于每个registeredResourceClass和registeredTranscodeClass，都会获取其`ResourceDecoder`和`ResourceTranscoder`，并将这些参数组成一个`DecodePath`保存到list，最后返回。我们先看一下这些方法的实现，最后在给出每一步的操作结果。

所以我们直接看`decoderRegistry.getDecoders`方法：

```
@NonNull
@SuppressWarnings("unchecked")
public synchronized <T, R> List<ResourceDecoder<T, R>> getDecoders(@NonNull Class<T> dataClass,
   @NonNull Class<R> resourceClass) {
 List<ResourceDecoder<T, R>> result = new ArrayList<>();
 for (String bucket : bucketPriorityList) {
   List<Entry<?, ?>> entries = decoders.get(bucket);
   if (entries == null) {
     continue;
   }
   for (Entry<?, ?> entry : entries) {
     if (entry.handles(dataClass, resourceClass)) {
       result.add((ResourceDecoder<T, R>) entry.decoder);
     }
   }
 }
 // TODO: cache result list.

 return result;
}
```

该方法也非常简单，那就是遍历所有的bucket中的所有entry，找出所有能处理dataClass、resourceClass的entry，保存其decoder。

最后看一下`transcoderRegistry.get`方法：

```
@NonNull
@SuppressWarnings("unchecked")
public synchronized <Z, R> ResourceTranscoder<Z, R> get(
   @NonNull Class<Z> resourceClass, @NonNull Class<R> transcodedClass) {
 // For example, there may be a transcoder that can convert a GifDrawable to a Drawable, which
 // will be caught above. However, if there is no registered transcoder, we can still just use
 // the UnitTranscoder to return the Drawable because the transcode class (Drawable) is
 // assignable from the resource class (GifDrawable).
 if (transcodedClass.isAssignableFrom(resourceClass)) {
   return (ResourceTranscoder<Z, R>) UnitTranscoder.get();
 }
 for (Entry<?, ?> entry : transcoders) {
   if (entry.handles(resourceClass, transcodedClass)) {
     return (ResourceTranscoder<Z, R>) entry.transcoder;
   }
 }

 throw new IllegalArgumentException(
     "No transcoder registered to transcode from " + resourceClass + " to " + transcodedClass);
}
```

该方法逻辑和我们之前谈到过的`transcoderRegistry.getTranscodeClasses`方法类似，只不过返回的是对应的`transcoder`对象。

了解到上面这些方法的作用后，我们列出`Registry.getDecodePaths`方法执行的步骤以及结果（dataClass为下面的fetcher.dataClass，resourceClass为Object.class，transcodeClass为Drawable.class）：

- `ByteBufferFileLoader`
  fetcher = `ByteBufferFetcher(file)`
  fetcher.dataClass = `ByteBuffer.class`
  1 registeredResourceClasses = `[GifDrawable.class, Bitmap.class, BitmapDrawable.class]`
  2 registeredTranscodeClasses = `[[Drawable.class], [Drawable.class], [Drawable.class]]`
  3 decoders = `[[ByteBufferGifDecoder], [ByteBufferBitmapDecoder], [BitmapDrawableDecoder]]`
  4 transcoder = `[UnitTranscoder, BitmapDrawableTranscoder, UnitTranscoder]`
  5 decodePaths = `[DecodePath(ByteBuffer.class, GifDrawable.class, Drawable.class, [ByteBufferGifDecoder], UnitTranscoder), DecodePath(ByteBuffer.class, Bitmap.class, Drawable.class, [ByteBufferBitmapDecoder], BitmapDrawableTranscoder), DecodePath(ByteBuffer.class, BitmapDrawable.class, Drawable.class, [BitmapDrawableDecoder], UnitTranscoder)]`
  6 loadPath = `LoadPath(ByteBuffer.class, Object.class, Drawable.class, decodePaths)`

在`ByteBufferFileLoader`中，我们已经找到一个一条可以加载的路径，那么就调用此`fetcher.loadData`方法进行加载。同时，该方法`ResourceCacheGenerator.startNext`返回true，这就意味着`DecodeJob`无需在尝试另外的`DataFetcherGenerator`进行加载，整个`into`过程已经大致完成，剩下的就是等待资源加载完毕后触发回调即可。

下面我们接着看看`loadData.fetcher.loadData(helper.getPriority(), this)`这条语句干了什么，在上面的分析中我们知道，这里的fetcher是`ByteBufferFetcher`对象，其loadData方法如下：

```
@Override
public void loadData(@NonNull Priority priority,
   @NonNull DataCallback<? super ByteBuffer> callback) {
 ByteBuffer result;
 try {
   // 这里的file就是缓存下来的source file
   // 路径在demo中为 /data/data/yorek.demoandtest/cache/image_manager_disk_cache/65a6e0855da59221f073aba07dc6c69206834ef83f60c58062bee458fcac7dde.0
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

`ByteBufferUtil.fromFile`使用了`RandomAccessFile`和`FileChannel`进行文件操作。如果操作失败，调用`callback.onLoadFailed(e)`通知`ResourceCacheGenerator`类，该类会将操作转发给`DecodeJob`；`callback.onDataReady`操作类似。这样程序就回到了`DecodeJob`回调方法中了。

我们暂时不继续分析`DecodeJob`的回调方法，因为在本节中缓存文件本来是没有的，所以会交给下一个`DataFetcherGenerator`进行尝试处理，所以后面肯定也会遇到`DecodeJob`的回调方法。



总结一下这里，ResourceCacheGenerator#startNext()方法：

* 第一句`List<Key> sourceIds = helper.getCacheKeys();`里面会根据model的class（这里传入的值是String.class）从注册表中找到相关的ModelLoaders，注册表中的字段如下，跟String.class对应的Loader就是[DataUrlLoader.StreamFactory，StringLoader.StreamFactory，StringLoader.FileDescriptorFactory，StringLoader.AssetFileDescriptorFactory]：

  ```java
  // model可能是一个base64格式的img，经过处理后可以变成InputStream类型(dataClass)的数据
  .append(String.class, InputStream.class, new DataUrlLoader.StreamFactory<String>())
  // model可能是一个uri，这种情况下可能性非常多，因为网络图片、assets图片、磁盘图片等等都是一个uri
  .append(String.class, InputStream.class, new StringLoader.StreamFactory())
  // model可能是一个本地图片、assets图片，所以可以处理成ParcelFileDescriptor.class类型的数据
  .append(String.class, ParcelFileDescriptor.class, new StringLoader.FileDescriptorFactory())
  // model可能是一个本地图片，所以可以处理成为AssetFileDescriptor.class类型的数据
  .append(
     String.class, AssetFileDescriptor.class, new StringLoader.AssetFileDescriptorFactory())
  ```

  根据对应的Factory又会初始化出一个ModelLoaders的列表（其中会过滤掉值DataUrlLoader）是[StringLoader，StringLoader，StringLoader]，但是每一个StringLoader里又有一个成员变量MultiModelLoader来保存从String.class对应的多种可能的ModelLoader，具体如下：

  1. 第一个StringLoader的情况

  ![StringLoader](/Users/lixiangyue/Personal/blog/Blog/源码解析/media/Glide/StringLoader.jpg)

  剩下两个StringLoader的情况就不画图了，描述一下第二个StringLoader里包含了AssetUriLoader和UriLoader

  第三个StringLoader包含了UriLoader

  之后会依次遍历handles方法，根据是否能处理Uri.class，筛选出只有HttpUriLoader和UriUriLoader是返回true的，同时会调用**buildLoadData**来初始化**LoadData**，而**LoadData**中初始化了真正执行网络请求的**HttpUrlFetcher**

  总之最终会得到一个LoadData，LoadData<>(sourceKey, new MultiFetcher<>(fetchers, exceptionListPool))

  最终我们得到结果，list中只有一个key：上面的`LoadData`的`sourceKey`，即`GlideUrl`。



第二步是`List<Class<?>> resourceClasses = helper.getRegisteredResourceClasses();`，来根据model.class，resourceClass，transcodeClass来获取（这里实际的值分别对应的String.class，Object.class，Drawable.class），首先会调用`List<Class<?>> dataClasses = modelLoaderRegistry.getDataClasses(modelClass);`来拿到所有的dataClass，最终得到的是[InputStream.class, ParcelFileDescriptor.class, AssetFileDescriptor.class];

之后会遍历这个dataClass的列表，再根据每一个dataClass和resourceClass，拿到所有的resourceClass，当前注册表中的情况为：

```java
.append(Registry.BUCKET_BITMAP, ByteBuffer.class, Bitmap.class, byteBufferBitmapDecoder)
.append(Registry.BUCKET_BITMAP, InputStream.class, Bitmap.class, streamBitmapDecoder)
.append(Registry.BUCKET_BITMAP, ParcelFileDescriptor.class, Bitmap.class,
parcelFileDescriptorVideoDecoder)
.append(Registry.BUCKET_BITMAP, AssetFileDescriptor.class, Bitmap.class, VideoDecoder.asset(bitmapPool))
.append(Registry.BUCKET_BITMAP, Bitmap.class, Bitmap.class, new UnitBitmapDecoder())
.append(Registry.BUCKET_BITMAP_DRAWABLE, ByteBuffer.class, BitmapDrawable.class, new BitmapDrawableDecoder<>(resources, byteBufferBitmapDecoder))
.append(Registry.BUCKET_BITMAP_DRAWABLE, InputStream.class, BitmapDrawable.class, new BitmapDrawableDecoder<>(resources, streamBitmapDecoder))
.append(Registry.BUCKET_BITMAP_DRAWABLE, ParcelFileDescriptor.class, BitmapDrawable.class, new BitmapDrawableDecoder<>(resources, parcelFileDescriptorVideoDecoder))
.append(Registry.BUCKET_GIF, InputStream.class, GifDrawable.class, new StreamGifDecoder(imageHeaderParsers, byteBufferGifDecoder, arrayPool))
.append(Registry.BUCKET_GIF, ByteBuffer.class, GifDrawable.class, byteBufferGifDecoder)
.append(Registry.BUCKET_BITMAP, GifDecoder.class, Bitmap.class, new GifFrameResourceDecoder(bitmapPool))
.append(Uri.class, Drawable.class, resourceDrawableDecoder)
.append(Uri.class, Bitmap.class, new ResourceBitmapDecoder(resourceDrawableDecoder, bitmapPool))
.append(File.class, File.class, new FileDecoder())
.append(Drawable.class, Drawable.class, new UnitDrawableDecoder())
```

之后会将所有的拿到的resourceClass放到registeredResourceClasses这个list中，当前registeredResourceClasses中的值则为[GifDrawable.class，Bitmap.class，BitmapDrawable.class]

接下来会继续遍历拿到的registeredResourceClasses的list，通过registeredResourceClass和transcodeClass来拿到transcodeClass，并将其加入到registeredTranscodeClasses的列表，最终会将能够使用的resourceClass放到result这个list中并返回。



第三步遍历resourceClass和sourceId，来生成一个currentKey，利用这个currentKey来从缓存中获取获取file文件，然后根据这个文件，再次从注册表中，构建出可以加载数据ModelLoader的列表，存入到modelLoaders中，此时拿到的值为[ByteBufferFileLoader, FileLoader, FileLoader, UnitModelLoader]



第四步，挨个遍历modelLoaders，从ModelLoader中构建出LoadData（LoadData中包含了一个Fetcher对象），并判断是否有符合的能到达的路径，如果有的话，就构建出可达的路径，封装了一堆decoder和一个transcoder。

第五步，拿到对应的路径后，使用对应的Fetcher执行真正的加载数据的逻辑，如果成功就调用onDataReady回调，失败则调用onLoadFailed回调，继续执行下一个Generator。



由于`Glide-with-load-into`三步没有在`ResourceCacheGenerator`中被fetch，所以回到`DecodeJob.runGenerators`方法中，继续执行while循环：

```
private void runGenerators() {
  currentThread = Thread.currentThread();
  startFetchTime = LogTime.getLogTime();
  boolean isStarted = false;
  while (!isCancelled && currentGenerator != null
      && !(isStarted = currentGenerator.startNext())) {
    stage = getNextStage(stage);
    currentGenerator = getNextGenerator();

    if (stage == Stage.SOURCE) {
      reschedule();
      return;
    }
  }
  // We've run out of stages and generators, give up.
  if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
    notifyFailed();
  }

  // Otherwise a generator started a new load and we expect to be called back in
  // onDataFetcherReady.
}
```

由于`diskCacheStrategy`默认为`DiskCacheStrategy.AUTOMATIC`，其`decodeCachedData()`返回true，所以`getNextStage(stage)`是`Stage.DATA_CACHE`。因此`getNextGenerator()`方法返回了`DataCacheGenerator(decodeHelper, this)`。然后在while循环中会执行其`startNext()`方法。



`ResourceCacheGenerator`在构造的时候就将`helper.getCacheKeys()`保存了起来，我们前面在谈`ResourceCacheGenerator`的时候提到过，`helper.getCacheKeys()`采取了防止重复加载的策略。

构造器相关代码如下：

```
private final List<Key> cacheKeys;
private final DecodeHelper<?> helper;
private final FetcherReadyCallback cb;

DataCacheGenerator(DecodeHelper<?> helper, FetcherReadyCallback cb) {
  this(helper.getCacheKeys(), helper, cb);
}

DataCacheGenerator(List<Key> cacheKeys, DecodeHelper<?> helper, FetcherReadyCallback cb) {
  this.cacheKeys = cacheKeys;
  this.helper = helper;
  this.cb = cb;
}
```

然后看一下它的`startNext()`方法，该方法和`ResourceCacheGenerator.startNext`方法非常相似，由于获取的是原始的源数据，所以这里的key的组成非常简单。

```
@Override
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

private boolean hasNextModelLoader() {
  return modelLoaderIndex < modelLoaders.size();
}
```

由于我们第一次加载，本地缓存文件肯定是没有的。我们接着看最后一个`SourceGenerator`，看看它是如何获取数据的。



在这之前我们需要注意，如果Glide在加载时指定了`.onlyRetrieveFromCache(true)`，那么在`DecodeJob.getNextStage(Stage)`方法中就会跳过`Stage.SOURCE`直接到达`Stage.FINISHED`。
且当为`Stage.SOURCE`时，`DecodeJob.runGenerators()`方法会调用`reschedule()`方法，这将会导致`DecodeJob`重新被提交到`sourceExecutor`这个线程池中，同时runReason被赋值为`RunReason.SWITCH_TO_SOURCE_SERVICE`。该线程池默认实现为`GlideExecutor.newSourceExecutor()`:

```
private static final int MAXIMUM_AUTOMATIC_THREAD_COUNT = 4;
private static final String DEFAULT_SOURCE_EXECUTOR_NAME = "source";

public static GlideExecutor newSourceExecutor() {
  return newSourceExecutor(
      calculateBestThreadCount(),
      DEFAULT_SOURCE_EXECUTOR_NAME,
      UncaughtThrowableStrategy.DEFAULT);
}

public static int calculateBestThreadCount() {
  if (bestThreadCount == 0) {
    bestThreadCount =
        Math.min(MAXIMUM_AUTOMATIC_THREAD_COUNT, RuntimeCompat.availableProcessors());
  }
  return bestThreadCount;
}
```

由于`DecodeJob`实现了`Runnable`接口，那么直接看`run()`方法里面的真正实现`runWrapped()`方法：

```
private void runWrapped() {
  switch (runReason) {
    ...
    case SWITCH_TO_SOURCE_SERVICE:
      runGenerators();
      break;
  }
}
```

这里还是执行了`runGenerators()`方法。该方法我们已经很熟悉了，在这里会执行`SourceGenerator.startNext()`方法。

```
private int loadDataListIndex;

@Override
public boolean startNext() {
  // 首次运行dataToCache为null
  if (dataToCache != null) {
    Object data = dataToCache;
    dataToCache = null;
    cacheData(data);
  }

  // 首次运行sourceCacheGenerator为null
  if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
    return true;
  }
  sourceCacheGenerator = null;

  // 准备加载数据
  loadData = null;
  boolean started = false;
  // 这里直接调用了DecodeHelper.getLoadData()方法
  // 该方法在前面在ResourceCacheGenerator中被调用过，且被缓存了下来
  while (!started && hasNextModelLoader()) {
    loadData = helper.getLoadData().get(loadDataListIndex++);
    if (loadData != null
        && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
        || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
      started = true;
      loadData.fetcher.loadData(helper.getPriority(), this);
    }
  }
  return started;
}

private boolean hasNextModelLoader() {
  return loadDataListIndex < helper.getLoadData().size();
}
```

`helper.getLoadData()`的值在`ResourceCacheGenerator`中就已经被获取并缓存下来了，这是一个`MultiModelLoader`对象生成的`LoadData`对象，`LoadData`对象里面有两个fetcher。详见[第3.6.1节的末尾部分](https://blog.yorek.xyz/android/3rd-library/glide2/#361-helpergetcachekeys)

在上面的方法中，我们会遍历LoadData list，找出符合条件的LoadData，然后调用`loadData.fetcher.loadData`加载数据。
在loadData不为空的前提下，会判断Glide的缓存策略是否可以缓存此数据源，或者是否有加载路径。

我们知道，默认情况下Glide的缓存策略是`DiskCacheStrategy.AUTOMATIC`，其`isDataCacheable`实现如下：

```
@Override
public boolean isDataCacheable(DataSource dataSource) {
  return dataSource == DataSource.REMOTE;
}
```

所以，我们看一下`loadData.fetcher.getDataSource()`返回了什么：

```
static class MultiFetcher<Data> implements DataFetcher<Data>, DataCallback<Data> {
  @NonNull
  @Override
  public DataSource getDataSource() {
    return fetchers.get(0).getDataSource();
  }
}

// MultiFetcher中fetchers数组保存的两个DataFetcher都是HttpUrlFetcher
public class HttpUrlFetcher implements DataFetcher<InputStream> {
  @NonNull
  @Override
  public DataSource getDataSource() {
  return DataSource.REMOTE;
  }
}
```

显然，Glide的缓存策略是可以缓存此数据源的。所以会进行数据的加载。接着看看`MultiFetcher.loadData`方法。
这里首先会调用内部的第0个DataFetcher进行加载，同时设置回调为自己。当这一个DataFetcher加载失败时，会尝试调用下一个DataFetcher进行加载，如果没有所有的DataFetcher都加载失败了，就把错误抛给上一层；当有DataFetcher加载成功时，也会把获取到的数据转交给上一层。

```
static class MultiFetcher<Data> implements DataFetcher<Data>, DataCallback<Data> {

  private final List<DataFetcher<Data>> fetchers;
  private int currentIndex;
  private Priority priority;
  private DataCallback<? super Data> callback;

  @Override
  public void loadData(
      @NonNull Priority priority, @NonNull DataCallback<? super Data> callback) {
    this.priority = priority;
    this.callback = callback;
    exceptions = throwableListPool.acquire();
    fetchers.get(currentIndex).loadData(priority, this);

    // If a race occurred where we cancelled the fetcher in cancel() and then called loadData here
    // immediately after, make sure that we cancel the newly started fetcher. We don't bother
    // checking cancelled before loadData because it's not required for correctness and would
    // require an unlikely race to be useful.
    if (isCancelled) {
      cancel();
    }
  }

  @Override
  public void onDataReady(@Nullable Data data) {
    if (data != null) {
      callback.onDataReady(data);
    } else {
      startNextOrFail();
    }
  }

  @Override
  public void onLoadFailed(@NonNull Exception e) {
    Preconditions.checkNotNull(exceptions).add(e);
    startNextOrFail();
  }

  private void startNextOrFail() {
    if (isCancelled) {
      return;
    }

    if (currentIndex < fetchers.size() - 1) {
      currentIndex++;
      loadData(priority, callback);
    } else {
      Preconditions.checkNotNull(exceptions);
      callback.onLoadFailed(new GlideException("Fetch failed", new ArrayList<>(exceptions)));
    }
  }
}
```

这里面两个DataFetcher都是参数相同的`HttpUrlFetcher`实例，我们直接看里面如何从网络加载图片的。

```
@Override
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

很显然，这里将请求操作放到了`loadDataWithRedirects`方法中，然后将请求结果通过回调返回上一层也就是`MultiFetcher`中。

`loadDataWithRedirects`第二个参数表示重定向的次数，在方法内部限制了重定向发生的次数不能超过`MAXIMUM_REDIRECTS=5`次。
第三个参数是发生重定向前的原始url，用来与当前url判断，是不是重定向到自身了。而且可以看出，Glide加载网络图片使用的是`HttpUrlConnection`。
第四个参数headers默认为`Headers.DEFAULT`，就是一个User-Agent的key-value对。

代码如下：

```
private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl,
    Map<String, String> headers) throws IOException {
  // 检查重定向次数
  if (redirects >= MAXIMUM_REDIRECTS) {
    throw new HttpException("Too many (> " + MAXIMUM_REDIRECTS + ") redirects!");
  } else {
    // Comparing the URLs using .equals performs additional network I/O and is generally broken.
    // See http://michaelscharf.blogspot.com/2006/11/javaneturlequals-and-hashcode-make.html.
    try {
      // 检查是不是重定向到自身了
      if (lastUrl != null && url.toURI().equals(lastUrl.toURI())) {
        throw new HttpException("In re-direct loop");

      }
    } catch (URISyntaxException e) {
      // Do nothing, this is best effort.
    }
  }

  // connectionFactory默认是DefaultHttpUrlConnectionFactory
  // 其build方法就是调用了url.openConnection()
  urlConnection = connectionFactory.build(url);
  for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
    urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
  }
  urlConnection.setConnectTimeout(timeout);
  urlConnection.setReadTimeout(timeout);
  urlConnection.setUseCaches(false);
  urlConnection.setDoInput(true);

  // Stop the urlConnection instance of HttpUrlConnection from following redirects so that
  // redirects will be handled by recursive calls to this method, loadDataWithRedirects.
  // 禁止HttpUrlConnection自动重定向，重定向功能由本方法自己实现
  urlConnection.setInstanceFollowRedirects(false);

  // Connect explicitly to avoid errors in decoders if connection fails.
  urlConnection.connect();
  // Set the stream so that it's closed in cleanup to avoid resource leaks. See #2352.
  stream = urlConnection.getInputStream();
  if (isCancelled) {
    return null;
  }
  final int statusCode = urlConnection.getResponseCode();
  if (isHttpOk(statusCode)) {
    // statusCode=2xx，请求成功
    return getStreamForSuccessfulRequest(urlConnection);
  } else if (isHttpRedirect(statusCode)) {
    // statusCode=3xx，需要重定向
    String redirectUrlString = urlConnection.getHeaderField("Location");
    if (TextUtils.isEmpty(redirectUrlString)) {
      throw new HttpException("Received empty or null redirect url");
    }
    URL redirectUrl = new URL(url, redirectUrlString);
    // Closing the stream specifically is required to avoid leaking ResponseBodys in addition
    // to disconnecting the url connection below. See #2352.
    cleanup();
    return loadDataWithRedirects(redirectUrl, redirects + 1, url, headers);
  } else if (statusCode == INVALID_STATUS_CODE) {
    // -1 表示不是HTTP响应
    throw new HttpException(statusCode);
  } else {
    // 其他HTTP错误
    throw new HttpException(urlConnection.getResponseMessage(), statusCode);
  }
}

// Referencing constants is less clear than a simple static method.
private static boolean isHttpOk(int statusCode) {
  return statusCode / 100 == 2;
}

// Referencing constants is less clear than a simple static method.
private static boolean isHttpRedirect(int statusCode) {
  return statusCode / 100 == 3;
}
```

现在我们已经获得网络图片的InputStream了，该资源会通过回调经过`MultiFetcher`到达`SourceGenerator`中。

下面是`DataCallback`回调在`SourceGenerator`中的实现。

```
@Override
public void onDataReady(Object data) {
  DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
  if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
    dataToCache = data;
    // We might be being called back on someone else's thread. Before doing anything, we should
    // reschedule to get back onto Glide's thread.
    cb.reschedule();
  } else {
    cb.onDataFetcherReady(loadData.sourceKey, data, loadData.fetcher,
        loadData.fetcher.getDataSource(), originalKey);
  }
}

@Override
public void onLoadFailed(@NonNull Exception e) {
  cb.onDataFetcherFailed(originalKey, e, loadData.fetcher, loadData.fetcher.getDataSource());
}
```

`onLoadFailed`很简单，直接调用`DecodeJob.onDataFetcherFailed`方法。`onDataReady`方法会首先判data能不能缓存，若能缓存则缓存起来，然后调用`DataCacheGenerator`进行加载缓存；若不能缓存，则直接调用`DecodeJob.onDataFetcherReady`方法通知外界data已经准备好了。

我们解读一下`onDataReady`里面的代码。首先，获取`DiskCacheStrategy`判断能不能被缓存，这里的判断代码在`SourceGenerator.startNext()`中出现过，显然是可以的。然后将data保存到`dataToCache`，并调用`cb.reschedule()`。
`cb.reschedule()`我们在前面分析过，该方法的作用就是将`DecodeJob`提交到Glide的source线程池中。然后执行`DecodeJob.run()`方法，经过`runWrapped()`、 `runGenerators()`方法后，又回到了`SourceGenerator.startNext()`方法。

在方法的开头，会判断`dataToCache`是否为空，此时显然不为空，所以会调用`cacheData(Object)`方法进行data的缓存处理。缓存完毕后，会为该缓存文件生成一个`SourceCacheGenerator`。然后在`startNext()`方法中会直接调用该变量进行加载。

```
@Override
public boolean startNext() {
  if (dataToCache != null) {
    Object data = dataToCache;
    dataToCache = null;
    cacheData(data);
  }

  if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
    return true;
  }
  sourceCacheGenerator = null;
}

private void cacheData(Object dataToCache) {
  long startTime = LogTime.getLogTime();
  try {
    Encoder<Object> encoder = helper.getSourceEncoder(dataToCache);
    DataCacheWriter<Object> writer =
        new DataCacheWriter<>(encoder, dataToCache, helper.getOptions());
    originalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());
    // 缓存data
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

由于在构造`DataCacheGenerator`时，指定了`FetcherReadyCallback`为自己，所以`DataCacheGenerator`加载结果会由`SourceGenerator`转发给`DecodeJob`。
由于资源会由`DataCacheGenerator`解码，所以我们可以在代码中看到，返回的data source是`DataSource.DATA_DISK_CACHE`。



我们先看一下fetch失败时干了什么，然后在看成功的时候。因为失败的代码比较简单。

`onDataFetcherFailed`代码如下：

```
@Override
public void onDataFetcherFailed(Key attemptedKey, Exception e, DataFetcher<?> fetcher,
    DataSource dataSource) {
  fetcher.cleanup();
  GlideException exception = new GlideException("Fetching data failed", e);
  exception.setLoggingDetails(attemptedKey, dataSource, fetcher.getDataClass());
  throwables.add(exception);
  if (Thread.currentThread() != currentThread) {
    runReason = RunReason.SWITCH_TO_SOURCE_SERVICE;
    callback.reschedule(this);
  } else {
    runGenerators();
  }
}
```

显然，如果fetch失败了，如果不在source线程池中就会切换到source线程，然后重新调用`runGenerators()`方法尝试使用下一个`DataFetcherGenerator`进行加载，一直到没有一个可以加载，这时会调用`notifyFailed()`方法，正式宣告加载失败。

然后在看成功的时候：`onDataFetcherReady`方法会保存传入的参数，然后确认执行线程后调用`decodeFromRetrievedData()`方法进行解码。

```
@Override
public void onDataFetcherReady(Key sourceKey, Object data, DataFetcher<?> fetcher,
    DataSource dataSource, Key attemptedKey) {
  this.currentSourceKey = sourceKey;
  this.currentData = data;
  this.currentFetcher = fetcher;
  this.currentDataSource = dataSource;
  this.currentAttemptingKey = attemptedKey;
  if (Thread.currentThread() != currentThread) {
    runReason = RunReason.DECODE_DATA;
    callback.reschedule(this);
  } else {
    GlideTrace.beginSection("DecodeJob.decodeFromRetrievedData");
    try {
      decodeFromRetrievedData();
    } finally {
      GlideTrace.endSection();
    }
  }
}
```

`decodeFromRetrievedData()`方法会先调用`decodeFromData`方法进行解码，然后调用`notifyEncodeAndRelease`方法进行缓存，同时也会通知`EngineJob`资源已经准备好了。

```
private void decodeFromRetrievedData() {
  if (Log.isLoggable(TAG, Log.VERBOSE)) {
    logWithTimeAndKey("Retrieved data", startFetchTime,
        "data: " + currentData
            + ", cache key: " + currentSourceKey
            + ", fetcher: " + currentFetcher);
  }
  Resource<R> resource = null;
  try {
    resource = decodeFromData(currentFetcher, currentData, currentDataSource);
  } catch (GlideException e) {
    e.setLoggingDetails(currentAttemptingKey, currentDataSource);
    throwables.add(e);
  }
  if (resource != null) {
    notifyEncodeAndRelease(resource, currentDataSource);
  } else {
    runGenerators();
  }
}
```

先看看decode相关的代码，`decodeFromData`相关的代码有一些，我们直接列出这些代码。`decodeFromData`方法内部又会调用`decodeFromFetcher`方法干活。在`decodeFromFetcher`方法中首先会获取LoadPath。然后调用`runLoadPath`方法解析成资源。

```
private <Data> Resource<R> decodeFromData(DataFetcher<?> fetcher, Data data,
    DataSource dataSource) throws GlideException {
  try {
    if (data == null) {
      return null;
    }
    long startTime = LogTime.getLogTime();
    Resource<R> result = decodeFromFetcher(data, dataSource);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logWithTimeAndKey("Decoded result " + result, startTime);
    }
    return result;
  } finally {
    fetcher.cleanup();
  }
}

@SuppressWarnings("unchecked")
private <Data> Resource<R> decodeFromFetcher(Data data, DataSource dataSource)
    throws GlideException {
  LoadPath<Data, ?, R> path = decodeHelper.getLoadPath((Class<Data>) data.getClass());
  return runLoadPath(data, dataSource, path);
}

private <Data, ResourceType> Resource<R> runLoadPath(Data data, DataSource dataSource,
    LoadPath<Data, ResourceType, R> path) throws GlideException {
  Options options = getOptionsWithHardwareConfig(dataSource);
  DataRewinder<Data> rewinder = glideContext.getRegistry().getRewinder(data);
  try {
    // ResourceType in DecodeCallback below is required for compilation to work with gradle.
    return path.load(
        rewinder, options, width, height, new DecodeCallback<ResourceType>(dataSource));
  } finally {
    rewinder.cleanup();
  }
}
```

注意`runLoadPath`方法使用到了`DataRewinder`，这是一个将数据流里面的指针重新指向开头的类，在调用`ResourceDecoder`对data进行编码时会尝试很多个编码器，所以每一次尝试后都需要重置索引。
在Glide初始化的时候默认注入了`ByteBufferRewinder`和`InputStreamRewinder`这两个类的工厂。这样就为`ByteBuffer`和`InputStream`的重定向提供了实现。

值得注意的是，在`path.load(rewinder, options, width, height, new DecodeCallback<ResourceType>(dataSource))`这行代码中，最后传入了一个`DecodeCallback`回调，该类的回调方法会回调给`DecodeJob`对应的方法：

```
private final class DecodeCallback<Z> implements DecodePath.DecodeCallback<Z> {

  private final DataSource dataSource;

  @Synthetic
  DecodeCallback(DataSource dataSource) {
    this.dataSource = dataSource;
  }

  @NonNull
  @Override
  public Resource<Z> onResourceDecoded(@NonNull Resource<Z> decoded) {
    return DecodeJob.this.onResourceDecoded(dataSource, decoded);
  }
}
```

然后我们看一下`LoadPath.load`方法的实现：

```
public Resource<Transcode> load(DataRewinder<Data> rewinder, @NonNull Options options, int width,
    int height, DecodePath.DecodeCallback<ResourceType> decodeCallback) throws GlideException {
  List<Throwable> throwables = Preconditions.checkNotNull(listPool.acquire());
  try {
    return loadWithExceptionList(rewinder, options, width, height, decodeCallback, throwables);
  } finally {
    listPool.release(throwables);
  }
}
```

ummmm，这里调用了`loadWithExceptionList`方法：

```
private Resource<Transcode> loadWithExceptionList(DataRewinder<Data> rewinder,
    @NonNull Options options,
    int width, int height, DecodePath.DecodeCallback<ResourceType> decodeCallback,
    List<Throwable> exceptions) throws GlideException {
  Resource<Transcode> result = null;
  //noinspection ForLoopReplaceableByForEach to improve perf
  for (int i = 0, size = decodePaths.size(); i < size; i++) {
    DecodePath<Data, ResourceType, Transcode> path = decodePaths.get(i);
    try {
      result = path.decode(rewinder, width, height, options, decodeCallback);
    } catch (GlideException e) {
      exceptions.add(e);
    }
    if (result != null) {
      break;
    }
  }

  if (result == null) {
    throw new GlideException(failureMessage, new ArrayList<>(exceptions));
  }

  return result;
}
```

对于每条DecodePath，都调用其`decode`方法，直到有一个DecodePath可以decode出资源。
那么我们继续看看`DecodePath.decode`方法：

```
public Resource<Transcode> decode(DataRewinder<DataType> rewinder, int width, int height,
    @NonNull Options options, DecodeCallback<ResourceType> callback) throws GlideException {
  Resource<ResourceType> decoded = decodeResource(rewinder, width, height, options);
  Resource<ResourceType> transformed = callback.onResourceDecoded(decoded);
  return transcoder.transcode(transformed, options);
}
```

显而易见，这里有3步：

1. 使用ResourceDecoder List进行decode
2. 将decoded的资源进行transform
3. 将transformed的资源进行transcode

在我们的示例中，第二条DecodePath(`DecodePath{ dataClass=class java.nio.DirectByteBuffer, decoders=[ByteBufferBitmapDecoder@1ca5fe14], transcoder=BitmapDrawableTranscoder@1d8f76bd}`)可以成功处理，并返回的是一个`LazyBitmapDrawableResource`对象。

我们看一下这里面的操作过程，首先是`decodeResource`的过程：

```
@NonNull
private Resource<ResourceType> decodeResource(DataRewinder<DataType> rewinder, int width,
    int height, @NonNull Options options) throws GlideException {
  List<Throwable> exceptions = Preconditions.checkNotNull(listPool.acquire());
  try {
    return decodeResourceWithList(rewinder, width, height, options, exceptions);
  } finally {
    listPool.release(exceptions);
  }
}

@NonNull
private Resource<ResourceType> decodeResourceWithList(DataRewinder<DataType> rewinder, int width,
    int height, @NonNull Options options, List<Throwable> exceptions) throws GlideException {
  Resource<ResourceType> result = null;
  //noinspection ForLoopReplaceableByForEach to improve perf
  for (int i = 0, size = decoders.size(); i < size; i++) {
    // decoders只有一条，就是ByteBufferBitmapDecoder
    ResourceDecoder<DataType, ResourceType> decoder = decoders.get(i);
    try {
      // rewinder自然是ByteBufferRewind
      // data为ByteBuffer
      DataType data = rewinder.rewindAndGet();
      // ByteBufferBitmapDecoder内部会调用Downsampler的hanldes方法
      // 它对任意的InputStream和ByteBuffer都返回true
      if (decoder.handles(data, options)) {
        // 调用ByteBuffer.position(0)复位
        data = rewinder.rewindAndGet();
        // 开始解码
        result = decoder.decode(data, width, height, options);
      }
      // Some decoders throw unexpectedly. If they do, we shouldn't fail the entire load path, but
      // instead log and continue. See #2406 for an example.
    } catch (IOException | RuntimeException | OutOfMemoryError e) {
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Failed to decode data for " + decoder, e);
      }
      exceptions.add(e);
    }

    if (result != null) {
      break;
    }
  }

  if (result == null) {
    throw new GlideException(failureMessage, new ArrayList<>(exceptions));
  }
  return result;
}
```

`ByteBufferBitmapDecoder.decode`方法会先将`ByteBuffer`转换成`InputStream`，然后在调用`Downsampler.decode`方法进行解码。

```
@Override
public Resource<Bitmap> decode(@NonNull ByteBuffer source, int width, int height,
    @NonNull Options options)
    throws IOException {
  InputStream is = ByteBufferUtil.toStream(source);
  return downsampler.decode(is, width, height, options);
}
```

这里面使用的技巧也主要是使用的[Bitmap的加载](https://blog.yorek.xyz/android/framework/Bitmap的缓存与加载/#1-bitmap)中提到的技巧。
不过在Glide中，除了设置了`BitmapFactory.Options`的`inJustDecodeBounds`和`inSampleSize`属性外，还会设置`inTargetDensity`、`inDensity`、`inScale`、`inPreferredConfig`、`inBitmap`属性。



这里执行完毕，会将decode出来的`Bitmap`包装成为一个`BitmapResource`对象。然后就一直往上返回，返回到`DecodePath.decode`方法中。接下来执行：

```
Resource<ResourceType> transformed = callback.onResourceDecoded(decoded);
```

这里的`callback`我们在前面提到过，这会调用`DecodeJob.onResourceDecoded(DataSource, Resource<Z>)`方法。

```
@Synthetic
@NonNull
<Z> Resource<Z> onResourceDecoded(DataSource dataSource,
    @NonNull Resource<Z> decoded) {
  @SuppressWarnings("unchecked")
  Class<Z> resourceSubClass = (Class<Z>) decoded.get().getClass();// Bitmap.class
  Transformation<Z> appliedTransformation = null;
  Resource<Z> transformed = decoded;
  // dataSource为DATA_DISK_CACHE，所以满足条件
  if (dataSource != DataSource.RESOURCE_DISK_CACHE) {
    // 在2.2节中给出了一个「optionalFitCenter()过程保存的KV表」，查阅得知Bitmap.class对应的正是FitCenter()
    appliedTransformation = decodeHelper.getTransformation(resourceSubClass);
    // 对decoded资源进行transform
    transformed = appliedTransformation.transform(glideContext, decoded, width, height);
  }
  // TODO: Make this the responsibility of the Transformation.
  if (!decoded.equals(transformed)) {
    decoded.recycle();
  }

  final EncodeStrategy encodeStrategy;
  final ResourceEncoder<Z> encoder;
  // Bitmap有注册对应的BitmapEncoder，所以是available的
  if (decodeHelper.isResourceEncoderAvailable(transformed)) {
    // encoder就是BitmapEncoder
    encoder = decodeHelper.getResultEncoder(transformed);
    // encodeStrategy为EncodeStrategy.TRANSFORMED
    encodeStrategy = encoder.getEncodeStrategy(options);
  } else {
    encoder = null;
    encodeStrategy = EncodeStrategy.NONE;
  }

  Resource<Z> result = transformed;
  // isSourceKey显然为true，所以isFromAlternateCacheKey为false，所以就返回了
  boolean isFromAlternateCacheKey = !decodeHelper.isSourceKey(currentSourceKey);
  // diskCacheStrategy为AUTOMATIC，该方法返回false
  if (diskCacheStrategy.isResourceCacheable(isFromAlternateCacheKey, dataSource,
      encodeStrategy)) {
    if (encoder == null) {
      throw new Registry.NoResultEncoderAvailableException(transformed.get().getClass());
    }
    final Key key;
    switch (encodeStrategy) {
      case SOURCE:
        key = new DataCacheKey(currentSourceKey, signature);
        break;
      case TRANSFORMED:
        key =
            new ResourceCacheKey(
                decodeHelper.getArrayPool(),
                currentSourceKey,
                signature,
                width,
                height,
                appliedTransformation,
                resourceSubClass,
                options);
        break;
      default:
        throw new IllegalArgumentException("Unknown strategy: " + encodeStrategy);
    }

    LockedResource<Z> lockedResult = LockedResource.obtain(transformed);
    deferredEncodeManager.init(key, encoder, lockedResult);
    result = lockedResult;
  }
  return result;
}
```

然后就回到`DecodePath.decode`方法的第三行了：

```
return transcoder.transcode(transformed, options);
```

这里的transcoder就是`BitmapDrawableTranscoder`，该方法返回了一个`LazyBitmapDrawableResource`。

至此，resource已经decode完毕。下面一直返回到`DecodeJob.decodeFromRetrievedData()`方法中。下面会调用`notifyEncodeAndRelease`方法完成后面的事宜。

```
private void notifyEncodeAndRelease(Resource<R> resource, DataSource dataSource) {
  // resource是BitmapResource类型，实现了Initializable接口
  if (resource instanceof Initializable) {
    // initialize方法调用了bitmap.prepareToDraw()
    ((Initializable) resource).initialize();
  }

  Resource<R> result = resource;
  LockedResource<R> lockedResource = null;
  // 由于在DecodeJob.onResourceDecoded方法中diskCacheStrategy.isResourceCacheable返回false
  // 所以没有调用deferredEncodeManager.init方法，因此此处为false
  if (deferredEncodeManager.hasResourceToEncode()) {
    lockedResource = LockedResource.obtain(resource);
    result = lockedResource;
  }

  // 通知回调，资源已经就绪
  notifyComplete(result, dataSource);

  stage = Stage.ENCODE;
  try {
    // 此处为false, skip
    if (deferredEncodeManager.hasResourceToEncode()) {
      deferredEncodeManager.encode(diskCacheProvider, options);
    }
  } finally {
    // lockedResource为null, skip
    if (lockedResource != null) {
      lockedResource.unlock();
    }
  }
  // Call onEncodeComplete outside the finally block so that it's not called if the encode process
  // throws.
  // 进行清理工作
  onEncodeComplete();
}
```

上面这段代码重点在于`notifyComplete`方法，该方法内部会调用`callback.onResourceReady(resource, dataSource)`将结果传递给回调，这里的回调是`EngineJob`：

```
// EngineJob.java
@Override
public void onResourceReady(Resource<R> resource, DataSource dataSource) {
  synchronized (this) {
    this.resource = resource;
    this.dataSource = dataSource;
  }
  notifyCallbacksOfResult();
}

void notifyCallbacksOfResult() {
  ResourceCallbacksAndExecutors copy;
  Key localKey;
  EngineResource<?> localResource;
  synchronized (this) {
    stateVerifier.throwIfRecycled();
    if (isCancelled) {
      // TODO: Seems like we might as well put this in the memory cache instead of just recycling
      // it since we've gotten this far...
      resource.recycle();
      release();
      return;
    } else if (cbs.isEmpty()) {
      throw new IllegalStateException("Received a resource without any callbacks to notify");
    } else if (hasResource) {
      throw new IllegalStateException("Already have resource");
    }
    // engineResourceFactory默认为EngineResourceFactory
    // 其build方法就是new一个对应的资源
    // new EngineResource<>(resource, isMemoryCacheable, /*isRecyclable=*/ true)
    engineResource = engineResourceFactory.build(resource, isCacheable);
    // Hold on to resource for duration of our callbacks below so we don't recycle it in the
    // middle of notifying if it synchronously released by one of the callbacks. Acquire it under
    // a lock here so that any newly added callback that executes before the next locked section
    // below can't recycle the resource before we call the callbacks.
    hasResource = true;
    copy = cbs.copy();
    incrementPendingCallbacks(copy.size() + 1);

    localKey = key;
    localResource = engineResource;
  }

  // listener就是Engine，该方法会讲资源保存到activeResources中
  listener.onEngineJobComplete(this, localKey, localResource);

  // 这里的ResourceCallbackAndExecutor就是我们在3.3节中创建EngineJob和DecodeJob
  // 并在执行DecodeJob之前添加的回调
  // entry.executor就是Glide.with.load.into中出现的Executors.mainThreadExecutor()
  // entry.cb就是SingleRequest
  for (final ResourceCallbackAndExecutor entry : copy) {
    entry.executor.execute(new CallResourceReady(entry.cb));
  }
  decrementPendingCallbacks();
}
```

`listener.onEngineJobComplete`的代码很简单。首先会设置资源的回调为自己，这样在资源释放时会通知自己的回调方法，将资源从active状态变为cache状态，如`onResourceReleased`方法；然后将资源放入activeResources中，资源变为active状态；最后将engineJob从Jobs中移除：

```
@Override
public synchronized void onEngineJobComplete(
    EngineJob<?> engineJob, Key key, EngineResource<?> resource) {
  // A null resource indicates that the load failed, usually due to an exception.
  if (resource != null) {
    resource.setResourceListener(key, this);

    if (resource.isCacheable()) {
      activeResources.activate(key, resource);
    }
  }

  jobs.removeIfCurrent(key, engineJob);
}

@Override
public synchronized void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
  activeResources.deactivate(cacheKey);
  if (resource.isCacheable()) {
    cache.put(cacheKey, resource);
  } else {
    resourceRecycler.recycle(resource);
  }
}
```

然后看下`entry.executor.execute(new CallResourceReady(entry.cb));`的实现，`Executors.mainThreadExecutor()`的实现之前说过，就是一个使用MainLooper的Handler，在execute Runnable时使用此Handler post出去。所以我们的关注点就在`CallResourceReady`上面了：

```
private class CallResourceReady implements Runnable {

    private final ResourceCallback cb;

    CallResourceReady(ResourceCallback cb) {
      this.cb = cb;
    }

    @Override
    public void run() {
      synchronized (EngineJob.this) {
        if (cbs.contains(cb)) {
          // Acquire for this particular callback.
          engineResource.acquire();
          callCallbackOnResourceReady(cb);
          removeCallback(cb);
        }
        decrementPendingCallbacks();
      }
    }
  }
```

ummmm，抛开同步操作不谈，首先调用`callCallbackOnResourceReady(cb)`调用callback，然后调用`removeCallback(cb)`移除callback。看看`callCallbackOnResourceReady(cb)`：

```
@Synthetic
  synchronized void callCallbackOnResourceReady(ResourceCallback cb) {
    try {
      // This is overly broad, some Glide code is actually called here, but it's much
      // simpler to encapsulate here than to do so at the actual call point in the
      // Request implementation.
      cb.onResourceReady(engineResource, dataSource);
    } catch (Throwable t) {
      throw new CallbackException(t);
    }
  }
```

这里就调用了`cb.onResourceReady`，这里说到过entry.cb就是SingleRequest。所以继续看看`SingleRequest.onResourceReady`方法，很显然`onResourceReady(Resource<?>, DataSource)`都在做sanity check，最后调用了`onResourceReady(Resource<?>, R, DataSource)`：

```
@Override
public synchronized void onResourceReady(Resource<?> resource, DataSource dataSource) {
  stateVerifier.throwIfRecycled();
  loadStatus = null;
  if (resource == null) {
    GlideException exception = new GlideException("Expected to receive a Resource<R> with an "
        + "object of " + transcodeClass + " inside, but instead got null.");
    onLoadFailed(exception);
    return;
  }

  Object received = resource.get();
  if (received == null || !transcodeClass.isAssignableFrom(received.getClass())) {
    releaseResource(resource);
    GlideException exception = new GlideException("Expected to receive an object of "
        + transcodeClass + " but instead" + " got "
        + (received != null ? received.getClass() : "") + "{" + received + "} inside" + " "
        + "Resource{" + resource + "}."
        + (received != null ? "" : " " + "To indicate failure return a null Resource "
        + "object, rather than a Resource object containing null data."));
    onLoadFailed(exception);
    return;
  }

  if (!canSetResource()) {
    releaseResource(resource);
    // We can't put the status to complete before asking canSetResource().
    status = Status.COMPLETE;
    return;
  }

  onResourceReady((Resource<R>) resource, (R) received, dataSource);
}
```

`onResourceReady(Resource<?>, R, DataSource)`方法如下，其处理过程和`onLoadFailed`方法非常类似：

```
private synchronized void onResourceReady(Resource<R> resource, R result, DataSource dataSource) {
  // We must call isFirstReadyResource before setting status.
  // 由于requestCoordinator为null，所以返回true
  boolean isFirstResource = isFirstReadyResource();
  // 将status状态设置为COMPLETE
  status = Status.COMPLETE;
  this.resource = resource;

  if (glideContext.getLogLevel() <= Log.DEBUG) {
    Log.d(GLIDE_TAG, "Finished loading " + result.getClass().getSimpleName() + " from "
        + dataSource + " for " + model + " with size [" + width + "x" + height + "] in "
        + LogTime.getElapsedMillis(startTime) + " ms");
  }

  isCallingCallbacks = true;
  try {
     // 尝试调用各个listener的onResourceReady回调进行处理
    boolean anyListenerHandledUpdatingTarget = false;
    if (requestListeners != null) {
      for (RequestListener<R> listener : requestListeners) {
        anyListenerHandledUpdatingTarget |=
            listener.onResourceReady(result, model, target, dataSource, isFirstResource);
      }
    }
    anyListenerHandledUpdatingTarget |=
        targetListener != null
            && targetListener.onResourceReady(result, model, target, dataSource, isFirstResource);

    // 如果没有一个回调能够处理，那么自己处理
    if (!anyListenerHandledUpdatingTarget) {
      // animationFactory默认为NoTransition.getFactory()，生成的animation为NO_ANIMATION
      Transition<? super R> animation =
          animationFactory.build(dataSource, isFirstResource);
      // target为DrawableImageViewTarget
      target.onResourceReady(result, animation);
    }
  } finally {
    isCallingCallbacks = false;
  }

  // 通知requestCoordinator
  notifyLoadSuccess();
}
```

`DrawableImageViewTarget`的基类`ImageViewTarget`实现了此方法：

```
// ImageViewTarget.java
@Override
public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
  // NO_ANIMATION.transition返回false，所以直接调用setResourceInternal方法
  if (transition == null || !transition.transition(resource, this)) {
    setResourceInternal(resource);
  } else {
    maybeUpdateAnimatable(resource);
  }
}

private void setResourceInternal(@Nullable Z resource) {
  // Order matters here. Set the resource first to make sure that the Drawable has a valid and
  // non-null Callback before starting it.
  // 先设置图片
  setResource(resource);
  // 然后如果是动画，会执行动画
  maybeUpdateAnimatable(resource);
}

private void maybeUpdateAnimatable(@Nullable Z resource) {
  // BitmapDrawable显然不是一个Animatable对象，所以走else分支
  if (resource instanceof Animatable) {
    animatable = (Animatable) resource;
    animatable.start();
  } else {
    animatable = null;
  }
}

// DrawableImageViewTarget
@Override
protected void setResource(@Nullable Drawable resource) {
  view.setImageDrawable(resource);
}
```









































