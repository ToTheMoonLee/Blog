# OKHttp源码解析

> 基于okhttp 4.9.0版本进行源码解析

### 简单使用

首先，先看一下OkHttp的简单用法

```
   val client = OkHttpClient.Builder().build()
   val request = Request.Builder().url("https://www.baidu.com/").get().build()
   client.newCall(request).enqueue(object : Callback {
       override fun onFailure(call: Call, e: IOException) {
           println("error message is ${e.message}")
       }

       override fun onResponse(call: Call, response: Response) {
           println("response code is ${response.code}  content is ${request.body.toString()}")
       }
   })
```

代码很简单，就是初始化了一个OkHttpClient对象和一个Request对象，然后调用OkHttpClient的newCall(request)方法初始化一个Call对象，最后调用Call对象的enqueue()方法，就完成了一次Http的请求，那么我们直接从enqueue()入手，一层一层的剖析。

### enqueue逻辑分析

首先点进去enqueue()方法，会发现这个方法是一个接口Call的方法，那么我们继续查看client.newCall(request)查看初始化的这个Call的实际子类，newCall(request)的代码如下：

```
  override fun newCall(request: Request): Call = RealCall(this, request, forWebSocket = false)
```

可以看出，实际是初始化了一个RealCall对象，那么我们继续查看RealCall的enqueue()方法：

```
  override fun enqueue(responseCallback: Callback) {
  	// 用来避免重复执行call的一个判断
    check(executed.compareAndSet(false, true)) { "Already Executed" }
		// 这个方法中调用了eventListener的callStart方法
    callStart()
    // 实际执行的是Dispatcher的enqueue方法，并且讲responseCallback包装了AsyncCall
    client.dispatcher.enqueue(AsyncCall(responseCallback))
  }
```

上面的代码可以看出：
	 1. 首先会判断是否同一个call多次调用enqueue方法，已经调用equeue方法的call，再次调用enqueue会抛出异常
	 2. 会调用eventListener.callStart(this)，EventListener是一个抽象类，我们可以自己实现这个抽象类，并实现相关的方法，来监听各个阶段的方法耗时。使用方法就是OkHttpclient初始化的时候，使用OkHttpclient.Builder的`fun eventListener(eventListener: EventListener)`方法将我们自己实现的EventListener添加进去即可
	 3. 最后真正执行enqueue的其实是Dispatcher

那么继续看下Dispatcher的enqueue又做了什么，首先说下Dispatcher这个类的功能，如果看过其他的开源库源码的话，大家应该都知道Dispatcher一般都是做线程调度的，进而实现对多线程的管理，其中使用ExecutorService去执行Call的run方法。其中有两个可配置的变量，一个是maxRequests，最大可同时执行的requests的个数，超出这个值的就需要排队等待。还有一个是maxRequestsPerHost，这个是每个主机可同时执行的最大的requests的个数，同样超出这个值就会排队。代码如下：

```
internal fun enqueue(call: AsyncCall) {	// 同步操作
    synchronized(this) {
    // 首先会将call添加到待执行的list中，这个list的结构是一个ArrayDeque，双端队列
      readyAsyncCalls.add(call)

      // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to the same host.
      // forWebSocket是用来判断，我们是否要使用WebSocket这种网络协议进行传输的，这种协议可以在可在单个TCP连接上进行全双工通信，一般我们用不到，所以这个值为false，而这里传进来的也是false
      if (!call.call.forWebSocket) {
      // findExistingCallWithHost做的操作是在runningAsyncCalls和readyAsyncCalls这两个list中找到主机名相同的AsyncCall，如果找到的话，就把当前这个call的callsPerHost的值改成队列里的那个
        val existingCall = findExistingCallWithHost(call.host)
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
      }
    }
    // 这个方法会找到有执行资格的call，并将其放到ExecutorService去执行
    promoteAndExecute()
  }
```

可以看到Diapatcher的enqueue主要做了如下几件事：
	1. 将call放入到一个待执行的双端队列ArrayDeque
	2. 从runningAsyncCalls和readyAsyncCalls中找到已经存在主机名相同的AsyncCall，并将这个existingCall的callsPerHost（这个是用来记数每个host的call的数量的，是一个原子操作AtomicInteger）赋值给当前的call。这个值就是用来判断同一个主机名的call的数量是否超过maxRequestsPerHost的
	3. 执行`promoteAndExecute()`来找到有执行资格的call，并启用线程池执行其中的run方法

readyAsyncCalls中存放的是准备执行，但是还没执行的call。有两类call会被放到这个队列，一类是刚enqueue的call；一类是超过了maxRequests和maxRequestsPerHost这两个值的限制的call

下面继续查看promoteAndExecute()，代码如下：

```
private fun promoteAndExecute(): Boolean {
    this.assertThreadDoesntHoldLock()
		// 用来保存找到的可执行的call的list
    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning: Boolean
    // 同步操作
    synchronized(this) {
      val i = readyAsyncCalls.iterator()
      // 循环遍历readyAsyncCalls中的call
      while (i.hasNext()) {
        val asyncCall = i.next()
			// 判断其是否超过maxRequests和maxRequestsPerHost的最大限制
        if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
        if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.

        i.remove()
        // 找到之后callsPerHost值+1
        asyncCall.callsPerHost.incrementAndGet()
        // 放入到可执行list
        executableCalls.add(asyncCall)
        // 放入到runningAsyncCalls的双端队列
        runningAsyncCalls.add(asyncCall)
      }
      // 这个方法源码为 runningAsyncCalls.size + runningSyncCalls.size，只要正在执行的队列中有call，就算是正在执行
      isRunning = runningCallsCount() > 0
    }
		// 遍历executableCalls可执行list中的call，并调用其executeOn的方法，传入的线程池是Dispatcher初始化的ExecutorService
    for (i in 0 until executableCalls.size) {
      val asyncCall = executableCalls[i]
      asyncCall.executeOn(executorService)
    }
    return isRunning
  }
```

可以看到`promoteAndExecute()`方法的实际工作：
	1. 遍历readyAsyncCalls这个双端队列，找到所有有资格执行的call，并放到executableCalls这个lisit中（所谓有执行资格的call要满足两个条件：第一是这个call是在readyAsyncCalls中的；第二是runningAsyncCalls的数量没有超过maxRequests限制，而且asyncCall.callsPerHost没有超过maxRequestsPerHost的限制）
	2. 遍历executableCalls这个list，并依次调用AsyncCall的executeOn(ExecutorService)方法

这里最终有回到了AsyncCall中，那么我们再回来看看`asyncCall.executeOn(executorService)`这个方法又做了什么，代码如下：

