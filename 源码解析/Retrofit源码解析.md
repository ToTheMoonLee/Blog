# Retrofit源码解析

> 基于Retrofit2.9.0源码解析

### 概述

[Retrofit官方文档](https://square.github.io/retrofit/)

可以看到，官方给出的第一句话是，Retrofit是一个类型安全的Http客户端库。所谓类型安全就是说在运行的时候不会报类型错误，即在编译期就检查出类型错误。Retrofit对OkHttp库进行了进一步的封装，进行了功能上的收窄，但是使用起来也会更加简单。

### 使用

第一步，根据官方文档的步骤，首先创建一个HTTP API的接口

```
interface ApiService {
    @GET("/")
    fun call(): Call<ResponseBody?>
}
```

第二步，使用Retrofit对象生成这个接口对象，并调用

```
val retrofit = Retrofit.Builder().baseUrl("https://api.github.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
        val apiService = retrofit.create(ApiService::class.java)
        val repos = apiService.getResponse("octocat")
        repos.enqueue(object : Callback<List<Repo?>> {
            override fun onFailure(call: Call<List<Repo?>>, t: Throwable) {
                println("response is failure : ${t}")
            }

            override fun onResponse(call: Call<List<Repo?>>, response: Response<List<Repo?>>) {
                println("response is successful: ${response.body()?.get(0)?.name}")
            }

        })
```

当然这只是Retrofit的简单使用，更多用法直接看官方文档即可。下面开始从入口分析Retrofit的源码结构。

### 源码分析

一般为了最快速的搞懂代码的结构，我们就从最直接的调用处入手，这里也就是从`enqueue()`方法入手，代码如下：

```
public interface Call<T> extends Cloneable {
	void enqueue(Callback<T> callback);
}
```

可以发现，`enqueue()`方法是Retroft中的一个`Call`的接口中的方法，为了看到这个具体的实现，所以我们需要继续往上找，看下`repos`是什么，代码如下：

```
interface ApiService {
    @GET("users/{user}/repos")
    fun getResponse(@Path("user") user: String?): Call<List<Repo?>>
}
```

可以看到`repos`就是我们定义的接口的返回值，那么我们就需要继续往上跟了，看下`service`是如何被造出来了，所以我们看下`Retrofit.create()`方法：

```
public final class Retrofit {
...
	public <T> T create(final Class<T> service) {
	// 1. 验证service接口
	validateServiceInterface(service);
	// 2. 使用Proxy.newProxyInstance来创建传入的定义的service的对象
	return (T)
	   Proxy.newProxyInstance(
	       service.getClassLoader(),
	       new Class<?>[] {service},
	       new InvocationHandler() {
	         private final Platform platform = Platform.get();
	         private final Object[] emptyArgs = new Object[0];
	
	         @Override
	         public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
	             throws Throwable {
	           // If the method is a method from Object then defer to normal invocation.
	           if (method.getDeclaringClass() == Object.class) {
	             return method.invoke(this, args);
	           }
	           args = args != null ? args : emptyArgs;
	           return platform.isDefaultMethod(method)
	               ? platform.invokeDefaultMethod(method, service, proxy, args)
	               : loadServiceMethod(method).invoke(args);
	         }
	       });
	}
}
```

这部分代码主要分为两部分，第一步，验证传入的`service.class`；第二步，使用`Proxy.newProxyInstance`来动态创建`service`的实体类。先看下`validateServiceInterface()`的实现：

```
private void validateServiceInterface(Class<?> service) {
	// 1. 验证传入的serice是否是一个类，isInterface()是Class类中的一个方法，里面具体的判断是验证表示类修饰符的flag是否是INTERFACE，JVM中已经定义好的
    if (!service.isInterface()) {
      throw new IllegalArgumentException("API declarations must be interfaces.");
    }
	// 2. 从传入的类开始依次向上遍历父类，看是有带泛型的接口，如果有就报错
    Deque<Class<?>> check = new ArrayDeque<>(1);
    check.add(service);
    while (!check.isEmpty()) {
      Class<?> candidate = check.removeFirst();
      if (candidate.getTypeParameters().length != 0) {
        StringBuilder message =
            new StringBuilder("Type parameters are unsupported on ").append(candidate.getName());
        if (candidate != service) {
          message.append(" which is an interface of ").append(service.getName());
        }
        throw new IllegalArgumentException(message.toString());
      }
      Collections.addAll(check, candidate.getInterfaces());
    }
	// 3. 在调用Retrofit的方法的时候，第一次会进行初始化操作，主要是验证一些方法的合法性等，只有在调用的时候才验证。而这里是判断是否用于激进的验证，如果开启，则在初始化的时候，就将所有的方法都初始化出来。主要用于在开发时，保证将错误第一时间暴露出来。正式环境不建议打开，因为在loadServiceMethod执行的时候会用到一点点反射，如果全部都在一开始就进行验证，则会造成性能的损失。而不打开这个开关的话，则可以将时间均摊到使用时
    if (validateEagerly) {
      Platform platform = Platform.get();
      // Java8可以写默认方法和静态方法，这里Retrofit对其进行了过滤
      for (Method method : service.getDeclaredMethods()) {
        if (!platform.isDefaultMethod(method) && !Modifier.isStatic(method.getModifiers())) {
          loadServiceMethod(method);
        }
      }
    }
  }
```

可以看到，这个`validateServiceInterface()`方法分为三部分，主要用来验证传入的service的合法性的，那么继续回头看下`Retrofit.create()`的第二部分，return的部分：

```
public final class Retrofit {
...
	public <T> T create(final Class<T> service) {
	// 2. 使用Proxy.newProxyInstance来创建传入的定义的service的对象
	return (T)
	   Proxy.newProxyInstance(
	       service.getClassLoader(),
	       new Class<?>[] {service},
	       new InvocationHandler() {
	         private final Platform platform = Platform.get();
	         private final Object[] emptyArgs = new Object[0];
	
	         @Override
	         public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
	             throws Throwable {
	           // If the method is a method from Object then defer to normal invocation.
	           if (method.getDeclaringClass() == Object.class) {
	             return method.invoke(this, args);
	           }
	           args = args != null ? args : emptyArgs;
	           return platform.isDefaultMethod(method)
	               ? platform.invokeDefaultMethod(method, service, proxy, args)
	               : loadServiceMethod(method).invoke(args);
	         }
	       });
	}
}
```

第二部分的代码，用到了`Proxy.newProxyInstance()`，也就是大家一直说的，动态代理。在说动态代理之前，首先看看静态代理，静态代理，就是有一个客户类，实现了一个特定的接口，而另外一个代理类持有这个类的对象，在内部调用客户类的方法，说的比较抽象，直接看代码：

```
interface  IClient {
    void say();
}

public class RealClient implements IClient {
    @Override
    public void say() {
        System.out.println("I am real client");
    }
}

public class ProxyClient implements IClient{
    private IClient client;
    public ProxyClient() {
        client = new RealClient();
    }
    @Override
    public void say() {
        client.say();
        System.out.println("I am myproxy client");
    }
}


public static void main(String[] args) {
   ProxyClient proxyClient = new ProxyClient();
   proxyClient.say();
}
```

使用静态代理，可以做到客户类的隔离；同时，使用代理类还能增强原有的类的功能，比如这个例子中，添加了`System.out.println("I am myproxy client");`，而不需要更改原来的类，符合封闭原则。

而动态代理，则是在程序运行期间，由JVM自动生成代理类（比如使用反射机制），代理关系是在运行时确定的，代码如下：

```
IClient client1 = (IClient) Proxy.newProxyInstance(IClient.class.getClassLoader(), new Class<?>[]{IClient.class}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                return method.invoke(new RealClient(), args);
            }
        });

        client1.say();

        IClient client2 = (IClient) Proxy.newProxyInstance(IClient.class.getClassLoader(), new Class<?>[]{IClient.class}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                return method.invoke(new RealClient2(), args);
            }
        });

        client2.say();
```

使用`Proxy.newProxyInstance()`最终会在运行期间依赖`InvocationHandler()`的方法，动态的生成代理类。至于动态生成的原理，其实就是根据传入的类，动态的生成代理类的字节码文件的过程，主要代码逻辑如下：

```
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
    ...
    		// 拿到生成的代理类字节码，这里是真正生成字节码的位置
           Class<?> cl = getProxyClass0(loader, intfs);
    ...
    final Constructor<?> cons = cl.getConstructor(constructorParams);
    final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
    // 使用反射生成代理对象并返回
     return cons.newInstance(new Object[]{h});
    }
```

继续看`getProxyClass0(loader, intfs);`的方法：

```
    private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        return proxyClassCache.get(loader, interfaces);
    }
```

其实注释里也已经写明白了，首先会从缓存里拿，如果拿到则直接返回，如果拿不到则会使用ProxyClassFactory类生成字节码，继续查看`get()`方法：

```
public V get(K key, P parameter) {
        ...
        // 核心方法其实就是subKeyFactory.apply(key, parameter)方法，会使用subKeyFactory.apply的apply()方法来生成字节码文件，实际上就是ProxyClassFactory的apply()方法，继续看下ProxyClassFactory的apply()
        // create subKey and retrieve the possible Supplier<V> stored by that
        // subKey from valuesMap
        Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));

...
    }
```

这里核心就是使用`ProxyClassFactory`的`apply()`来生成字节码，继续查看`ProxyClassFactory`的`apply()`做了什么：

```
public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
			...
			// 会生成proxyName的包名

            if (proxyPkg == null) {
                // if no non-public proxy interfaces, use com.sun.proxy package
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }

            /*
             * Choose a name for the proxy class to generate.
             */
            long num = nextUniqueNumber.getAndIncrement();
            String proxyName = proxyPkg + proxyClassNamePrefix + num;

            /*
             * Generate the specified proxy class.
             */
             // 这里是核心，会生成对应的字节码文件
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
                throw new IllegalArgumentException(e.toString());
            }
        }
    }
```

最终会传入的对应的参数，使用`ProxyGenerator.generateProxyClass()`的方法，来动态的生成代理类的字节码，其实就是根据JVM的规范，生成字节码文件，使用IO写入内存而已，具体代码见：

[generateProxyClass](https://github.com/JetBrains/jdk8u_jdk/blob/master/src/share/classes/sun/misc/ProxyGenerator.java)

[generateClassFile](https://github.com/JetBrains/jdk8u_jdk/blob/94318f9185757cc33d2b8d527d36be26ac6b7582/src/share/classes/sun/misc/ProxyGenerator.java#L56)

![class_arrangement](https://github.com/ToTheMoonLee/Blog/blob/main/JVM/illustration/class_code_image.jpg)

所以到这里，我们就知道了，我们所有的service里面的方法调用都被`InvocationHandler`的`invoke()`方法给代理了（有点像`OnclickListener()`），所以我们重点看下这个方法里面的核心方法`loadServiceMethod(method)`又做了什么：

```
public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                  throws Throwable {
                // If the method is a method from Object then defer to normal invocation.
                // 1. 如果是Object中的方法，比如toString()等，直接返回，不做代理
                if (method.getDeclaringClass() == Object.class) {
                  return method.invoke(this, args);
                }
                args = args != null ? args : emptyArgs;
                // 2. 如果是Java8里的默认方法，也不做代理，直接返回
                return platform.isDefaultMethod(method)
                    ? platform.invokeDefaultMethod(method, service, proxy, args)
                    // 3. 如果都不是，那么调用loadServiceMethod(method)的invoke()方法
                    : loadServiceMethod(method).invoke(args);
              }
```	

可以看到核心方法其实就是`loadServiceMethod(method).invoke(args)`，这里`invoke(args)`是一个接口方法，所以我们要重点关注下`loadServiceMethod()`，代码如下：

```
ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = ServiceMethod.parseAnnotations(this, method);
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

代码结构其实还是很简单的，如果要是能从缓存里取到，那么就直接拿到返回，如果没有取到，则调用`ServiceMethod.parseAnnotations(this, method)`来创建`ServiceMethod`，所以，继续看这个核心方法：

```
abstract class ServiceMethod<T> {
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(
          method,
          "Method return type must not include a type variable or wildcard: %s",
          returnType);
    }
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }

    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }

  abstract @Nullable T invoke(Object[] args);
}
```

可以看到，其实`parseAnnotations()`跟`invoke()`都是同一个类里的，而这个方法的核心方法也是最后一行`HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory)`，所以我们继续点击看下这个方法：

```
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
	... 
	
    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    if (!isKotlinSuspendFunction) {
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    } 
    ... 
  }
