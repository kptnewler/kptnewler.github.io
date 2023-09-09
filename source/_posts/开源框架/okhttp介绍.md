---
title: okhttp介绍
date: 2023-09-09 15:38:16
tags:
categories:
---

<meta name="referrer" content="no-referrer" />
# OkHttp源码分析

OkHttp是一个高效的客户端 Http 请求框架，OkHttp 是对 HTTP 协议的实现，在Android客户端开发被广泛使用。

>支持Http1、Http2、Quic以及WebSocket
>
>连接池复用底层TCP(Socket)，减少请求延时
>
>无缝的支持GZIP减少数据流量
>
>缓存响应数据减少重复的网络请求
>
>请求失败自动重试主机的其他ip，自动重定向

学习OKhttp源码就要先看全局，然后根据使用需要再研究细节，体会其精妙之处。
了解它的原理，并加以实践，可以实现一个简单的HTTP请求客户端。



## OkHttp 的使用

OkHttpClient是 HTTP 协议中的**请求方**，使用 HTTP 协议获取网络上的各种资源，简单称为客户端，角色和浏览器一样。

OkHttpClient 中包含了各种组件，通过组合使用，实现HTTP请求。

Request 是OkHttp 的请求实体，可以封装各种请求参数。

通过`OkHttpClient.newCall()`创建出一个网络请求执行器`call`，有两种请求方式：

- 调用`enqueue(callback)`实现异步请求，通过子线程请求，当请求完成，将响应结果通过回调函数传递。
- 调用`execute()` 实现同步请求，堵塞当前线程，直到请求完成，返回响应结果。

```kotlin
val client = OkHttpClient.Builder().build()
// 构建请求    
val request = Request.Builder()
    .url("https://baidu.com")
    .build()
// 网络请求执行器    
val realCall = client.newCall(request)
// 执行异步请求    
val asyncResponse = realCall.enqueue(object : Callback {
    override fun onFailure(call: Call, e: IOException) {
        Log.d("okhttp", "请求失败")
    }
    override fun onResponse(call: Call, response: Response) {
        Log.d("okhttp", "请求成功")
    }
})
// 执行同步请求
val syncResponse = realCall.excute()   
```


## OkHttpClient 组件介绍

要想了解OkHttp 如何实现 HTTP 协议完成网络请求之前，就先要了解 OkHttpClient 中的组件。