```
fun executeOn(executorService: ExecutorService) {
      client.dispatcher.assertThreadDoesntHoldLock()

      var success = false
      try {
      	// 调用了executorService的execute()方法，我们知道ExecutorService执行的是Runnable的run()方法，而且用自己管理的线程去执行
        executorService.execute(this)
        success = true
      } catch (e: RejectedExecutionException) {
      	// 这里是线程池抛出拒绝异常的时候的处理策略
        val ioException = InterruptedIOException("executor rejected")
        ioException.initCause(e)
        noMoreExchanges(ioException)
        // 直接回调我们的onFailure，这个callback就是我们在执行enqueue的时候，传入的自己实现的Callback
        responseCallback.onFailure(this@RealCall, ioException)
      } finally {
      	// 如果没有成功执行，那么在client.dispatcher.finished中的逻辑会继续调用promoteAndExecute()方法找到可执行的call，或者执行idleCallback.run()逻辑
        if (!success) {
          client.dispatcher.finished(this) // This call is no longer running!
        }
      }
    }
```

这段逻辑可以看到：
	1. 真正调用的还是ExecutorService的execute方法，进而会使用多线程来执行AsyncCall的run()方法，其实AsyncCall实现了Runnable接口
	2. 处理ExecutorService抛出拒绝异常的逻辑，直接回调我们自己实现的responseCallback.onFailure方法
	3. 在最后会判断这个call是否成功调用，如果没有成功调用的话，则会进而继续调用promoteAndExecute()去执行其他的可执行的call，或者执行idleCallback.run()逻辑

到此我们需要再跟进一步，那就是看下AsyncCall的run()方法中，又做了什么，代码如下：

```
override fun run() {
      threadName("OkHttp ${redactedUrl()}") {
        var signalledCallback = false
        timeout.enter()
        try {
        		// 最核心的逻辑，执行一个拦截器链条，来获取Response
          val response = getResponseWithInterceptorChain()
          signalledCallback = true
          // 回调我们实现的responseCallback.onResponse
          responseCallback.onResponse(this@RealCall, response)
        } catch (e: IOException) {
          if (signalledCallback) {
            // Do not signal the callback twice!
            Platform.get().log("Callback failure for ${toLoggableString()}", Platform.INFO, e)
          } else {
          	// IO异常，回调我们实现的responseCallback.onFailure
            responseCallback.onFailure(this@RealCall, e)
          }
        } catch (t: Throwable) {
        		// 关闭socket等收尾工作
          cancel()
          if (!signalledCallback) {
            val canceledException = IOException("canceled due to $t")
            canceledException.addSuppressed(t)
            // 回调我们实现的responseCallback.onFailure
            responseCallback.onFailure(this@RealCall, canceledException)
          }
          throw t
        } finally {
        	// 与线程池中finally中的调用一致
          client.dispatcher.finished(this)
        }
      }
    }
```

可以看到`run()`方法中的主要逻辑为：
1. getResponseWithInterceptorChain()来获取response，这个是OkHttp最核心的逻辑。获取成功之后会回调我们实现的responseCallback.onResponse
2. 如果出现异常会回调我们的responseCallback.onFailure，并做一些`cancel()`等收尾工作
3. 同时也会finishi中再次调用`client.dispatcher.finished()`进而调用`promoteAndExecute()`来继续之后的call的调用

到此先进行一下小小的总结
![okhttp](/Users/lixiangyue/Personal/blog/Blog/源码解析/media/okhttp.png)
整体执行流程如上图所示。最终还是会使用线程池来调用相关方法，但是核心方法是`getResponseWithInterceptorChain()`。

总结一下几个核心类，OkHttpClient的作用就是作为一个大管家，来进行各种参数的统一配置，比如配置各种超时的参数、设置自定义的interceptor（可以用来统一添加请求头）、添加EventListener来监听网络请求过程中每个阶段的数据等等。Request封装了url、method、body等参数。真正执行逻辑的是RealCall，RealCall中包含了最开始初始化的OkHttpClient和Request。

### OkHttpClient相关参数简介

直接上代码看注释吧,直解除参数名和类型

```
// 线程调度器，来统一的调度任务
dispatcher: Dispatcher = builder.dispatcher 
// 连接池，用来复用连接避免额外开销（比如在TCP的连接上完成一次请求之后，暂时不销毁，会先放入到连接池，下一个请求就可以直接复用请求池中的连接啦，可以省去TCP的握手过程，已经重新connection对象创建等）
connectionPool: ConnectionPool = builder.connectionPool
// 用户可以自定义拦截器，添加到这个list中，这个拦截器list中的interceptor会第一个被调用
interceptors: List<Interceptor> = builder.interceptors.toImmutableList()
// 用户可以自定义拦截器，添加到这个list中，这个拦截器list中的interceptor会在连接建立成功之后调用
networkInterceptors: List<Interceptor> = builder.networkInterceptors.toImmutableList()
// 用来创建eventListener的工厂类，会在OkHttpClient.newCall()方法调用的时候执行create方法创建EventListener，EventListener可以用来监听一次请求的各个阶段的信息（比如请求开始发起，连接开始发起，读写字节流开始等等）
eventListenerFactory: EventListener.Factory = builder.eventListenerFactory
// 连接失败后是否重试，在RetryAndFollowUpInterceptor和ExchangeFinder中会用到
retryOnConnectionFailure: Boolean = builder.retryOnConnectionFailure
// token失效时，可以使用Authenticator重新授权身份
authenticator: Authenticator = builder.authenticator
// 是否重定向，用于在服务器返回301/302等时，是否继续重试，在RetryAndFollowUpInterceptor有使用
followRedirects: Boolean = builder.followRedirects
// 在followRedirects为true的情况下，是否在协议切换的时候进行重定向（比如由http协议重定向到https协议），在RetryAndFollowUpInterceptor中有使用
followSslRedirects: Boolean = builder.followSslRedirects
// 用来保存cookie的，可以自己实现保存cookie的策略，默认为空实现
cookieJar: CookieJar = builder.cookieJar
// 用来实现缓存机制的
cache: Cache? = builder.cache
// 域名系统，顾名思义，是用来解析域名的，默认实现为DnsSystem，使用java.net包里的方法InetAddress.getAllByName(hostname).toList()使用操作系统的默认实现，解析域名。也可以使用自定义的DNS
dns: Dns = builder.dns
// 用来配置代理，我们可以选择使用代理服务器帮我们转发请求，而不是直接请求我们自己的服务器，一般我们都是直连我们服务器的。代理类型有三种（DIRECT、HTTP、SOCKS），默认为空
proxy: Proxy? = builder.proxy
// 代理选择器，当proxy为空时，就会使用这个选择器来找到可用的proxy，默认会找到类型为DIRECT的Proxy
proxySelector: ProxySelector =
      when {
        // Defer calls to ProxySelector.getDefault() because it can throw a SecurityException.
        builder.proxy != null -> NullProxySelector
        else -> builder.proxySelector ?: ProxySelector.getDefault() ?: NullProxySelector
      }
// 代理服务器的Authenticator，也是用于身份校验，刷新token
proxyAuthenticator: Authenticator =
      builder.proxyAuthenticator
// 我们建立连接时，本质会使用Socket来实现，SocketFactory就是用来创建Socket对象的
socketFactory: SocketFactory = builder.socketFactory
// 建立加密连接时，使用SSLSocketFactory创建用来构建加密连接的Socket
sslSocketFactoryOrNull: SSLSocketFactory?
sslSocketFactory: SSLSocketFactory
    get() = sslSocketFactoryOrNull ?: throw IllegalStateException("CLEARTEXT-only client")
// 用来校验证合法性的工具，查看证书是否在自己的根证书信任机构，证书的hash值等（证书格式为x509）
x509TrustManager: X509TrustManager?
// ConnectionSpec全称为Connection Specification，也就是连接的规格，即使用的对称加密和非对称加密的加密算法、Hash算法（RSA，AES，SHA256），使用的TLS版本（v1.1，v1.2，v1.3）等等。已经默认定义了三种不同的Cipher Suite了
connectionSpecs: List<ConnectionSpec> = builder.connectionSpecs
// 协议列表（HTTP_1_0，HTTP_1_1，SPDY_3 Http2的前身，HTTP_2）
protocols: List<Protocol> = builder.protocols
// 用来验证证书的hostname是否是要请求的那个hostname
hostnameVerifier: HostnameVerifier = builder.hostnameVerifier
// 可以在验证证书合法性的基础上，额外验证证书的其他信息（sha256）（作用之一就是可以防止证书签发机构腐败的情况，签发一个可以通过我们自己证书验证的真实的证书，而加了额外的sha256校验的话，就通过不了了。但是要慎用，因为如果自己的服务器证书换证书的话，之前版本的App就无法通过证书校验了）
certificatePinner: CertificatePinner
// 用来操作X509TrustManager的工具来验证证书的，是一个操作员的角色
certificateChainCleaner: CertificateChainCleaner?
// 超时时间的阀值相关的设置
callTimeoutMillis: Int = builder.callTimeout
connectTimeoutMillis: Int = builder.connectTimeout
readTimeoutMillis: Int = builder.readTimeout
writeTimeoutMillis: Int = builder.writeTimeout
```