```

留下核心代码，可以看到，这里又返回了一个`new CallAdapted()`，但是注意到这个方法属于`class HttpServiceMethod<ResponseT, ReturnT> extends ServiceMethod<ReturnT>`类，而这个类正式我们要找的`ServiceMethod`的子类，所以，我们直接看下这里面的`invoke()`方法的调用：

```
  @Override
  final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
  }

```

可以看到，这里创建了一个`OkHttpCall`的对象，根据经验，`adapt()`类型的方法，其实都是非核心的，所以我们直接看看`OkHttpCall`又干了什么：

```
final class OkHttpCall<T> implements Call<T> {
	...
}
```

注意到`OkHttpCall`正式我们一开始就找的`Call<T>`的子类，所以我们直接看下`OkHttpCall`的`enqueue()`方法即可：

```
public void enqueue(final Callback<T> callback) {
    Objects.requireNonNull(callback, "callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
        // 核心代码是createRawCall()，这个方法会创建一个okhttp3.Call对象，这个是okhttp3的call对象，使用的是  override fun newCall(request: Request): Call = RealCall(this, request, forWebSocket = false)方法来创建的。
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          throwIfFatal(t);
          failure = creationFailure = t;
        }
      }
    }

    if (failure != null) {
      callback.onFailure(this, failure);
      return;
    }

    if (canceled) {
      call.cancel();
    }
	// 这里调用call.enqueue()与OkHttp里的用法就一样了
    call.enqueue(
        new okhttp3.Callback() {
          @Override
          public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
            Response<T> response;
            try {
            // 这个位置会将返回的response进行解析成Retrofit的Response
              response = parseResponse(rawResponse);
            } catch (Throwable e) {
              throwIfFatal(e);
              callFailure(e);
              return;
            }

            try {
            // 最终会使用我们传入的callback.onResponse的回调
              callback.onResponse(OkHttpCall.this, response);
            } catch (Throwable t) {
              throwIfFatal(t);
              t.printStackTrace(); // TODO this is not great
            }
          }

          @Override
          public void onFailure(okhttp3.Call call, IOException e) {
            callFailure(e);
          }

          private void callFailure(Throwable e) {
            try {
              callback.onFailure(OkHttpCall.this, e);
            } catch (Throwable t) {
              throwIfFatal(t);
              t.printStackTrace(); // TODO this is not great
            }
          }
        });
  }