两大核心组件是： **Dispatcher(分发器）和 Interceptor(拦截器)** ，这两个着重介绍，它们和OkHttp请求流程息息相关，下面先看看其他组件。



### ConnectionPool(连接池)

连接池和线程池类似，批量管理 TCP 连接，通过重用和自动回收，实现性能和资源占用的动态平衡。

TCP连接使用完成后不会被直接销毁，而是重新放回到连接池中，等待下次 HTTP 请求使用。

同时连接池中会制定规则，自动回收无用的连接，释放资源。如此就实现了连接的复用，防止一有HTTP请求就重新创建新的连接。


### EventListenerFactory(事件监听器工厂)

对如连接事件，域名解析成功事件等设置监听器。


### retryOnConnectionFailure

这是一个bool值，当连接、请求失败时，OkHttp是否重试，默认为true。注意请求失败，比如返回状态码500等，不符合重试条件。
需要是「同一个域名的多个IP重试」，「Socket连接失败」。



### Authenticator(认证器)

用于自动重新认证，配置之后当请求收到 401 未授权的状态码后，会直接调用 `authenticator` ，手动加入请求头字段`Authenticator`，重新发起请求。

比如当token 过期时，可以配置 `authenticator` 重新获取新的token，添加到请求头中，再次发起请求。

```kotlin
val client = OkHttpClient.Builder()
    .authenticator(Authenticator { route, response ->
        val request = response.request
        // TODO 重新请求服务器新的token
        val newToken = requestToken()
        request.newBuilder()
            .addHeader("Authorization", "Bearer ${newToken}")
            .build()
    })
    .build()
```



### followRedirects 和 followSslRedirects

这两个都是bool值。

followRedirects，表示是否重定向，跳转到 301 返回的 location url，默认为true。

followSslRedirects，在上面 followRedirects 为true 的基础上，当协议发生切换时，是否依然需要重定向，默认为true。

比如请求的网址是`http://baidu.com`，重定向需要跳转的网址为`https:baidu.com`，由http协议变成了https协议。



### CookieJar(Cookie管理器)

OkHttp 提供了 Cookie 的存取管理类，但和浏览器不同，`CookieJar` 中没有实现存取的逻辑（什么时候存Cookie，什么时候取Cookie，需要自己实现）。

在OkHttp 中CookieJar的默认实现为`NoCookies`，里面什么也没有，返回的也是空List，不像浏览器会自动存储请求响应的Cookie。

```kotlin
val NO_COOKIES: CookieJar = NoCookies()
private class NoCookies : CookieJar {
    override fun saveFromResponse(url: HttpUrl, cookies: List<Cookie>) {
    }
    override fun loadForRequest(url: HttpUrl): List<Cookie> {
        return emptyList()
    }
}
```

所以如果需要管理Cookie，需要在OkHttpClient 中配置，并且实现一个CookieJar，编写条件，可以用`Map`保存在内存中，用`SharedPreferences`保存在文件中。

```kotlin
val client = OkHttpClient.Builder()
    .cookieJar(object : CookieJar {
        override fun loadForRequest(url: HttpUrl): List<Cookie> {
            // 返回Cookie的逻辑
            return map.get(url)
        }
        override fun saveFromResponse(url: HttpUrl, cookies: List<Cookie>) {
            // TODO 存取Cookie的逻辑
            map.put(url, cookies)
        }
    }).build()
```


### Cache(缓存管理配置)

缓存用于保存服务器的响应结果，下次相同请求时可以直接使用。Cache，默认是空实现，需要自己配置Cache存储的文件位置和存储空间上限。

```koltin
val client = OkHttpClient.Builder()
    .cache(Cache(File(Environment.getDownloadCacheDirectory(), "cache"), 100*1024*1024))
    .build()
```


### Dns

Dns负责将主机名解析成IP地址，默认使用JDK中的`InetAddress.getAllByName(hostname).toList()`。


### Proxy(代理)

表示请求代理设置，有三种类型

- `DIRECT`,直连模式，直接和目标服务器通信，中间没有代理服务器
- `HTTP`, 代理模式，客户端请求代理服务器，代理服务器HTTP协议转发HTTP数据包给目标服务器。
- `SOCKS`，代理模式，通过Socks服务器和目标服务器通信，Socks代理只是简单地转发数据包(传输层)，而不必关心是何种应用协议（比如FTP、HTTP和NNTP请求）。


### PoxySelector(代理选择器)

代理选择器，根据你要连接的URL自动选择最合适的代理。如果需要编写定制代理选择器，需要继承并实现`select`方法。

默认实现为`NullProxySelector`，默认返回直连模式，即不使用代理。

```kotlin
// NullProxySelector
override fun select(uri: URI?): List<Proxy> {
    requireNotNull(uri) { "uri must not be null" }
    return listOf(Proxy.NO_PROXY)
}

// Proxy
public final static Proxy NO_PROXY = new Proxy();
// Creates the proxy that represents a {@code DIRECT} connection.
private Proxy() {
    type = Type.DIRECT
    sa = null
}
```


### ProxyAuthenticator(代理认证器)

和上面的`authenticator`类似，针对代理服务器做授权处理。


### SocketFactory 和 SSLSocketFactory

Socket连接工程，当需要TCP连接建立时，提供Socket服务。

SSLSocketFactory针对HTTPS 请求，在TCP连接基础上再建立一个TLS连接，提供SSLSocket



### X509TrustManager(证书验证器)

X509表示证书格式，当建立HTTPS连接时，需要验证服务端证书签名，证书签发机构证书签名，根证书签名，这些工作都是由 X509TrustManager 完成。


### ConnectionSpecs(TLS连接配置)

Https 建立连接时，客户端需要发送支持的TLS版本协议 和 对称加密、非对称加密、摘要(hash)算法套件。

默认使用`MODERN_TLS`。

```koltin
val MODERN_TLS = Builder(true)
    .cipherSuites(*APPROVED_CIPHER_SUITES)
    .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2)
    .supportsTlsExtensions(true)
    .build()
    
// 明文，不加密    
val CLEARTEXT = Builder(false).build()    
```



### Protocols(HTTP协议管理)

http协议版本管理，如HTTP/1.0、HTTP/1.1、HTTP/2等

```text
HTTP_1_0("http/1.0"),
HTTP_1_1("http/1.1"),
HTTP_2("h2"),
H2_PRIOR_KNOWLEDGE("h2_prior_knowledge"),
QUIC("quic");
```



### HostnameVerifier(主机名验证器)

用于验证HTTPS 握手中服务端证书中的主机名 是否和 客户端请求的主机一致。



### CertificatePinner(证书固定验证)

用于设置HTTPS 握手过程中针对某个在Host 额外的 Certificate Public Key Pinner，即把网站证书链中的每一个证书公钥直接拿来提前配置进 OkHttpClient 里去、作为正常证书验证机制外的一次额外验证，一般不使用。

```java
String hostname = "publicobject.com";
CertificatePinner certificatePinner = new CertificatePinner.Builder()
    .add(hostname, "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
    .build();
OkHttpClient client = OkHttpClient.Builder()
    .certificatePinner(certificatePinner)
    .build();
```



### CertificateChainCleaner(验证操作员)

使用X509TrustManager 验证整个服务端证书链。



### TimeOut超时时间

connectTimeout：建立连接(TCL 或 TLS) 的超时时间。

readTimeout：发起请求到读 到响应数据的超时时间。

writeTimeout：发起请求并被目标服务器接受的超时时间。有时候对方服务器可能由于某种原因不读取你的Request。



## 请求流程

整个OkHttp 请求主流程 靠分发器 和 拦截器 ，其他组件配合两者。

分发器Dispatcher：负责调配请求任务，内部包含一个线程池执行请求任务，对总请求数和单主机请求数有限制。

### 一、请求任务分配

`okClient.newCall(request)` 返回的是 `RealCall` 类型。下面调用`realCall.enqueue(callback)`。

`client.dispatcher` 就是分发器`Dispatcher`，将异步任务`AsyncCall`交给分发器。

`AsyncCall`继承自`RealCall`，RealCall是整个网络请求执行器，相当于大管家，管理请求相关的参数。

```kotlin
// RealCall
override fun enqueue(responseCallback: Callback) {
  check(executed.compareAndSet(false, true)) { "Already Executed" }
  callStart()
  // Dispatcher.enqueue   
  client.dispatcher.enqueue(AsyncCall(responseCallback))
}
```

异步请求执行器会先加入到等待执行队列中。
`findExistingCallWithHost()` 判断相同域名下，队列是否存在队列请求调用 `AsynCall`。 
如果有相同请求执行`call.reuseCallsPerHostFrom(existingCall)`，同步 `callsPerHost` 统一主机并发请求次数。 

`promoteAndExecute` 就是从队列中推举并执行请求。

```kotlin
// Dispatcher

// 所有异步请求最多 64 个
var maxRequests = 64
// 相同域名下，最多5个异步请求
var maxRequestsPerHost = 5
// 异步请求等待执行队列
private val readyAsyncCalls = ArrayDeque<AsyncCall>()
// 异步请求正在等待执行队列
private val runningAsyncCalls = ArrayDeque<AsyncCall>()
// 同步请求正在执行队列
private val runningSyncCalls = ArrayDeque<RealCall>()
// 异步请求使用的线程池    
private var executorServiceOrNull: ExecutorService? = null
    
internal fun enqueue(call: AsyncCall) {
  synchronized(this) {
    // 先加入到等待执行队列中  
    readyAsyncCalls.add(call)
	// 如果相同域名下的call，callsPerHost会直接被赋值成队列中已有 Call 的callsPerHost
    if (!call.call.forWebSocket) {
      val existingCall = findExistingCallWithHost(call.host)
      if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
    }
  }
  promoteAndExecute()
}


fun reuseCallsPerHostFrom(other: AsyncCall) {
    this.callsPerHost = other.callsPerHost
}
```

`promoteAndExecute()`，遍历等待执行队列中，推举出符合要求的 Call，需要满足两个条件才能执行：
- **正在执行队列已有异步请求数量不能超过 64 个。**
- **正在执行队列中相同域名下，`callsPerHost` 同一主机并发请求数不能超过 5 个，防止服务器资源扛不住，Http 连接管理中对此进行了说明。**

相同域名下的Call中`callsPerHost`在上一步都会同步为相同对象，所以当`callsPerHost.incrementAndGet()`，相同域名下的其他Call 也会 + 1。
即使相同域名下建立了多个 Call 请求对象，对相同域名的并发请求数是同步的。
想要绕过相同域名连接数限制，只能使用域名分片。

如果都符合条件，
- 加入到`executableCalls`临时执行队列，`promoteAndExecute()`执行后队列销毁。
- `runningAsyncCalls`正在执行队列中，只要请求没有结束不会被移除。

如果线程池被被关闭则移除且不执行所有队列中所有请求。
连接整成，遍历`executableCalls` 执行队列，调用`asyncCall.executeOn(executorService)` 使用线程池执行请求。

```kotlin
// Dispatcher
private fun promoteAndExecute(): Boolean {
  this.assertThreadDoesntHoldLock()
  // 可执行异步请求容器    
  val executableCalls = mutableListOf<AsyncCall>()
  val isRunning: Boolean
  synchronized(this) {
    val i = readyAsyncCalls.iterator()
    // 遍历等待队列    
    while (i.hasNext()) {
      val asyncCall = i.next()
      // 超过异步请求数 64 直接不执行。 
      if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
      // 超过相同域名并发请求数，当前不执行，可以执行其他域名的 call
      if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.
      i.remove()
      // 并发请求次数+1    
      asyncCall.callsPerHost.incrementAndGet()
      // 添加到执行队列中
      executableCalls.add(asyncCall)
      // 添加到正在执行队列中
      runningAsyncCalls.add(asyncCall)
    }
    isRunning = runningCallsCount() > 0
  }
  if (executorService.isShutdown) {
     for (i in 0 until executableCalls.size) {
        val asyncCall = executableCalls[i]
        asyncCall.callsPerHost.decrementAndGet()

        synchronized(this) {
           runningAsyncCalls.remove(asyncCall)
        }

           asyncCall.failRejected()
        }
     idleCallback?.run()
  } else { 
      for (i in 0 until executableCalls.size) {
          val asyncCall = executableCalls[i]
          asyncCall.executeOn(executorService)
      }
  }
  return isRunning
}
```

`executorService` 配置线程池参数如下:
- 核心线程数 0: 当没有任务执行时，线程池中没有任何线程资源，最大程度地减少系统资源的占用。
- 最大线程数为 MAX_VALUE: 这样线程池可以处理任意数量的任务，保证了高并发。
- 工作队列 SynchronousQueue: 该队列没有存储空间，所有有任务会执行创建线程，保证高并发。
- 存活时间为 60s: 线程空间超过 60s 就会被销毁，节约资源。

```kotlin
// Dispather
val executorService: ExecutorService
    get() {
        if (executorServiceOrNull == null) {
            executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
                SynchronousQueue(), threadFactory("$okHttpName Dispatcher", false))
        }
        return executorServiceOrNull!!
}
```

线程池会执行 RealCall 请求。因为RealCall，默认实现了`Runnable`接口，会执行它的`run`方法。
另外当执行失败时，会调用`Dispatcher.finished`方法结尾。

```kotlin
// RealCall
fun executeOn(executorService: ExecutorService) {
    client.dispatcher.assertThreadDoesntHoldLock()
        var success = false
        try {
            // 执行请求，this表示Runnable
            executorService.execute(this)
            success = true
        } catch (e: RejectedExecutionException) {
            val ioException = InterruptedIOException("executor rejected")
            ioException.initCause(e)
            noMoreExchanges(ioException)
            responseCallback.onFailure(this@RealCall, ioException)
        } finally {
            if (!success) { 
                // 请求执行失败
                client.dispatcher.finished(this)
            }
        }
}
```

调用`getResponseWithInterceptorChain()`发起请求，获取响应结果，返回给调用层。

当执行成功结束后，也会调用`Dispatcher.finished`方法结尾。

```kotlin
// RealCall
override fun run() {
  threadName("OkHttp ${redactedUrl()}") {
    var signalledCallback = false
    timeout.enter()
    try {
      val response = getResponseWithInterceptorChain()
      signalledCallback = true
      responseCallback.onResponse(this@RealCall, response)
    } catch (e: IOException) {
      ...
    } catch (t: Throwable) {
      cancel()
      ...
    } finally {
      // 请求成功完成  
      client.dispatcher.finished(this)
    }
  }
}
```

`finished()`方法中，当一个请求执行结束后，调用`promoteAndExecute()` 执行等待队列中的下一个请求任务，并且将域名下的并发请求数 -1.

如果等待队列中无任务可以执行，则能利用线程池执行其他任务。

```kotlin
internal fun finished(call: AsyncCall) {
  call.callsPerHost.decrementAndGet()
  finished(runningAsyncCalls, call)
}

private fun <T> finished(calls: Deque<T>, call: T) {
  val idleCallback: Runnable?
  synchronized(this) {
    if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")
    idleCallback = this.idleCallback
  }
  // 执行队列中的下个Call  
  val isRunning = promoteAndExecute()
  // 如果线程池空调可以先执行其他任务    
  if (!isRunning && idleCallback != null) {
    idleCallback.run()
  }
}
```

### 二、拦截器责任链

OkHttp 的核心工作是在 `getResponseWithInterceptorChain()` 中完成。

前部分将自定义的拦截器和 OkHttp 提供的拦截器依次添加到拦截器容器中。

创建 `RealInterceptorChain` 责任链对象`chain`，调用`chain.proceed(originalRequest)`，传入外部构建的`request`对象，启动责任链。

```kotlin
// RealCall
internal fun getResponseWithInterceptorChain(): Response {
  // 创建拦截器容器.
  val interceptors = mutableListOf<Interceptor>()
  // 把拦截器添加到容器中
  interceptors += client.interceptors
  interceptors += RetryAndFollowUpInterceptor(client)
  interceptors += BridgeInterceptor(client.cookieJar)
  interceptors += CacheInterceptor(client.cache)
  interceptors += ConnectInterceptor
      
  // 如果不是websocket，则可以自定义networkInterceptors 
  if (!forWebSocket) {
    interceptors += client.networkInterceptors
  }
  interceptors += CallServerInterceptor(forWebSocket)
  // 创建责任链对象	
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
    // 启动拦截器责任链，等待结果返回  
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

下面调用` chain.proceed(originalRequest)`，整个拦截器链条是如何运转的？

先拷贝一个`RealInterceptorChain`，然后index + 1，获取当前index 下的 拦截器，调用`intercept()`方法。

```java
// RealInterceptorChain
private val index: Int    override fun proceed(request: Request): Response {   
    ...    
    val next = copy(index = index + 1, request = request)	
    val interceptor = interceptors[index]    
    val response = interceptor.intercept(next) ?: throw NullPointerException("interceptor$interceptor returned null")      
...    
} 
```

在拦截器的`interceptors`做了 3 件事情：

1. 对请求预处理。
2. 调用`chain.proceed()`，index已经+1，将请求交给下个拦截器，执行`intercept()`方法。
3. 最后一个拦截器请求完成后，获取响应结果，返回给上个拦截器处理，直至传递到顶部。

整个责任链模式就像工程流水线，各司其职。一个玩具飞机有外壳装配员，引擎装配员，螺旋桨装配员，模型包装员组成。

当玩具流到谁那里，谁就负责安装他负责的这一部分，这部分安装完成后流到下一个环节，直到玩具生产完成。
![image.png](https://upload-images.jianshu.io/upload_images/4538003-00d6440ffa1710e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面看看具体每个拦截器的作用，整个拦截器链按照Interceptor添加顺序依次调用`intercept()`方法。


#### 1、自定义Interceptor

先执行开发者调用`OkHttpClient.Builder().addInterceptor()`添加的自定义拦截器，它在系统拦截器之前工作，进行最早的Request预处理工作，已经最后处理响应结果Response，可以根据需要添加header。

#### 2、重试及重定向拦截器RetryAndFollowUpInterceptor

**它会对连接做⼀些初始化⼯作，并且负责在请求失败时的重试，以及重定向的⾃动后续请求**。它的存在，可以让重试和重定向对于开发者是无感知。

1. 前置工作，调用`call.enterNetworkInterceptorExchange`创建`ExChangeFinder`。
2. 后置工作是调用`recover()`重试
3. `followUpRequest(response, exchange)`生成重定向请求，重新请求重定向的链接。

```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
  val realChain = chain as RealInterceptorChain
  var request = chain.request
  val call = realChain.call
  var followUpCount = 0
  var priorResponse: Response? = null
  var newExchangeFinder = true
  var recoveredFailures = listOf<IOException>()
  while (true) {
    // 1、初始化 
    call.enterNetworkInterceptorExchange(request, newExchangeFinder)
    var response: Response
    var closeActiveExchange = true
    try {
      // 请求取消  
      if (call.isCanceled()) {
        throw IOException("Canceled")
      }
      try {
        // 2、交给下一个拦截器  
        response = realChain.proceed(request)
        newExchangeFinder = true
      } catch (e: RouteException) {
        // 3、路由异常，连接未成功，请求没有发出去
        if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
          throw e.firstConnectException.withSuppressed(recoveredFailures)
        } else {
          recoveredFailures += e.firstConnectException
        }
        newExchangeFinder = false
        continue
      } catch (e: IOException) {
        // 3、请求发出去了，但是和服务器通信失败，服务器读取请求内容突然关闭
        if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
          throw e.withSuppressed(recoveredFailures)
        } else {
          recoveredFailures += e
        }
        newExchangeFinder = false
        continue
      }
      // 重定向会执行两次请求，第一次响应结果保存到重定向后的response中
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                .body(null)
                .build())
            .build()
      }
      val exchange = call.interceptorScopedExchange
      // 构建重定向请求    
      val followUp = followUpRequest(response, exchange)
      // 为空直接返回response   
      if (followUp == null) {
        if (exchange != null && exchange.isDuplex) {
          call.timeoutEarlyExit()
        }
        closeActiveExchange = false
        return response
      }
      val followUpBody = followUp.body
      // isOneShot默认返回false，true表示只请求一次，所以重定向失效。   
      if (followUpBody != null && followUpBody.isOneShot()) {
        closeActiveExchange = false
        return response
      }
      response.body?.closeQuietly()
      // 最多重定向20次    
      if (++followUpCount > MAX_FOLLOW_UPS) {
        throw ProtocolException("Too many follow-up requests: $followUpCount")
      }
      // 保存重定向请求参数  
      request = followUp
      // 保存上个重定向结果    
      priorResponse = response
    } finally {
      call.exitNetworkInterceptorExchange(closeActiveExchange)
    }
  }
}
```



##### enterNetworkInterceptorExchange()初始化

Exchange 表示请求- 响应一次数据交换，只有建立连接才能数据交换。

ExchangeFinder 的作用就是将寻找可重用的连接，此类的实例不是线程安全的。

```java
fun enterNetworkInterceptorExchange(request: Request, newExchangeFinder: Boolean) {  
    ...  
    if (newExchangeFinder) {    
        this.exchangeFinder = ExchangeFinder(connectionPool, createAddress(request.url),this,eventListener) 
    }
}
```



##### recover请求重试

只有满足以下所有的条件，才能发起重连：

1. `retryOnConnectionFailure = true` okhttpClient 配置时 允许重连。
2. `isRecoverable()`，如果是协议异常、超时异常、HTTPS 连接异常导致连接失败，则不能重试
3. 还有新的路由可以连接

```kotlin
// RouteException
recover(e.lastConnectException, call, request, requestSendStarted = false)
    // IOException    
    recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)    
    private fun recover(e: IOException, call: RealCall,userRequest: Request, requestSendStarted: Boolean): Boolean {  
    // 如果配置不重连，则直接返回false  
    if (!client.retryOnConnectionFailure) return false  
    // 一般不满足  
    if (requestSendStarted && requestIsOneShot(e, userRequest)) return false  
    // 是否是重连的异常  
    if (!isRecoverable(e, requestSendStarted)) return false  
   // 没有新的路由可以重连  
    if (!call.retryAfterFailure()) return false 
    // For failure recovery, use the same route selector with a new connection. 
    return true
}
```



##### 重定向

重定向是当客户端和服务端建立TCP连接成功，但是HTTP请求失败，会根据状态码是否重试。

407和401 因为未授权导致请求失败，会调用在OkHttpClient 组件介绍部分的认证器重新认证再次发起请求。

3XX，重定向状态码，会调用`buildRedirectRequest()` ，取出location字段的url，重新建立请求。

408、503、421 因为客户端和服务端自身原因导致失败，会根据条件判断是否发起一个新的请求。

**重定向次数最多只有20次**

```kotlin
private fun followUpRequest(userResponse: Response, exchange: Exchange?): Request? {
    val route = exchange?.connection?.route()
        val responseCode = userResponse.code

        val method = userResponse.request.method
        when (responseCode) {
       	//  407 代理服务器未授权，会调用proxyAuthenticator重新授权再发起请求
        HTTP_PROXY_AUTH -> {
            val selectedProxy = route!!.proxy
                if (selectedProxy.type() != Proxy.Type.HTTP) {
                    throw ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy")
                }
            return client.proxyAuthenticator.authenticate(route, userResponse)
        }
		// 401 目的服务器未授权，调用authenticator重新授权
        HTTP_UNAUTHORIZED -> return client.authenticator.authenticate(route, userResponse)
        //  3xx 重定向状态码，发起重定向请求   
        HTTP_PERM_REDIRECT, HTTP_TEMP_REDIRECT, HTTP_MULT_CHOICE, HTTP_MOVED_PERM, HTTP_MOVED_TEMP, HTTP_SEE_OTHER -> {
            return buildRedirectRequest(userResponse, method)
        }
	
        // 408 客户端连接超时
        HTTP_CLIENT_TIMEOUT -> {
           	// 是否允许能连接重试
            if (!client.retryOnConnectionFailure) {
                // The application layer has directed us not to retry the request.
                return null
            }

            val requestBody = userResponse.request.body
                if (requestBody != null && requestBody.isOneShot()) {
                    return null
                }
            val priorResponse = userResponse.priorResponse
                if (priorResponse != null && priorResponse.code == HTTP_CLIENT_TIMEOUT) {
                    // We attempted to retry and got another timeout. Give up.
                    return null
                }

            if (retryAfter(userResponse, 0) > 0) {
                return null
            }

            return userResponse.request
        }

        // 503 服务不可用
        HTTP_UNAVAILABLE -> {
            val priorResponse = userResponse.priorResponse
                if (priorResponse != null && priorResponse.code == HTTP_UNAVAILABLE) {
                    // We attempted to retry and got another timeout. Give up.
                    return null
                }

            if (retryAfter(userResponse, Integer.MAX_VALUE) == 0) {
                // specifically received an instruction to retry without delay
                return userResponse.request
            }

            return null
        }
	    // 421 从当前客户端所在的IP地址到服务器的连接数超过了服务器许可的最大范围
        HTTP_MISDIRECTED_REQUEST -> {
            val requestBody = userResponse.request.body
                if (requestBody != null && requestBody.isOneShot()) {
                    return null
                }

            if (exchange == null || !exchange.isCoalescedConnection) {
                return null
            }

            exchange.connection.noCoalescedConnections()
                return userResponse.request
        }

        else -> return null
    }
}
```



### 3、 桥接拦截器BridgeInterceptor

它负责为请求添加一些开发者不需要手动添加，但服务端需要的请求字段，加载保存的cookie。

如果有请求实体，添加`Content-Type`、`Content-Length`，另外再加上`Host`、`Connection:Keep-Alive`、`User-Agent`等常用请求头字段。

获取响应后，会用CookieJar保存服务器返回的cookie，如果使用`gzip`返回数据，则使用`GzipSource`用于解压。

```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
  val userRequest = chain.request()
  val requestBuilder = userRequest.newBuilder()
  // 1、处理请求头    
  val body = userRequest.body
  // 添加实体首部   
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
  // 添加host  
  if (userRequest.header("Host") == null) {
    requestBuilder.header("Host", userRequest.url.toHostHeader())
  }
  // 默认为长连接  
  if (userRequest.header("Connection") == null) {
    requestBuilder.header("Connection", "Keep-Alive")
  }

  // 默认期望服务器返回压缩字段
  var transparentGzip = false
  if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
    transparentGzip = true
    requestBuilder.header("Accept-Encoding", "gzip")
  }
  // 加载cookie  
  val cookies = cookieJar.loadForRequest(userRequest.url)
  if (cookies.isNotEmpty()) {
    requestBuilder.header("Cookie", cookieHeader(cookies))
  }
  // 添加UA  
  if (userRequest.header("User-Agent") == null) {
    requestBuilder.header("User-Agent", userAgent)
  }
    
  // 2、发给下一个拦截器  
  val networkResponse = chain.proceed(requestBuilder.build())

  // 3、处理响应头   
  // 保存cookie    
  cookieJar.receiveHeaders(userRequest.url, networkResponse.headers)
  val responseBuilder = networkResponse.newBuilder()
      .request(userRequest)
  // 如果服务器返回压缩格式数据，进行解压    
  if (transparentGzip &&
      "gzip".equals(networkResponse.header("Content-Encoding"), ignoreCase = true) &&
      networkResponse.promisesBody()) {
    val responseBody = networkResponse.body
    if (responseBody != null) {
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
  return responseBuilder.build()
}
```



### 4、缓存拦截器CacheInterceptor

负责HTTP的缓存处理，在建立连接发出请求前，先判断是否存在本地响应结果缓存，如果存在可以直接返回不请求服务器。，



```kotlin
@Throws(IOException::class)
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
    // 2、转向下一个拦截器  
    networkResponse = chain.proceed(networkRequest)
  } finally {
    // If we're crashing on I/O or otherwise, don't leak the cache body.
    if (networkResponse == null && cacheCandidate != null) {
      cacheCandidate.body?.closeQuietly()
    }
  }
  // If we have a cache response too, then we're doing a conditional get.
  if (cacheResponse != null) {
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
      cache.update(cacheResponse, response)
      return response.also {
        listener.cacheHit(call, it)
      }
    } else {
      cacheResponse.body?.closeQuietly()
    }
  }
  val response = networkResponse!!.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .networkResponse(stripBody(networkResponse))
      .build()
  if (cache != null) {
    if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
      // Offer this request to the cache.
      val cacheRequest = cache.put(response)
      return cacheWritingResponse(cacheRequest, response).also {
        if (cacheResponse != null) {
          // This will log a conditional cache miss only.
          listener.cacheMiss(call)
        }
      }
    }
    if (HttpMethod.invalidatesCache(networkRequest.method)) {
      try {
        cache.remove(networkRequest)
      } catch (_: IOException) {
        // The cache cannot be written.
      }
    }
  }
  return response
}
```



### 5、连接拦截器ConnectInterceptor

它负责建立连接，会使用Socket建立TCP连接，如果是HTTPS 协议，则在TCP之上建立TLS连接，返回对应的 HttpCodec 对象（⽤于编码解码 HTTP 请求）。

代码虽然不多，但是大多数功能逻辑都封装到其他类中，核心代码`realChain.call.initExchange(chain)`。

```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
  val realChain = chain as RealInterceptorChain
  val exchange = realChain.call.initExchange(chain)
  val connectedChain = realChain.copy(exchange = exchange)
  return connectedChain.proceed(realChain.request)
}
```



调用`exchangeFinder.find()`返回HTTPCodec，负责请求编解码，然后组成一个`Exchange`，负责传输单个 HTTP 请求和响应对，完成一次数据交换，保存到`RealCall`的 `exchange`字段中。

```kotlin
// RealCall
internal fun initExchange(chain: RealInterceptorChain): Exchange {
  synchronized(this) {
    check(expectMoreExchanges) { "released" }
    check(!responseBodyOpen)
    check(!requestBodyOpen)
  }
  val exchangeFinder = this.exchangeFinder!!
  // 找连接，返回codc    
  val codec = exchangeFinder.find(client, chain) 
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



1. 调用`findHealthyConnection`查找可用连接。
2. 调用`resultConnection.newCodec`创建HTTP编解码器。

```kotlin
// ExchangeFinder
fun find(
  client: OkHttpClient,
  chain: RealInterceptorChain
): ExchangeCodec {
    // 找到一个可用的连接
 	val resultConnection = findHealthyConnection(
        connectTimeout = chain.connectTimeoutMillis,
        readTimeout = chain.readTimeoutMillis,
        writeTimeout = chain.writeTimeoutMillis,
        pingIntervalMillis = client.pingIntervalMillis,
        connectionRetryEnabled = client.retryOnConnectionFailure,
        doExtensiveHealthChecks = chain.request.method != "GET"
    )
    // 根据连接创建Codec    
    return resultConnection.newCodec(client, chain)
}
```



1. `findConnection()` 查找连接
2. `isHealthy()`判断连接是否可用

如果获取的Connection不可用，则再次循环调用`findConnection()`再从连接池中获取连接。

```java
// ExchangeFinder
private fun findHealthyConnection(
  connectTimeout: Int,
  readTimeout: Int,
  writeTimeout: Int,
  pingIntervalMillis: Int,
  connectionRetryEnabled: Boolean,
  doExtensiveHealthChecks: Boolean
): RealConnection {
  while (true) {
    //查找连接  
    val candidate = findConnection(
        connectTimeout = connectTimeout,
        readTimeout = readTimeout,
        writeTimeout = writeTimeout,
        pingIntervalMillis = pingIntervalMillis,
        connectionRetryEnabled = connectionRetryEnabled
    )
    // 确认Connection可用
    if (candidate.isHealthy(doExtensiveHealthChecks)) {
      return candidate
    }
    ...
  }
}
```



从连接池获取连接的逻辑非常复杂，我们需要拆分来看，整体逻辑如下，从连接池获取连接的核心代码`connectionPool.callAcquirePooledConnection()`

```kotlin
// ExchangeFinder
private fun findConnection(
  connectTimeout: Int,
  readTimeout: Int,
  writeTimeout: Int,
  pingIntervalMillis: Int,
  connectionRetryEnabled: Boolean
): RealConnection {
  // 请求被取消直接抛出异常  
  if (call.isCanceled()) throw IOException("Canceled")
  // Attempt to reuse the connection from the call.
  val callConnection = call.connection 
  // 1、第一次请求时connection = null，跳过，第二次请求连接可能已经建立
  if (callConnection != null) {
    var toClose: Socket? = null
    synchronized(callConnection) {
      if (callConnection.noNewExchanges || !sameHostAndPort(callConnection.route().address.url)) {
        toClose = call.releaseConnectionNoEvents()
      }
    }
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

  refusedStreamCount = 0
  connectionShutdownCount = 0
  otherFailureCount = 0
  // 2、从连接池中获取连接
  if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
    val result = call.connection!!
    eventListener.connectionAcquired(call, result)
    return result
  }
  // Nothing in the pool. Figure out what route we'll try next.
  val routes: List<Route>?
  val route: Route
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
    // 3、从连接池中获取连接
    if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
      val result = call.connection!!
      eventListener.connectionAcquired(call, result)
      return result
    }
    route = localRouteSelection.next()
  }
  // 4、创建一个新的连接
  val newConnection = RealConnection(connectionPool, route)
  call.connectionToCancel = newConnection
  try {
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
  // 5、从连接池获取连接
  if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
    val result = call.connection!!
    nextRouteToTry = route
    newConnection.socket().closeQuietly()
    eventListener.connectionAcquired(call, result)
    return result
  }
  synchronized(newConnection) {
    connectionPool.put(newConnection)
    call.acquireConnectionNoEvents(newConnection)
  }
  eventListener.connectionAcquired(call, newConnection)
  return newConnection
}
```



第一个条件，`requireMultiplexed`表示 是否只获取多路复用的连接，即HTTP2连接。

因为HTTP 1.0 和 1.1 请求需要排队发送，当队头请求堵塞时，后面的请求都会被影响。因此HTTP2 做了优化，采用多路复用技术，多个请求合并使用同一个连接。

如果`requireMultiplexed = true`要求只拿HTTP2连接，但连接池中连接不支持，则continue，遍历下个连接，反之连接支持HTTP2则执行第二个条件。

如果`requireMultiplexed = false`不强求连接支持多路复用，直接执行第二个条件。

第二个条件`isEligible()`需要根据代码来判断是否有符合条件的连接，传入`address` 和 `routes`。

```java
// RealConnectionPool
fun callAcquirePooledConnection(
  address: Address,
  call: RealCall,
  routes: List<Route>?,
  requireMultiplexed: Boolean
): Boolean {
  for (connection in connections) {
    synchronized(connection) {
      // http2多路复用  
      if (requireMultiplexed && !connection.isMultiplexed) return@synchronized
      if (!connection.isEligible(address, routes)) return@synchronized
      call.acquireConnectionNoEvents(connection)
      return true
    }
  }
  return false
}
```



如果HTTP 1.0 或 1.1 连接，满足以下两个条件则可复用连接

1. 先判断连接中已有的请求数，一个连接每次只能执行 1 个HTTP请求，并且当前连接没有正在执行的请求。
2. 端口、主机名、代理、TLS版本、密码套件等连接相关的配置都要一致，则证明连接的是同一个服务器，比如请求`https://baidu.com/index/1` 和 `https://baidu.com/index/2`，两者可共用同一个连接。

如果HTTP1.0或1.1条件不满足，再验证HTTP2，HTTP2一个连接支持执行多个请求，需要满足以下4个条件：

1. 支持HTTP2连接
2. `routes` 不为空，且使用直连模式，ip相同
3. 主机名可以不同，但证书为多域名证书，当前主机名可以验证通过。

如果`https://github.com` 和 `https://githlab.com`是同一个服务器的虚拟主机，支持HTTP2，则请求这两个主机名都可以复用同一个连接。

- 如果`routes = null`，则只能获取HTTP 1.0 / 1.1 的连接
- 如果`routes != null`，则可以获取HTTP 1.0 / 1.1 的连接HTTP2 的连接

```java
// RealConnection
internal fun isEligible(address: Address, routes: List<Route>?): Boolean {
  assertThreadHoldsLock()
  // http1.0或1.1
  if (calls.size >= allocationLimit || noNewExchanges) return false
  
  if (!this.route.address.equalsNonHost(address)) return false
  
  if (address.url.host == this.route().address.url.host) {
    return true 
  }

  // HTTP/2
  if (http2Connection == null) return false
 
  if (routes == null || !routeMatchesAny(routes)) return false
 
  if (address.hostnameVerifier !== OkHostnameVerifier) return false
  if (!supportsUrl(address.url)) return false
  
  try {
    address.certificatePinner!!.check(address.url.host, handshake()!!.peerCertificates)
  } catch (_: SSLPeerUnverifiedException) {
    return false
  }
  return true
}
```



#### 从连接池获取连接过程

下面会拆分`ExchangeFinder.findConnection()`

第一次请求，connection 默认为null，当请求重试，比如重新刷新token，重定向。第一次请求完成后，当前connection已经建立可以直接复用。

比如请求`https://baidu.com/index1` 重定向返回的`https://baidu.com/index2`，则重定向可以复用连接。

如果第二次请求连接不能被复用，则会调用`closeQuietly()`方法安全释放当前连接。

```java
 // 1、第一次请求时connection = null，跳过，第二次请求连接可能已经建立
  if (callConnection != null) {
    var toClose: Socket? = null
    synchronized(callConnection) {
      // 当前连接不接受新请求或post、host不同，则会释放当前连接  
      if (callConnection.noNewExchanges || !sameHostAndPort(callConnection.route().address.url)) {
        toClose = call.releaseConnectionNoEvents()
      }
    }
	// 如果当前连接可以直接复用
    if (call.connection != null) {
      check(toClose == null)
      return callConnection
    }
    // The call's connection was released.
    toClose?.closeQuietly()
    eventListener.connectionReleased(call, callConnection)
  }
```

`routes = null`，`requireMultiplexed = false`，这两个条件一综合，先向连接池中获取HTTP1.0/1.1的连接。

```java
  refusedStreamCount = 0
  connectionShutdownCount = 0
  otherFailureCount = 0
  // 2、从连接池中获取连接
  if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
    val result = call.connection!!
    eventListener.connectionAcquired(call, result)
    return result
  }
```

`Route`：包含`address`、`proxy`、`socketAddress`，`address`包括`host`和`port`。

一个域名下有多个IP，`proxy`包含直连和代理两种模式，路由就是IP、Port、主机名、代理模式的不同组合，网络数据包客户端->代理服务器->源服务器之间的传输路径。

Selection 按照Proxy 进行分类，直连模式下route分为一类，代理模式下route 分为一类。

如果有新的路由信息，再尝试从连接池获取连接，由于`routes != null`、`requireMultiplexed  = false`,既可以获取到HTTP1.0/1.1的连接，也可以获取到HTTP2的连接。

```java
// Nothing in the pool. Figure out what route we'll try next.
  val routes: List<Route>?
  val route: Route
    
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
    //如果有新的路由选择器
    var localRouteSelector = routeSelector
    if (localRouteSelector == null) {
      localRouteSelector = RouteSelector(address, call.client.routeDatabase, call, eventListener)
      this.routeSelector = localRouteSelector
    }
    val localRouteSelection = localRouteSelector.next()
    routeSelection = localRouteSelection
    routes = localRouteSelection.routes
    if (call.isCanceled()) throw IOException("Canceled")
    // 3、从连接池中获取连接
    if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
      val result = call.connection!!
      eventListener.connectionAcquired(call, result)
      return result
    }
    route = localRouteSelection.next()
  }
```



如果前三次都没有从连接池中获取连接，则会创建一个新的连接，然后放入到缓存池当中。

```java
  // 4、创建一个新的连接
  val newConnection = RealConnection(connectionPool, route)
  call.connectionToCancel = newConnection
  try {
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
```



创建完连接为什么还要再从连接池中获取。

这是考虑到极端情况，在上面线程池中并发执行两个相同域名的请求，同时创建了两个连接。

如果连接支持HTTP2，则可以将尝试两个请求合并到一个连接中。

所以`routes != null`，`requireMultiplexed = true`只获取HTTP2的连接。

如果获取到则之前创建的2个连接会弃用其中一个，节约资源。

如果没获取到则将新创建的连接放入到连接池中。

```java
 // 5、从连接池获取连接
  if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
    val result = call.connection!!
    nextRouteToTry = route
    // 关闭弃用当前新创建的连接    
    newConnection.socket().closeQuietly()
    eventListener.connectionAcquired(call, result)
    return result
  }
  // 新连接放入到连接池中
  synchronized(newConnection) {
    connectionPool.put(newConnection)
    call.acquireConnectionNoEvents(newConnection)
  }
```

#### 连接池管理

ConnectionPool 初始画有两个构造函数，需要传入一个代理对象`RealConnectionPool`，主要功能都由`RealConnectionPool`完成。

`maxIdleConnections = 5` 表示连接池中最大空闲连接为5个。

`keepAliveDuration = 5`表示连接池中连接的最长空闲时间为5分钟。

```java
class ConnectionPool internal constructor(
  internal val delegate: RealConnectionPool
) {	
  constructor(
    maxIdleConnections: Int,
    keepAliveDuration: Long,
    timeUnit: TimeUnit
  ) : this(RealConnectionPool(
      taskRunner = TaskRunner.INSTANCE,
      maxIdleConnections = maxIdleConnections,
      keepAliveDuration = keepAliveDuration,
      timeUnit = timeUnit
  ))
  constructor() : this(5, 5, TimeUnit.MINUTES)
  /** Returns the number of idle connections in the pool. */
  fun idleConnectionCount(): Int = delegate.idleConnectionCount()
  /** Returns total number of connections in the pool. */
  fun connectionCount(): Int = delegate.connectionCount()
  /** Close and remove all idle connections in the pool. */
  fun evictAll() {
    delegate.evictAll()
  }
}
```



会先将`connection`添加到线程安全的队列`connections`中，然后添加到任务队列中，执行连接清理任务。

```java
// RealConnectionPool
private val connections = ConcurrentLinkedQueue<RealConnection>()
private val cleanupQueue: TaskQueue = taskRunner.newQueue()
private val cleanupTask = object : Task("$okHttpName ConnectionPool") {
  override fun runOnce() = cleanup(System.nanoTime())
}
fun put(connection: RealConnection) {
  this.assertThreadHoldsLock()
  connections.add(connection)
  cleanupQueue.schedule(cleanupTask)
}
```



`task.runOnce()` 即 `RealConnectionPool.cleanup()`方法执行清理任务。

先遍历任务队列中的任务，正在使用的连接跳过，如果有空闲连接，则获取一个空闲时间最长的连接。

- 队列中空闲连接已经超过 5 个，则移除空闲时间最长的连接。
- 连接空闲时间超过 5 分钟 则直接移除
- 连接空闲时间没有超过 5 分钟，比如只有 3 分钟，则记录剩余时间，2 分钟之后再调用`afterRun()`再执行。
- 没有空闲连接，但有正在使用的连接，则等5分钟之后再检查。
- 没有连接，则退出清理任务。

```java
// TaskRunner
private fun runTask(task: Task) {
  this.assertThreadDoesntHoldLock()
  val currentThread = Thread.currentThread()
  val oldName = currentThread.name
  currentThread.name = task.name
  var delayNanos = -1L
  try {
    delayNanos = task.runOnce()
  } finally {
    synchronized(this) {
      afterRun(task, delayNanos)
    }
    currentThread.name = oldName
  }
}

// RealConnectionPool
fun cleanup(now: Long): Long {
  var inUseConnectionCount = 0
  var idleConnectionCount = 0
  var longestIdleConnection: RealConnection? = null
  var longestIdleDurationNs = Long.MIN_VALUE
  // Find either a connection to evict, or the time that the next eviction is due.
  for (connection in connections) {
    synchronized(connection) {
      // 检查连接是否正在使用
      if (pruneAndGetAllocationCount(connection, now) > 0) {
        inUseConnectionCount++
      } else {
        // 如果是空闲连接  
        idleConnectionCount++
        // 计算空闲时间
        val idleDurationNs = now - connection.idleAtNs
         // 获取最长空闲时间的连接
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs
          longestIdleConnection = connection
        } else {
          Unit
        }
      }
    }
  }
  when {
    // 如果推举的连接最长闲置时间 > 5分钟 或者 空闲连接数超过 5 个，则移除它
    longestIdleDurationNs >= this.keepAliveDurationNs
        || idleConnectionCount > this.maxIdleConnections -> {
      // We've chosen a connection to evict. Confirm it's still okay to be evict, then close i
      val connection = longestIdleConnection!!
      synchronized(connection) {
        if (connection.calls.isNotEmpty()) return 0L // No longer idle.
        if (connection.idleAtNs + longestIdleDurationNs != now) return 0L // No longer oldest.
        connection.noNewExchanges = true
        connections.remove(longestIdleConnection)
      }
      connection.socket().closeQuietly()
      if (connections.isEmpty()) cleanupQueue.cancelAll()
      // Clean up again immediately.
      return 0L
    }
    // 如果空闲连接没有超过5分钟，则计算还有多久。
    idleConnectionCount > 0 -> {
      return keepAliveDurationNs - longestIdleDurationNs
    }
    // 如果有正在使用的连接则等5分钟后再检查  
    inUseConnectionCount > 0 -> {
      return keepAliveDurationNs
    }
    else -> {
      // 如果没有连接，则等有连接了再说，退出任务
      return -1
    }
  }
}
```






##### 建立连接

`newConnection.connect()`中先判断是否支持http tunnel，如果支持调用`connectTunnel()`。

如果是常用的HTTP连接，调用`connectSocket()`。

```java
// RealConnection
fun connect(
  connectTimeout: Int,
  readTimeout: Int,
  writeTimeout: Int,
  pingIntervalMillis: Int,
  connectionRetryEnabled: Boolean,
  call: Call,
  eventListener: EventListener
) {
  check(protocol == null) { "already connected" }
  var routeException: RouteException? = null
  val connectionSpecs = route.address.connectionSpecs
  val connectionSpecSelector = ConnectionSpecSelector(connectionSpecs)
  ...
  while (true) {
    try {
      if (route.requiresTunnel()) {
        // 建立httpTunnel
        connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener)
        if (rawSocket == null) {
          // We were unable to connect the tunnel but properly closed down our resources.
          break
        }
      } else {
        // 连接socket  
        connectSocket(connectTimeout, readTimeout, call, eventListener)
      }
     // 建立tls、http连接   
      establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener)
      eventListener.connectEnd(call, route.socketAddress, route.proxy, protocol)
      break
    } catch (e: IOException) {
      socket?.closeQuietly()
      rawSocket?.closeQuietly()
      ...
  }
  ...
}
```



如果是直连，调用`socketFactory.createSocket()`，`rawSocket`用于保存Socket 对象，调用`connectSocket()`建立TCP连接。

```java
// RealConnection
private fun connectSocket(
  connectTimeout: Int,
  readTimeout: Int,
  call: Call,
  eventListener: EventListener
) {
  val proxy = route.proxy
  val address = route.address
  val rawSocket = when (proxy.type()) {
    Proxy.Type.DIRECT, Proxy.Type.HTTP -> address.socketFactory.createSocket()!!
    else -> Socket(proxy)
  }
  this.rawSocket = rawSocket
  eventListener.connectStart(call, route.socketAddress, proxy)
  rawSocket.soTimeout = readTimeout
  try {
    Platform.get().connectSocket(rawSocket, route.socketAddress, connectTimeout)
  } catch (e: ConnectException) {
    throw ConnectException("Failed to connect to ${route.socketAddress}").apply {
      initCause(e)
    }
  }
```



如果Socket创建TCP连接成功，调用`establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener)`

分为两种情况，HTTP连接 和 HTTPS连接。

- HTTP连接，HTTP1.0或1.1的协议则不需要做什么，直接返回`rawSocket`，如果支持HTTP2协议则调用`startHttp2()`,则会写入`>> CONNECTION PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n` 开启HTTP2 连接。
- HTTPS连接，先会调用`connectTls()` 建立TLS连接，然后再判断支持需要开启HTTP2连接。

```java
// RealConnection
private fun establishProtocol(
  connectionSpecSelector: ConnectionSpecSelector,
  pingIntervalMillis: Int,
  call: Call,
  eventListener: EventListener
) {
  if (route.address.sslSocketFactory == null) {
    if (Protocol .H2_PRIOR_KNOWLEDGE in route.address.protocols) {
      socket = rawSocket
      protocol = Protocol.H2_PRIOR_KNOWLEDGE
      startHttp2(pingIntervalMillis)
      return
    }
    socket = rawSocket
    protocol = Protocol.HTTP_1_1
    return
  }
  eventListener.secureConnectStart(call)
  connectTls(connectionSpecSelector)
  eventListener.secureConnectEnd(call, handshake)
  if (protocol === Protocol.HTTP_2) {
    startHttp2(pingIntervalMillis)
  }
}
```



#### 获取HTTP编解码器

当成功获取可用连接后，会调用`resultConnection.newCodec()`创建HTTP编解码器。

由于HTTP1.0 或 1.1 和 HTTP2 的报文格式不同，因此根据HTTP协议有`Http1ExchangeCodec` 和  `Http2ExchangeCodec` 两种编解码器。

```java
internal fun newCodec(client: OkHttpClient, chain: RealInterceptorChain): ExchangeCodec {
  val socket = this.socket!!
  val source = this.source!!
  val sink = this.sink!!
  val http2Connection = this.http2Connection
  return if (http2Connection != null) {
    Http2ExchangeCodec(client, this, chain, http2Connection)
  } else {
    socket.soTimeout = chain.readTimeoutMillis()
    source.timeout().timeout(chain.readTimeoutMillis.toLong(), MILLISECONDS)
    sink.timeout().timeout(chain.writeTimeoutMillis.toLong(), MILLISECONDS)
    Http1ExchangeCodec(client, this, source, sink)
  }
}
```



Http1ExchangeCodec 会将headers 中的字段编码为 HTTP的请求报文格式，比如`Host: www.baidu.com\r\n`。

```java
// Http1ExchangeCodec
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



HTTP2 使用二进制分帧`Steam`，由

- **帧长度**
- **帧类型**(HEADERS 帧和 DATA 帧属于数据帧，存放的是 HTTP 报文，而 SETTINGS、PING、PRIORITY 等则是用来管理流的控制帧。)**
- 帧标志（ND_HEADERS**表示头数据结束，相当于 HTTP/1 里头后的空行（“\r\n”），**END_STREAM**表示单方向数据发送结束（即 EOS，End of Stream），相当于 HTTP/1 里 Chunked 分块结束标志（“0\r\n\r\n”））
- **流标识符**，也就是帧所属的“流”，接收方使用它就可以从乱序的帧里识别出具有相同流 ID 的帧序列，按顺序组装起来就实现了虚拟的“流”

```java
// Http2ExchangeCodec 
override fun writeRequestHeaders(request: Request) {
  if (stream != null) return
  val hasRequestBody = request.body != null
  val requestHeaders = http2HeadersList(request)
  stream = http2Connection.newStream(requestHeaders, hasRequestBody)
  // We may have been asked to cancel while creating the new stream and sending the request
  // headers, but there was still no stream to close.
  if (canceled) {
    stream!!.closeLater(ErrorCode.CANCEL)
    throw IOException("Canceled")
  }
  stream!!.readTimeout().timeout(chain.readTimeoutMillis.toLong(), TimeUnit.MILLISECONDS)
  stream!!.writeTimeout().timeout(chain.writeTimeoutMillis.toLong(), TimeUnit.MILLISECONDS)
}
```



至此ConnectInterceptor工作完成，将HTTP编解码器，建立成功的连接，都放到`Exchange(this, eventListener, exchangeFinder, codec)`对象中。

Exchanage 具备了发送请求接收响应，对请求参数编码和对响应参数接码的功能，可以完成一次数据交换了。



### 6、NetworkInterceptor 网络调试拦截器

`addNetworkInterceptor(Interceptor)` 和 前面介绍的 `addInterceptor(Interceptor)` 创建的拦截器行为逻辑使用都一样，唯一区别是位置不同。

它在发送请求拦截器前，重试重定向拦截器后，它可以获取每个请求和响应的完整原始数据（包括重定向以及重试的⼀些中间请求和响应)，主要用于做网络调试。



### 7、CallServerInterceptor

它负责实质的请求与响应的 I/O 操作，即往 Socket ⾥写⼊请求数据，和从 Socket ⾥读取响应数据。

主要通过`Exchange`实现，通过调用`writeRequestHeaders`写入数据、`flushRequest()`发送请求，`readResponseHeaders()`读取数据。

最后处理响应结果返回给上个拦截器。

```java
class CallServerInterceptor(private val forWebSocket: Boolean) : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val exchange = realChain.exchange!!
    val request = realChain.request
    val requestBody = request.body
    val sentRequestMillis = System.currentTimeMillis()

    // 写入请求报文
    exchange.writeRequestHeaders(request)

    var invokeStartEvent = true
    var responseBuilder: Response.Builder? = null
     
    // 允许使用请求体，除了HEAD 和 GET请求
    if (HttpMethod.permitsRequestBody(request.method) && requestBody != null) {
      // 客户端有POST数据要上传，可以考虑使用100-continue协议。加入头{"Expect":"100-continue"}，征询服务器情况，看服务器是否处理POST的数据
      if ("100-continue".equals(request.header("Expect"), ignoreCase = true)) {
        // 发送请求  
        exchange.flushRequest()
        responseBuilder = exchange.readResponseHeaders(expectContinue = true)
        exchange.responseHeadersStart()
        invokeStartEvent = false
      }
      if (responseBuilder == null) {
        if (requestBody.isDuplex()) {
          // Prepare a duplex body so that the application can send a request body later.
          exchange.flushRequest()
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
      // 发送请求    
      exchange.finishRequest()
    }
    if (responseBuilder == null) {
      // 读取返回结果  
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
}
```


至此OKHttp请求流程分析完毕，通过 7 大拦截器完成HTTP的请求和响应。