上面对于OkHttpClient中各个参数的解析已经相当清晰了，这里主要说一下连接复用的问题。首先我们要先明白什么叫做Http的连接。我们知道Http的请求是需要建立在TCP的连接之上的，所谓TCP的连接过程，就是我们所熟知的TCP的三次握手(3-way-handshake)的过程。TCP是可靠的传输协议，但是是怎么保证可靠性的呢？我们可能会说是通过建立一个连接通道，但是这个连接本质上是什么呢？首先，TCP在传输数据的时候，会将数据分段，数据起始段的编号就需要在三次握手的过程中去确认。而为了保证可靠性传输，TCP会采用确认机制，而且两端会维护两个滑动窗口，防止双方发送数据太快或者太慢，之后对方返回了ACK相应的数据端的号之后，才能继续发之后的数据。所以TCP三次握手建立连接的过程，其实就是在确认双发送数据的起始编号，让双方维护保存一个相同的状态。TCP的具体内容，可以看下[Transmission Control Protocol](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)。而Http的一次请求是需要在TCP连接之上来进行的，在Http1.0的时候，每次请求就需要建立一次TCP连接，请求完成之后就会关闭连接，所以速度会很慢。而Http1.1进行了改进，每次请求完成之后，不会关闭TCP连接，而会keep-alive一段时间，如果之后再次有请求，就可以复用这个TCP连接了，省去了三次握手的过程，但是需要注意的是，只有请求完成之后的TCP连接才能复用。而http2.0则进一步引入了多路复用，使得一条TCP连接上，在一次请求结束之前，也可以在这个TCP连接上进行数据传输。关于Http的资料可以查看[Hypertext Transfer Protocol](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#History)，[Http/2](https://en.wikipedia.org/wiki/HTTP/2)

### 核心方法getResponseWithInterceptorChain()

直接上代码，这个方法大概做了三件事，第一件事是将每个interceptor放入到一个list中，第二件事是使用这个interceptor的list以及其他的参数构建一个RealInterceptorChain对象，第三件事是调用这个RealInterceptorChain对象的chain.proceed方法，来开启链式操作。具体是怎么开启链式操作的我们一会儿在画图说明，先看代码

```
internal fun getResponseWithInterceptorChain(): Response {
    // 1. 构建interceptor的list，每个interceptor的作用也之后细聊
    val interceptors = mutableListOf<Interceptor>()
    interceptors += client.interceptors
    interceptors += RetryAndFollowUpInterceptor(client)
    interceptors += BridgeInterceptor(client.cookieJar)
    interceptors += CacheInterceptor(client.cache)
    interceptors += ConnectInterceptor
    if (!forWebSocket) {
      interceptors += client.networkInterceptors
    }
    interceptors += CallServerInterceptor(forWebSocket)

		// 2. 使用interceptors来构建RealInterceptorChain对象，注意一下，这里的index传入的是0
    val chain = RealInterceptorChain(
        call = this,
        interceptors = interceptors,
        index = 0,
        exchange = null,
        request = originalRequest,
        connectTimeoutMillis = client.connectTimeoutMillis,
        readTimeoutMillis = client.readTimeoutMillis,
        writeTimeoutMillis = client.writeTimeoutMillis
    )

    var calledNoMoreExchanges = false
    try {
    	// 3. 调用chain.proceed来开启链式工作
      val response = chain.proceed(originalRequest)
      if (isCanceled()) {
        response.closeQuietly()
        throw IOException("Canceled")
      }
      return response
    } catch (e: IOException) {
      calledNoMoreExchanges = true
      throw noMoreExchanges(e) as Throwable
    } finally {
      if (!calledNoMoreExchanges) {
        noMoreExchanges(null)
      }
    }
  }
```

那重点就是chain.proceed(originalRequest)的代码了，直接看代码如下,可以看到，这里首先会复制一下RealInterceptorChain对象，并将当前的index+1传入（初次的时候传入的index值为0），然后通过index从interceptor列表中找到当前的interceptor，并执行interceptor的interceptor.intercept(next)方法，next就是新copy的RealInterceptorChain对象：

```
override fun proceed(request: Request): Response {
    check(index < interceptors.size)

    calls++

    if (exchange != null) {
      check(exchange.finder.sameHostAndPort(request.url)) {
        "network interceptor ${interceptors[index - 1]} must retain the same host and port"
      }
      check(calls == 1) {
        "network interceptor ${interceptors[index - 1]} must call proceed() exactly once"
      }
    }

    // Call the next interceptor in the chain.
    val next = copy(index = index + 1, request = request)
    val interceptor = interceptors[index]

    @Suppress("USELESS_ELVIS")
    val response = interceptor.intercept(next) ?: throw NullPointerException(
        "interceptor $interceptor returned null")

    if (exchange != null) {
      check(index + 1 >= interceptors.size || next.calls == 1) {
        "network interceptor $interceptor must call proceed() exactly once"
      }
    }

    check(response.body != null) { "interceptor $interceptor returned a response with no body" }

    return response
  }
```

这里我们点进去Interceptor的intercept方法可以看到，这个又是接口方法，但是我们知道interceptor对象是从interceptor的list中拿到的，所以我们直接看列表中第一个interceptor的方法。由于我们没有传入我们自己实现的Interceptor对象，所以我们直接看RetryAndFollowUpInterceptor的intercept方法，这里只看关键的一句：

```
override fun intercept(chain: Interceptor.Chain): Response {
    
    while (true) {
      // 省略 ... 
        try {
        // 重点就是这句 realChain.proceed，这里会继续调用RealInterceptorChain的proceed方法，但是由于当前的RealInterceptorChain的index为1，所以就会继续interceptor的list中，位置为2的Interceptor进行链条式的工作
          response = realChain.proceed(request)
          newExchangeFinder = true
        } 
      // 省略 ... 
  }
```

到这里，我们先总结一下这个链条工作起来的流程，假设只有两个interceptor，如图：

![interceptor-chain](/Users/lixiangyue/Personal/blog/Blog/源码解析/media/interceptor-chain.jpg)

在interceptor的链条中，每个拦截器都会先执行自己的前置工作，然后调用chain.proceed向前推动链条，之后等到下一个拦截器返回结果，待下一个拦截器返回结果之后，在执行自己的后置工作。就这样一步步的完成最终的response返回。

下面这张是抽象的链条工作模型
![chain](/Users/lixiangyue/Personal/blog/Blog/源码解析/media/chain.jpg)


至此，我们大概有一个OkHttp的大框架了。首先OkHttpClient会使用request初始化一个RealCall，然后RealCall会调用enqueue方法，enqueue内部实际执行的是Dispatcher的enqueue（这里实际传入了AsyncCall，他是RealCall的内部类），这里就使用了线程池来执行RealCall的内部类，AsyncCall的runnable方法。而在AsyncCall的run方法中，会执行getResponseWithInterceptorChain()来触发拦截器链条的执行，每一个拦截器各司其职，会进行相应的前置工作，推进链条，后置工作。进而得到最终的response，然后执行我们的callback方法。

接下来就来分析一个各个拦截器的作用，在查看每个拦截器的作用之前，我们首先要把握一下每个拦截器的基本结构。其实每个拦截器核心的方法就是intercept，在intercept方法内部又分为三部分，realChain.proceed(request)之前的代码为前置工作，realChain.proceed(request)为向下传递触发下一个拦截器执行，realChain.proceed(request)之后的代码为后置工作。那么我们遵循这个原则，来首先看下RetryAndFollowUpInterceptor

### RetryAndFollowUpInterceptor

关于RetryAndFollowUpInterceptor，从名字也可以看出，是用来进行重试和重定向的，当然他的功能也的确如此。重试的情况比如：建立连接失败，建立ssl连接失败或者请求超时等等，都会进行重试，所以里面会用到while(true)。而如果结果正确返回，但是状态码是301或者302等等，则需要重定向。

```
override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    var request = chain.request
    val call = realChain.call
    var followUpCount = 0
    var priorResponse: Response? = null
    var newExchangeFinder = true
    var recoveredFailures = listOf<IOException>()
    // 这里使用while(true)，要么成功请求返回response，要么会重试，要么就抛出异常结束
    while (true) {
    	// 前置工作：这里是初始化一个Exchange的寻找器
      call.enterNetworkInterceptorExchange(request, newExchangeFinder)

      var response: Response
      var closeActiveExchange = true
      try {
        if (call.isCanceled()) {
          throw IOException("Canceled")
        }

        try {
        	  // 中置工作：向下接力，触发下一个interceptor的执行
          response = realChain.proceed(request)
          // 如果成功返回response，证明没有出现异常，则下次需要重新初始化ExchangeFinder
          newExchangeFinder = true
        } catch (e: RouteException) {
          // The attempt to connect via a route failed. The request will not have been sent.
          // 出现在某条路由走不通的异常，先判断是否是可恢复的异常（比如设置了retryOnConnectionFailure为false，则直接不重试了，或者是证书异常，ProtocolException等，也不重试），具体细节可以查看代码
          if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
            throw e.firstConnectException.withSuppressed(recoveredFailures)
          } else {
            recoveredFailures += e.firstConnectException
          }
          // 如果可恢复，则直接continue
          newExchangeFinder = false
          continue
        } catch (e: IOException) {
          // An attempt to communicate with a server failed. The request may have been sent.
          // 同样判断是否可恢复
          if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
            throw e.withSuppressed(recoveredFailures)
          } else {
            recoveredFailures += e
          }
          // 如果可恢复，则直接continue
          newExchangeFinder = false
          continue
        }

        // Attach the prior response if it exists. Such responses never have a body.
        if (priorResponse != null) {
          response = response.newBuilder()
              .priorResponse(priorResponse.newBuilder()
                  .body(null)
                  .build())
              .build()
        }

        val exchange = call.interceptorScopedExchange
        // 在followUpRequest方法中，会判断如果服务端返回的code状态码为301，302等等需要重定向的状态码，则会build一个新的重定向的request
        val followUp = followUpRequest(response, exchange)
			// 如果不是重定向的状态码，则直接返回response
        if (followUp == null) {
          if (exchange != null && exchange.isDuplex) {
            call.timeoutEarlyExit()
          }
          closeActiveExchange = false
          return response
        }

        val followUpBody = followUp.body
        if (followUpBody != null && followUpBody.isOneShot()) {
          closeActiveExchange = false
          return response
        }

        response.body?.closeQuietly()

        if (++followUpCount > MAX_FOLLOW_UPS) {
          throw ProtocolException("Too many follow-up requests: $followUpCount")
        }
			// 如果没有返回response，则证明需要重定向，则直接将build的新的Request类型的变量followUp赋值给request，进而进行下一轮的重试逻辑
        request = followUp
        priorResponse = response
      } finally {
        call.exitNetworkInterceptorExchange(closeActiveExchange)
      }
    }
  }
```

从代码的逻辑可以看出，在此方法中，主要做了三件事：
1. 初始化一个ExchangeFinder，为之后的拦截器做准备。关于ExchangeFinder需要解释一下，他的作用就是来寻找一个Exchange，其实就是来寻找一个可用的连接的。一个Exchange就代表是一次数据请求和一次数据返回的过程，里面会保存这次请求的端口了，ip地址了，协议了，使用的TLS版本了等等信息。
2. 调用chain.proceed向下推进链条工作
3. 如果出现异常或者需要重定向的话，则进行重试

工作流程图如下

![retry-and-follow-up-interceptor](/Users/lixiangyue/Personal/blog/Blog/源码解析/media/retry-and-follow-up-interceptor.jpg)

接下来我们继续看下一个interceptor

### BridgeInterceptor

看名字，这是一个桥拦截器，从代码注释来看`Bridges from application code to network code`，就是用来将我们应用程序中的代码转换为真正要网络请求的代码。主要功能是用来帮我们填充一个完整的Request的，一个完整的Http请求包括请求行，请求头，请求体，而且请求头中会有host、content-type等等，但是我们在使用OkHttp在创建Request的时候，并没有传入这些东西，其实就是BridgeInterceptor在帮我们将这些信息填充完整的。同样代码分为三部分，chain.proceed(requestBuilder.build())之前为前置工作，chain.proceed(requestBuilder.build())为中置工作，chain.proceed(requestBuilder.build())之后为后置工作。那么我们先看前置工作的代码：

```
 override fun intercept(chain: Interceptor.Chain): Response {
    val userRequest = chain.request()
    val requestBuilder = userRequest.newBuilder()

    val body = userRequest.body
    if (body != null) {
      val contentType = body.contentType()
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString())
      }

      val contentLength = body.contentLength()
      if (contentLength != -1L) {
        requestBuilder.header("Content-Length", contentLength.toString())
        requestBuilder.removeHeader("Transfer-Encoding")
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked")
        requestBuilder.removeHeader("Content-Length")
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", userRequest.url.toHostHeader())
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive")
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    var transparentGzip = false
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true
      requestBuilder.header("Accept-Encoding", "gzip")
    }

    val cookies = cookieJar.loadForRequest(userRequest.url)
    if (cookies.isNotEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies))
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", userAgent)
    }

```

可以看到，前置工作就是在帮我们添加各种头信息，包括Content-Type、Content-Length、Host等等，同时还会自动帮我们添加Accept-Encoding为gzip，支持gzip压缩传输（后置工作中会自动帮我们完成解压工作）。中置工作就是调用chain.proceed(requestBuilder.build())来继续推进链式工作的进行，那么我们继续看后置工作的代码：

```
cookieJar.receiveHeaders(userRequest.url, networkResponse.headers)

    val responseBuilder = networkResponse.newBuilder()
        .request(userRequest)

    if (transparentGzip &&
        "gzip".equals(networkResponse.header("Content-Encoding"), ignoreCase = true) &&
        networkResponse.promisesBody()) {
      val responseBody = networkResponse.body
      if (responseBody != null) {
      // 自动gzip解压
        val gzipSource = GzipSource(responseBody.source())
        val strippedHeaders = networkResponse.headers.newBuilder()
            .removeAll("Content-Encoding")
            .removeAll("Content-Length")
            .build()
        responseBuilder.headers(strippedHeaders)
        val contentType = networkResponse.header("Content-Type")
        responseBuilder.body(RealResponseBody(contentType, -1L, gzipSource.buffer()))
      }
    }
```

可以看到，会将返回的结果自动帮我们使用gzip解压。可以看到`BridgeInterceptor`的功能比较简单，那么继续查看下一个拦截器

### CacheInterceptor

这个拦截器看名字就知道，是一个缓存拦截器，功能就是根据缓存策略，然后来判断是否从本地拿数据，而不需要从从服务器拿，如果本地有数据，则直接返回，而不需要再向下推进拦截器链式工作的操作。如果需要访问网络，则需要继续推进拦截器链的工作。

```
override fun intercept(chain: Interceptor.Chain): Response {
    val call = chain.call()
    val cacheCandidate = cache?.get(chain.request())

    val now = System.currentTimeMillis()

    val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
    val networkRequest = strategy.networkRequest
    val cacheResponse = strategy.cacheResponse

    cache?.trackResponse(strategy)
    val listener = (call as? RealCall)?.eventListener ?: EventListener.NONE

    if (cacheCandidate != null && cacheResponse == null) {
      // The cache candidate wasn't applicable. Close it.
      cacheCandidate.body?.closeQuietly()
    }

    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
      return Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(HTTP_GATEWAY_TIMEOUT)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build().also {
            listener.satisfactionFailure(call, it)
          }
    }

		// 如果不需要网络，而在cache中找到，则直接返回
    // If we don't need the network, we're done.
    if (networkRequest == null) {
      return cacheResponse!!.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build().also {
            listener.cacheHit(call, it)
          }
    }

    if (cacheResponse != null) {
      listener.cacheConditionalHit(call, cacheResponse)
    } else if (cache != null) {
      listener.cacheMiss(call)
    }

    var networkResponse: Response? = null
    try {
    // 如果需要网络操作，则继续推链
      networkResponse = chain.proceed(networkRequest)
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        cacheCandidate.body?.closeQuietly()
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
    	// 如果状态码为HTTP Status-Code 304: Not Modified，则直接将cache中的response返回即可
      if (networkResponse?.code == HTTP_NOT_MODIFIED) {
        val response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers, networkResponse.headers))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis)
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis)
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build()

        networkResponse.body!!.close()

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache!!.trackConditionalCacheHit()
        // 更新缓存
        cache.update(cacheResponse, response)
        return response.also {
          listener.cacheHit(call, it)
        }
      } else {
        cacheResponse.body?.closeQuietly()
      }
    }
```

CacheInterceptor对于OkHttp的整体流程其实没有太大的影响，属于一个独立的流程，所以对于OkHttp的流程来说，没必要深入分析，之后有时间，我会再对其单独分析一下。那么接下来，我们来看一下最重要的，真正建立连接的拦截器

### ConnectInterceptor

先贴一下代码吧：

```
override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val exchange = realChain.call.initExchange(chain)
    val connectedChain = realChain.copy(exchange = exchange)
    return connectedChain.proceed(realChain.request)
  }
```

乍一看，好像挺简单的，从结构上看只有前置工作和中置工作就返回了，而没有后置工作。从代码量上看，就四行代码，但是，其实里面蕴含了很多逻辑，我们逐个来看下。从realChain.call.initExchange开始看起，代码入下：

```
/** Finds a new or pooled connection to carry a forthcoming request and response. */
  internal fun initExchange(chain: RealInterceptorChain): Exchange {
    synchronized(this) {
      check(expectMoreExchanges) { "released" }
      check(!responseBodyOpen)
      check(!requestBodyOpen)
    }

    val exchangeFinder = this.exchangeFinder!!
    // 通过exchangeFinder.find来创建一个codec，
    val codec = exchangeFinder.find(client, chain)
    // 根据拿到的exchangeFinder，codec来创建一个Exchange，Exchange是什么上面已经解释过了
    val result = Exchange(this, eventListener, exchangeFinder, codec)
    this.interceptorScopedExchange = result
    this.exchange = result
    synchronized(this) {
      this.requestBodyOpen = true
      this.responseBodyOpen = true
    }

    if (canceled) throw IOException("Canceled")
    return result
  }
```

其实这个方法的注释也写的很清楚，要寻找一个新的或者连接池里的连接，用于传输即将发起的请求或者响应。代码中所做的事如下：
	1. 通过exchangeFinder.find来创建一个codec，exchangeFinder其实已经在第一个拦截器RetryAndFollowUpInterceptor就已经创建好了，然后就是codec的就是，所谓codec全称其实是coder and decoder，意思是一个编码解码器。子类有两个，一个是http1的，一个是http2的。其实就是一个真正的根据http1的协议或者http2的协议来将request中的内容写入流，或者将流中的数据读出到response。
	2. 根据拿到的exchangeFinder，codec来创建一个Exchange，Exchange是什么上面已经解释过了，一个Exchange就是一次数据交换，也即是一个请求，一个响应的过程。
继续顺着往下，我们看下exchangeFinder.find又做了什么，代码如下：

```
fun find(
    client: OkHttpClient,
    chain: RealInterceptorChain
  ): ExchangeCodec {
    try {
    // 找到一个健康的连接
      val resultConnection = findHealthyConnection(
          connectTimeout = chain.connectTimeoutMillis,
          readTimeout = chain.readTimeoutMillis,
          writeTimeout = chain.writeTimeoutMillis,
          pingIntervalMillis = client.pingIntervalMillis,
          connectionRetryEnabled = client.retryOnConnectionFailure,
          doExtensiveHealthChecks = chain.request.method != "GET"
      )
      // 通过这个健康的连接来初始化一个codec
      return resultConnection.newCodec(client, chain)
    } catch (e: RouteException) {
      trackFailure(e.lastConnectException)
      throw e
    } catch (e: IOException) {
      trackFailure(e)
      throw RouteException(e)
    }
  }
```

这个代码中所做的两件事：
	1. 找到一个健康的连接，所谓一个健康的连接，其实就是一个可用的连接，而且该连接还没有被关闭
	2. 通过这个健康的连接来初始化一个codec
还是需要我们继续跟进，findHealthyConnection

```
private fun findHealthyConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean,
    doExtensiveHealthChecks: Boolean
  ): RealConnection {
    while (true) {
    	// 首先会寻找一个连接
      val candidate = findConnection(
          connectTimeout = connectTimeout,
          readTimeout = readTimeout,
          writeTimeout = writeTimeout,
          pingIntervalMillis = pingIntervalMillis,
          connectionRetryEnabled = connectionRetryEnabled
      )
		// 之后来判断其是否事健康的
      // Confirm that the connection is good.
      if (candidate.isHealthy(doExtensiveHealthChecks)) {
        return candidate
      }

      // If it isn't, take it out of the pool.
      candidate.noNewExchanges()

      // Make sure we have some routes left to try. One example where we may exhaust all the routes
      // would happen if we made a new connection and it immediately is detected as unhealthy.
      if (nextRouteToTry != null) continue

      val routesLeft = routeSelection?.hasNext() ?: true
      if (routesLeft) continue

      val routesSelectionLeft = routeSelector?.hasNext() ?: true
      if (routesSelectionLeft) continue

      throw IOException("exhausted all routes")
    }
  }
```

这段代码主要做了两件事：
	1. 找到一个连接
	2. 判断其是否事健康的
到这里我们还是需要再次跟进，看下findConnection方法，代码如下：
这里算是一个核心的位置了，需要详细解释一下，先看注释

```
  private fun findConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean
  ): RealConnection {
  		// 如果我们在连接开始之前调用了call.cancel，那么我们需要直接抛出异常，而不进行连接操作
    if (call.isCanceled()) throw IOException("Canceled")

		// 这里发生在连接超时、重定向、连接失败时的重试的情况下，这时callConnection不为空，会进入之后的判断逻辑
    // Attempt to reuse the connection from the call.
    val callConnection = call.connection // This may be mutated by releaseConnectionNoEvents()!
    if (callConnection != null) {
      var toClose: Socket? = null
      synchronized(callConnection) {
      	// 首先会判断端口号，主机名等是否一致，即查看这个连接是否可以直接使用（比如http重定向到https时，由于端口号变了，就无法使用该连接了）
        if (callConnection.noNewExchanges || !sameHostAndPort(callConnection.route().address.url)) {
          toClose = call.releaseConnectionNoEvents()
        }
      }
			// 如果这个连接经过上面的判断，没有被释放，证明可以直接使用，那么就会直接返回，而不需要后续操作
      // If the call's connection wasn't released, reuse it. We don't call connectionAcquired() here
      // because we already acquired it.
      if (call.connection != null) {
        check(toClose == null)
        return callConnection
      }

      // The call's connection was released.
      toClose?.closeQuietly()
      eventListener.connectionReleased(call, callConnection)
    }

    // We need a new connection. Give it fresh stats.
    refusedStreamCount = 0
    connectionShutdownCount = 0
    otherFailureCount = 0
		// 如果没有call.connection == null，也即是我们第一次发起请求时发生的情况，那么我们会先尝试从连接池里拿可用的连接，这次只拿非多路复用的连接，而且不能连接合并的连接，拿到则返回
    // Attempt to get a connection from the pool.
    if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
      val result = call.connection!!
      eventListener.connectionAcquired(call, result)
      return result
    }

    // Nothing in the pool. Figure out what route we'll try next.
    val routes: List<Route>?
    val route: Route
    // 这里是进行route的尝试操作，会先从RouteSelector找到routeSelection，然后在routeSelection中尝试每一个Route
    if (nextRouteToTry != null) {
      // Use a route from a preceding coalesced connection.
      routes = null
      route = nextRouteToTry!!
      nextRouteToTry = null
    } else if (routeSelection != null && routeSelection!!.hasNext()) {
      // Use a route from an existing route selection.
      routes = null
      route = routeSelection!!.next()
    } else {
      // Compute a new route selection. This is a blocking operation!
      var localRouteSelector = routeSelector
      if (localRouteSelector == null) {
        localRouteSelector = RouteSelector(address, call.client.routeDatabase, call, eventListener)
        this.routeSelector = localRouteSelector
      }
      val localRouteSelection = localRouteSelector.next()
      routeSelection = localRouteSelection
      routes = localRouteSelection.routes

      if (call.isCanceled()) throw IOException("Canceled")

      // Now that we have a set of IP addresses, make another attempt at getting a connection from
      // the pool. We have a better chance of matching thanks to connection coalescing.
      // 这里会继续尝试从连接池里拿可用连接，这里传入了routes参数，只有有route参数的时候，才能判断出，是否能够进行连接合并。拿到则直接返回
      if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
        val result = call.connection!!
        eventListener.connectionAcquired(call, result)
        return result
      }

      route = localRouteSelection.next()
    }
		// 如果两次尝试从连接池里拿都没有拿到，就自己创建一个连接对象
    // Connect. Tell the call about the connecting call so async cancels work.
    val newConnection = RealConnection(connectionPool, route)
    call.connectionToCancel = newConnection
    try {
    // 真正的去自己建立TCP连接
      newConnection.connect(
          connectTimeout,
          readTimeout,
          writeTimeout,
          pingIntervalMillis,
          connectionRetryEnabled,
          call,
          eventListener
      )
    } finally {
      call.connectionToCancel = null
    }
    call.client.routeDatabase.connected(newConnection.route())

    // If we raced another call connecting to this host, coalesce the connections. This makes for 3
    // different lookups in the connection pool!
    // 在连接建立之后，再次尝试从连接池里拿一下，这次最后一个参数传入的是true，只看允许多路复用的
    if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
    	// 如果找到了，就把当前的这个连接关闭，而使用连接池里拿到的连接
      val result = call.connection!!
      nextRouteToTry = route
      newConnection.socket().closeQuietly()
      eventListener.connectionAcquired(call, result)
      return result
    }
	// 如果没有从连接池中拿到连接，其新的连接放入到连接池中
    synchronized(newConnection) {
      connectionPool.put(newConnection)
      call.acquireConnectionNoEvents(newConnection)
    }

    eventListener.connectionAcquired(call, newConnection)
    return newConnection
  }

```

这段代码逻辑如下：
1. 首先，如果是第一次发起请求，call.connection肯定是空的，所以会执行`if(connectionPool.callAcquirePooledConnection(address, call, null, false))`代码，尝试从连接池里拿到一个可用的连接，注意此时传入的末尾的两个参数，一个是null，一个是false。这里表示我们只看连接池里是否有跟当前的call匹配的http1的连接。关于`callAcquirePooledConnection()`这个方法的逻辑，下面详解。
2. 如果第一次从连接池中拿连接的逻辑没有拿到可用的连接，那么我们就首先来找到我们要尝试的`routes: List<Route>?`，和要用来建立连接的`route: Route`，这两个参数是为了接下来再次从连接池里拿连接和自己创建连接做准备的。这里用到了`RouteSelector`、`RouteSelector.Selection`、`routes: List<Route>`、`route: Route`，这些概念也在下面详细解析。
3. 之后会进行第二次尝试从连接池里拿连接，这次会传入`routes: List<Route>`参数，调用`if (connectionPool.callAcquirePooledConnection(address, call, routes, false))`，当传入routes之后就可以判断是否可以进行http2的连接合并了，所谓连接合并也会根据下面的图具体分析。
4. 如果第二次连接池里尝试的逻辑还拿不到可用的连接，那么我们就会自己初始化一个连接对象，并使用这个连接对象的`newConnection.connect()`方法来进行真正的连接，在这个方法里，其实就是使用java的socket来发起TCP连接和TLS连接。
5. 在连接建立之后，再次尝试从连接池里拿一下，这次最后一个参数传入的是true，只看允许多路复用的，这里主要是考虑到一种情况，如果同时发起多个http2的连接的话，当第一个请求连接成功后，之后的多个连接是可以进行连接合并的，这时如果能从连接池中拿到该连接的话，则直接进行连接合并，并将连接返回。同时将刚刚建立的连接关闭。这样就可以节省资源。
6. 如果前面那次从连接池那连接的操作也没有拿到可用连接的话，就将新的连接放入到连接池，并最终返回。
7. 如果发生了重试操作的话，则会再次进入该方法，而此时最开始的call.connection就不是空了，而且判断这个连接是否符合可用的条件，如果符合，就直接返回使用。

至此，该方法逻辑就分析完了，但是有几个坑，接下来填一下。首先是`callAcquirePooledConnection()`做了什么，我们看下代码：

```
fun callAcquirePooledConnection(
    address: Address,
    call: RealCall,
    routes: List<Route>?,
    requireMultiplexed: Boolean
  ): Boolean {
    for (connection in connections) {
      synchronized(connection) {
      	// 是否需要多路复用
        if (requireMultiplexed && !connection.isMultiplexed) return@synchronized
        // 连接是否可用，该方法逻辑见下面代码
        if (!connection.isEligible(address, routes)) return@synchronized
        // 如果都没问题，则直接拿来用
        call.acquireConnectionNoEvents(connection)
        return true
      }
    }
    return false
  }
  
  internal fun isEligible(address: Address, routes: List<Route>?): Boolean {
    assertThreadHoldsLock()
		// 首先判断这个连接上的call的数量是否超限，http1.1的协议要求一个连接上同时只能有一个请求，http2才允许一个连接上同时有多个请求
    // If this connection is not accepting new exchanges, we're done.
    if (calls.size >= allocationLimit || noNewExchanges) return false
		// 这里是具体判断协议、dns、proxy、port等等是否一致，一致才可以复用
    // If the non-host fields of the address don't overlap, we're done.
    if (!this.route.address.equalsNonHost(address)) return false

    // If the host exactly matches, we're done: this connection can carry the address.
    // 这里是在上面比对一直的基础上，如果host也一致，则表明该连接可以复用
    if (address.url.host == this.route().address.url.host) {
      return true // This connection is a perfect match.
    }

    // At this point we don't have a hostname match. But we still be able to carry the request if
    // our connection coalescing requirements are met. See also:
    // https://hpbn.co/optimizing-application-delivery/#eliminate-domain-sharding
    // https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/
	// 这里的逻辑要求一定得是http2的，因为http2才支持connection coalescing，也就是连接合并
    // 1. This connection must be HTTP/2.
    if (http2Connection == null) return false
		// 连接合并要求只要ip地址一致，就可以进行连接合并
    // 2. The routes must share an IP address.
    if (routes == null || !routeMatchesAny(routes)) return false
		// 但是也要判断另一种情况，就是如果一个ip下挂着多个虚拟主机，这些虚拟主机虽然也是ip地址一致，但是网站证书是不一样的，只有网站证书一致，才是真的可以连接合并
    // 3. This connection's server certificate's must cover the new host.
    if (address.hostnameVerifier !== OkHostnameVerifier) return false
    if (!supportsUrl(address.url)) return false

    // 4. Certificate pinning must match the host.
    try {
    	// 最后在进一步验证一下CertificatePinner
      address.certificatePinner!!.check(address.url.host, handshake()!!.peerCertificates)
    } catch (_: SSLPeerUnverifiedException) {
      return false
    }
		// 如果校验都通过了，则证明可以进行连接合并，那么证明该连接可用
    return true // The caller's address can be carried by this connection.
  }
```

以上逻辑如下：
	1. 首先会判断http1的可用的连接，就是通过对比host、port、proxy，TLS协议，加密套件等等信息是否一致，一致则可用
	2. 对于http2的连接，还可以进行连接合并，如果ip地址一样，其他的证书也一样的话，则可以进行连接合并，同样也符合可用要求

解释一下`RouteSelector`、`RouteSelector.Selection`、`routes: List<Route>`、`route: Route`、`Address`的概念，如图所示：

![routeselector](/Users/lixiangyue/Personal/blog/Blog/源码解析/media/routeselector.jpg)

![connection-reusing](/Users/lixiangyue/Personal/blog/Blog/源码解析/media/connection-reusing.jpg)

我们知道，我们在请求一个url的时候，主要由三部分组成，ip地址，端口号，还有代理方式（分为直连或者使用代理服务器连接）。ip地址可用由域名解析，一个域名可以解析出多个ip，而端口号则在我们访问url的时候就定下来了，比如http端口为80，https为443等。所以Okhttp则对这些数据进行了封装，首先Address类中，封装了host、port、DNS、HostnameVerifier等等一些列的信息，用来描述连接地址。而Route则是一个路由，用来描述我们访问一个url的时候做经过的路线是什么样的，是直接连到服务器呢，还是要经过一个代理服务器进行转发呢，Route中封装了Address类。而一个RouteSelector.Selection封装了一个路由的list，但是他们都是属于同一种代理方式的，而一个RouteSelector则封装了多个RouteSelector.Selection，所以在进行可用Route的搜索时，则会先从RouteSelector找到RouteSelector.Selection，再从RouteSelector.Selection中找到Route。具体的结构则如上图所示，并且下面那张图也可以看出多条路径，也就是多条Route。

接下来解释连接合并：
连接合并是Http2的概念，因为多个域名是可以配置到同一个IP地址上的，所以当我们虽然域名不一样，但是要访问的IP地址一样时，在http2的时候，就可以进行连接合并，使用同一个连接进行传输。但是有一个特殊情况，就是虚拟主机的情况，如图：

![virtual-host](/Users/lixiangyue/Personal/blog/Blog/源码解析/media/virtual-host.jpg)

当有虚拟主机的话，虽然也是不同域名，但是是同一个IP，但是这个时候Route明显不同，所以这个时候，应该考虑再校验一下网站的证书是否一致，也即是我们在isEligible()方法中的逻辑。

至此我们就拿到了建立好的TCP连接了，当然如果是Https请求的话，是建立好了TLS连接，并且有响应的socket来让我们操作写入和读出流。
那么来看下最后一个拦截器

### CallServerInterceptor

这个拦截器是整个链条的最后一环，所以直接用来发送数据和读出数据了，具体的逻辑就是用初始化好的Exchange对象来exchange.writeRequestHeaders(request)，exchange.createRequestBody(request, true)等等，将request真正的写入到流中，并在返回流中读出数据，而在Exchange的方法内部，真正调用的是Codec对象，例如Http1ExchangeCodec的一个方法片段，就是使用Okio来真正的操作流数据。

```
  fun writeRequest(headers: Headers, requestLine: String) {
    check(state == STATE_IDLE) { "state: $state" }
    sink.writeUtf8(requestLine).writeUtf8("\r\n")
    for (i in 0 until headers.size) {
      sink.writeUtf8(headers.name(i))
          .writeUtf8(": ")
          .writeUtf8(headers.value(i))
          .writeUtf8("\r\n")
    }
    sink.writeUtf8("\r\n")
    state = STATE_OPEN_REQUEST_BODY
  }
```

最后把代码贴一下，加上注释：

```
override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val exchange = realChain.exchange!!
    val request = realChain.request
    val requestBody = request.body
    val sentRequestMillis = System.currentTimeMillis()
		// 将头信息写入流
    exchange.writeRequestHeaders(request)

    var invokeStartEvent = true
    var responseBuilder: Response.Builder? = null
    if (HttpMethod.permitsRequestBody(request.method) && requestBody != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equals(request.header("Expect"), ignoreCase = true)) {
        exchange.flushRequest()
        responseBuilder = exchange.readResponseHeaders(expectContinue = true)
        exchange.responseHeadersStart()
        invokeStartEvent = false
      }
      if (responseBuilder == null) {
        if (requestBody.isDuplex()) {
          // Prepare a duplex body so that the application can send a request body later.
          exchange.flushRequest()
          // 写入请求体
          val bufferedRequestBody = exchange.createRequestBody(request, true).buffer()
          requestBody.writeTo(bufferedRequestBody)
        } else {
          // Write the request body if the "Expect: 100-continue" expectation was met.
          val bufferedRequestBody = exchange.createRequestBody(request, false).buffer()
          requestBody.writeTo(bufferedRequestBody)
          bufferedRequestBody.close()
        }
      } else {
        exchange.noRequestBody()
        if (!exchange.connection.isMultiplexed) {
          // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
          // from being reused. Otherwise we're still obligated to transmit the request body to
          // leave the connection in a consistent state.
          exchange.noNewExchangesOnConnection()
        }
      }
    } else {
      exchange.noRequestBody()
    }

    if (requestBody == null || !requestBody.isDuplex()) {
      exchange.finishRequest()
    }
    if (responseBuilder == null) {
      responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
      if (invokeStartEvent) {
        exchange.responseHeadersStart()
        invokeStartEvent = false
      }
    }
    var response = responseBuilder
        .request(request)
        .handshake(exchange.connection.handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build()
    var code = response.code
    if (code == 100) {
      // Server sent a 100-continue even though we did not request one. Try again to read the actual
      // response status.
      responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
      if (invokeStartEvent) {
        exchange.responseHeadersStart()
      }
      response = responseBuilder
          .request(request)
          .handshake(exchange.connection.handshake())
          .sentRequestAtMillis(sentRequestMillis)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build()
      code = response.code
    }

    exchange.responseHeadersEnd(response)

    response = if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response.newBuilder()
          .body(EMPTY_RESPONSE)
          .build()
    } else {
      response.newBuilder()
          .body(exchange.openResponseBody(response))
          .build()
    }
    if ("close".equals(response.request.header("Connection"), ignoreCase = true) ||
        "close".equals(response.header("Connection"), ignoreCase = true)) {
      exchange.noNewExchangesOnConnection()
    }
    if ((code == 204 || code == 205) && response.body?.contentLength() ?: -1L > 0L) {
      throw ProtocolException(
          "HTTP $code had non-zero Content-Length: ${response.body?.contentLength()}")
    }
    return response
  }
```

至此从我们调用RealCall.enqueue()开始，一直到最终流的写入读出，就完成了。

	