```

最终我们看到，实际上是使用OkHttp的Call对象进行了`enqueue()`的调用，从而实现了网络交互。
这里先总结一下：
1. 使用`retrofit.create(ApiService::class.java)`来创建代理对象，代理对象中真正调用某个方法的时候，在这个例子中会返回对应的`call`（Retrofit的call）对象
2. `retrofit.create(ApiService::class.java)`中的核心是`InvocationHandler`类的`invoke()`方法，这个方法会在请求网络方法调用的时候调用，在这个例子中，就是`apiService.getResponse("octocat")`调用的时候会被调用，并返回一个`Call`对象
3. `InvocationHandler`类的`invoke()`方法中，真正会调用`ServiceMethod`的`invoke()`方法，而`ServiceMethod`类真正调用`invoke()`的是`HttpServiceMethod`中的`invoke()`方法
4. `HttpServiceMethod`中的`invoke()`方法真正执行的是`new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);`，虽然里面有`adapt(call, args)`方法，但是不是核心，其实核心逻辑就是返回一个`OkHttpCall`的对象
5. 从而之后执行`repos.enqueue()`方法的时候，真正执行的就是`OkHttpCall`的`enqueue()`方法，而该方法中真正执行的是`OkHttp`的`Call`的`enqueue()`方法，并将结果解析后返回

接下来，我们来看下`adapt(call, args)`方法又到底做了什么事情，首先，还是从`HttpServiceMethod<ResponseT, ReturnT> parseAnnotations()`方法开始：

```
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
 ... 
 // 最终CallAdapted的adapt方法其实是调用的callAdapter.adapt(call)，所以需要看下CallAdapter是如何创建的
    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
 	...
    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    // 返回一个包装了callAdapter的CallAdapted对象，而之后调用的invoke()其实是调用的CallAdapted的invoke()方法
    if (!isKotlinSuspendFunction) {
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    } 
  }
  
  // CallAdapted的adapt方法其实是调用的callAdapter.adapt(call)
  protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
      return callAdapter.adapt(call);
    }
```

可以看到，其实`parseAnnotations()`最终会返回一个`CallAdapted`对象，而`CallAdapted`中的`invoke()`实际调用的就是`callAdapter.adapt(call)`，所以需要看下他的创建：

```
 private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
      Retrofit retrofit, Method method, Type returnType, Annotation[] annotations) {
    try {
    // 使用retrofit.callAdapter创建，继续跟
      //noinspection unchecked
      return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
    } 
  }
  
  // Retrofit.java 类的callAdapter方法返回
    public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }
  
  // Retrofit.java 的nextCallAdapter中会使用callAdapterFactories.get()来获取到CallAdapter并返回，继续看callAdapterFactories是从哪儿来的
  public CallAdapter<?, ?> nextCallAdapter(
      @Nullable CallAdapter.Factory skipPast, Type returnType, Annotation[] annotations) {
      ... 
    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
	 ... 
  }
  
  // Retrofit.java 的初始化方法传入了callAdapterFactories，继续看调用初始化方法的地方
  Retrofit(
      okhttp3.Call.Factory callFactory,
      HttpUrl baseUrl,
      List<Converter.Factory> converterFactories,
      List<CallAdapter.Factory> callAdapterFactories,
      @Nullable Executor callbackExecutor,
      boolean validateEagerly) {
    this.callFactory = callFactory;
    this.baseUrl = baseUrl;
    this.converterFactories = converterFactories; // Copy+unmodifiable at call site.
    this.callAdapterFactories = callAdapterFactories; // Copy+unmodifiable at call site.
    this.callbackExecutor = callbackExecutor;
    this.validateEagerly = validateEagerly;
  }
  
   // Retrofit.java 的build()方法中，会默认创建一个platform.defaultCallAdapterFactories的factories
   
   public Retrofit build() {
     	...
           Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
      // 创建了默认的factory
      callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories =
          new ArrayList<>(
              1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());
		... 
      return new Retrofit(
          callFactory,
          baseUrl,
          unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories),
          callbackExecutor,
          validateEagerly);
    }

```

这里可以看出，首先从`Retrofit`的`build()`方法中，初始化了一个默认的`DefaultCallAdapterFactory`，然后这个对象会保存在`Retrofit`对象中，而`Retrofit`的`callAdapter()`方法又会使用这个对象的`get()`方法，来初始化一个`Adapter`对象，而在`HttpServiceMethod`的`createCallAdapter()`方法中，会调用`Retrofit`的`callAdapter()`方法来初始化这个类，所以我们只需要看看`DefaultCallAdapterFactory`到底初始化了什么类即可，直接看该类的get()方法：

```
@Override
  public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    if (!(returnType instanceof ParameterizedType)) {
      throw new IllegalArgumentException(
          "Call return type must be parameterized as Call<Foo> or Call<? extends Foo>");
    }
    final Type responseType = Utils.getParameterUpperBound(0, (ParameterizedType) returnType);

    final Executor executor =
        Utils.isAnnotationPresent(annotations, SkipCallbackExecutor.class)
            ? null
            : callbackExecutor;
	// 最终会返回一个CallAdapter对象
    return new CallAdapter<Object, Call<?>>() {
      @Override
      public Type responseType() {
        return responseType;
      }
		// 不要忘了，我们重点是要看adapt()方法
      @Override
      public Call<Object> adapt(Call<Object> call) {
        return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
      }
    };
  }
```

看到最终会返回一个CallAdapter对象，终于可以看到真正调用`adapt()`的实现了，可以看到，这里是真正返回了一个`ExecutorCallbackCall`的对象，而这个`Call`中的`enqueue()`最终干了什么事儿呢，继续往进跟：

```
    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }
@Override
    public void enqueue(final Callback<T> callback) {
      Objects.requireNonNull(callback, "callback == null");
		// 可以看到，使用了delegate这个call对象的enqueue()方法，其实实际上，就是我们初始化的OkHttpCall对象的enqueue()方法，但是里面在onResponse又使用到了一个线程池的callbackExecutor.execute()方法
      delegate.enqueue(
          new Callback<T>() {
            @Override
            public void onResponse(Call<T> call, final Response<T> response) {
              callbackExecutor.execute(
                  () -> {
                    if (delegate.isCanceled()) {
                      // Emulate OkHttp's behavior of throwing/delivering an IOException on
                      // cancellation.
                      callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                    } else {
                      callback.onResponse(ExecutorCallbackCall.this, response);
                    }
                  });
            }

            @Override
            public void onFailure(Call<T> call, final Throwable t) {
              callbackExecutor.execute(() -> callback.onFailure(ExecutorCallbackCall.this, t));
            }
          });
    }
```

可以看到这里实质上就是调用了`OkHttpCall`的`euqueue()`方法，但是为什么会使用到了一个线程池的`callbackExecutor.execute()`方法呢，继续看下`callbackExecutor`是从哪儿来的。其实跟代码可以看到，这个`executor`是从`Retrofit`的`build()`中初始化的：

```
      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }
      
      // 真正的实现类
      static final class Android extends Platform {
    Android() {
      super(Build.VERSION.SDK_INT >= 24);
    }

    @Override
    public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    @Nullable
    @Override
    Object invokeDefaultMethod(
        Method method, Class<?> declaringClass, Object object, Object... args) throws Throwable {
      if (Build.VERSION.SDK_INT < 26) {
        throw new UnsupportedOperationException(
            "Calling default methods on API 24 and 25 is not supported");
      }
      return super.invokeDefaultMethod(method, declaringClass, object, args);
    }

	// MainThreadExecutor中使用了Looper.getMainLooper，从而将后台线程切换到了前台
    static final class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override
      public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }
```

看到这里就明白了，原来是`Retrofit`使用`MainThreadExecutor`自动将返回Response的调用从子线程切到了住线程，从而不用我们自己手动切线程了。

再次总结一下：
1. 在`Retrofit`配置的过程中，会初始化一个`DefaultCallAdapterFactory`对象，这个对象会用来初始化`ExecutorCallbackCall`对象，这`ExecutorCallbackCall`对象中会用到一个`MainThreadExecutor`来切换主线程
2. 在`HttpServiceMethod`的`parseAnnotations()`方法中会返回`CallAdapted`的对象，而其中的	`adapt()`方法进而会调用`CallAdapter`的`adapt()`来返回一个`ExecutorCallbackCall`对象
3. 拿到这个对象之后，则会真正调用`OkHttpCall`的`enqueue()`，同时在`enqueue()`中使用主线程的	`Executor`来切换主线程

主流程大概搞明白了，接下来看下其他的细节，首先看下创建OkHttp3的Call的参数是哪儿来的，看下代码：

```
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
// 重点关注下这个位置
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
    ...
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }
  
  // RequestFactory.java  中的这个方法主要是使用build()来创建一个RequestFactory对象
  
    static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    return new Builder(retrofit, method).build();
  }
  
   // RequestFactory.java Builder方法中会拿到方法的注解，参数类型等等
      Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      this.methodAnnotations = method.getAnnotations();
      this.parameterTypes = method.getGenericParameterTypes();
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }
    
    // RequestFactory.java  中的build()方法重点是parseMethodAnnotation(annotation）、parseParameter()等，来分析出各种记录，从而根据这些数据拼装一个request，就是okhttp3的使用方案
    RequestFactory build() {
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      if (httpMethod == null) {
        throw methodError(method, "HTTP method annotation is required (e.g., @GET, @POST, etc.).");
      }

      if (!hasBody) {
        if (isMultipart) {
          throw methodError(
              method,
              "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
          throw methodError(
              method,
              "FormUrlEncoded can only be specified on HTTP methods with "
                  + "request body (e.g., @POST).");
        }
      }

      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0, lastParameter = parameterCount - 1; p < parameterCount; p++) {
        parameterHandlers[p] =
            parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p], p == lastParameter);
      }

      if (relativeUrl == null && !gotUrl) {
        throw methodError(method, "Missing either @%s URL or @Url parameter.", httpMethod);
      }
      if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
        throw methodError(method, "Non-body HTTP method cannot contain @Body.");
      }
      if (isFormEncoded && !gotField) {
        throw methodError(method, "Form-encoded method must contain at least one @Field.");
      }
      if (isMultipart && !gotPart) {
        throw methodError(method, "Multipart method must contain at least one @Part.");
      }

      return new RequestFactory(this);
    }
    
    // 其实就是用来对方法的注解进行解析，并记录
    private void parseMethodAnnotation(Annotation annotation) {
      if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
      } else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
      } else if (annotation instanceof PATCH) {
        parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
      } else if (annotation instanceof POST) {
        parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
      } else if (annotation instanceof PUT) {
        parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
      } else if (annotation instanceof OPTIONS) {
        parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
      } else if (annotation instanceof HTTP) {
        HTTP http = (HTTP) annotation;
        parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
      } else if (annotation instanceof retrofit2.http.Headers) {
        String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
        if (headersToParse.length == 0) {
          throw methodError(method, "@Headers annotation is empty.");
        }
        headers = parseHeaders(headersToParse);
      } else if (annotation instanceof Multipart) {
        if (isFormEncoded) {
          throw methodError(method, "Only one encoding annotation is allowed.");
        }
        isMultipart = true;
      } else if (annotation instanceof FormUrlEncoded) {
        if (isMultipart) {
          throw methodError(method, "Only one encoding annotation is allowed.");
        }
        isFormEncoded = true;
      }
    }
```

总之就是根据对方法的解析，得到Okhttp3需要使用的各种参数，并帮我们生成request、call等。

接下来继续看看convertor是如何使用的，其实跟callAdapterFactories的方式是一样的，都是从Retrofit的build中传入进来的，只不过在使用的时候，是在response方法中使用的，代码如下：

```
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
	....
    try {
    // 这里就是使用responseConverter.convert放啊放来进行转换
      T body = responseConverter.convert(catchingBody);
      return Response.success(body, rawResponse);
    } 
    ...
  }
```

比如我们使用的是`GsonConverterFactory.create()`，那么其实内部还是使用的`Gson`来对返回值进行解析。

关于与RxJava结合可以直接在配置Retrofit的时候，加上`.addCallAdapterFactory(RxJava2CallAdapterFactory.create())`，这样在`callAdapterFactories`去取`Adapter`的时候，就会去自动匹配：

```
public CallAdapter<?, ?> nextCallAdapter(
      @Nullable CallAdapter.Factory skipPast, Type returnType, Annotation[] annotations) {
    Objects.requireNonNull(returnType, "returnType == null");
    Objects.requireNonNull(annotations, "annotations == null");

    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
    // 在这里就会使用RxJava2CallAdapterFactory的get()方法来初始化CallAdapter
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
```

具体初始化的代码就需要看`RxJava2CallAdapterFactory`的`get()`的方法了：

```
public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    Class<?> rawType = getRawType(returnType);

    if (rawType == Completable.class) {
      // Completable is not parameterized (which is what the rest of this method deals with) so it
      // can only be created with a single configuration.
      return new RxJava2CallAdapter(
          Void.class, scheduler, isAsync, false, true, false, false, false, true);
    }

    boolean isFlowable = rawType == Flowable.class;
    boolean isSingle = rawType == Single.class;
    boolean isMaybe = rawType == Maybe.class;
    if (rawType != Observable.class && !isFlowable && !isSingle && !isMaybe) {
      return null;
    }

    boolean isResult = false;
    boolean isBody = false;
    Type responseType;
    if (!(returnType instanceof ParameterizedType)) {
      String name =
          isFlowable ? "Flowable" : isSingle ? "Single" : isMaybe ? "Maybe" : "Observable";
      throw new IllegalStateException(
          name
              + " return type must be parameterized"
              + " as "
              + name
              + "<Foo> or "
              + name
              + "<? extends Foo>");
    }

    Type observableType = getParameterUpperBound(0, (ParameterizedType) returnType);
    Class<?> rawObservableType = getRawType(observableType);
    if (rawObservableType == Response.class) {
      if (!(observableType instanceof ParameterizedType)) {
        throw new IllegalStateException(
            "Response must be parameterized" + " as Response<Foo> or Response<? extends Foo>");
      }
      responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
    } else if (rawObservableType == Result.class) {
      if (!(observableType instanceof ParameterizedType)) {
        throw new IllegalStateException(
            "Result must be parameterized" + " as Result<Foo> or Result<? extends Foo>");
      }
      responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
      isResult = true;
    } else {
      responseType = observableType;
      isBody = true;
    }

    return new RxJava2CallAdapter(
        responseType, scheduler, isAsync, isResult, isBody, isFlowable, isSingle, isMaybe, false);
  }
```

上述代码中会先判断是否是`Flowable.class`，是否是`Single.class`等等，从而最终会返回一个`RxJava2CallAdapter`的对象，从而就能够与`RxJava`相结合了





