#### OkHttp

> 基于 `OkHttp 4.10.0`，官网：[OkHttp (square.github.io)](https://square.github.io/okhttp/)

`OkHttp` 是 `Android` 开发中常用的网络请求框架，用于发送请求与接收响应

从发送一个最简单的 `HTTP` 请求开始

````kotlin
val okHttpClient = OkHttpClient()
val request: Request = Request.Builder()
	.url("https://www.google.com/")
	.build()
okHttpClient.newCall(request).execute()
````

> `OkHttpClient`

`OkHttpClient` 是 `HTTP` 请求的工厂，用于发送请求与接收响应

应用程序中，`OkHttpClient` 最好作为单例使用，原因是其内部维护着 `TCP` 连接池与线程池

> `Request`

`Request` 代表 `HTTP` 请求报文，结构如下

````kotlin
/**
 * url
 */
val url: HttpUrl,

/**
 * 请求方法，默认 GET
 */
val method: String,

/**
 * 首部字段
 */
val headers: Headers,

/**
 * 请求报文主体，可为空
 */
val body: RequestBody?
````

> `Call`

`OkHttpClient#newCall(request)` 返回一个 `Call` 对象，代表的是已准备好执行的请求，主要提供以下方法来执行请求

````kotlin
interface Call {
    /**
     * 同步操作，立刻执行请求
     * 返回响应报文
     */
    fun execute(): Response
    
    /**
     * 异步操作，进入异步队列等待某个时机执行
     * 通过 responseCallback 回调来处理响应报文
     */
    fun enqueue(responseCallback: Callback)
}
````

> `RealCall` 是 `Call` 接口的唯一实现类

`OkHttp` 整个通信流程通过责任链实现，`RealCall#getResponseWithInterceptorChain()` 是责任链构建与处理的核心方法

关于 `OkHttp` 是如何使用责任链模式来实现拦截器的，参考 [Note/设计模式/行-责任链模式.md at master · WeYan1223/Note (github.com)](https://github.com/WeYan1223/Note/blob/master/设计模式/行-责任链模式.md)

整体流程如下

<img src="https://raw.githubusercontent.com/WeYan1223/Pic/master/Android/OkHttp_整体流程.webp" alt="OkHttp_整体流程.webp (1044×715) (raw.githubusercontent.com)" style="zoom:80%;" />  

> `RetryAndFollowUpInterceptor`，负责重试与重定向

伪代码如下

````kotlin
while (true) {
    try {
        // 执行下一个拦截器获取响应 
        response = realChain.proceed(request)
    } catch (e: RouteException) {
        // 连接路由异常，此时请求还未发送，尝试恢复
        if (!recover()) {
            throw e.firstConnectException.withSuppressed(recoveredFailures)
        }
        continue
    } catch (e: IOException) {
        // IO异常，请求可能已经发出，尝试恢复
        if (!recover()) {
            throw e.withSuppressed(recoveredFailures)
        }
        continue
    }

    // 跟进结果，主要作用是根据响应码处理请求
    val followUp = followUpRequest(response, exchange)
    
    // followUp 为空，不需要重定向
    if (followUp == null) {
        return response
    }

    // followUp 不为空，需要重定向
    request = followUp

    // 最多重定向 20 次
    if (++followUpCount > MAX_FOLLOW_UPS) {
        throw ProtocolException("Too many follow-up requests: $followUpCount")
    }
}
````

> `BridgeInterceptor`，负责封装请求报文与简单处理响应报文

发送报文前：

````kotlin
// 若报文主体不为空，需要添加必要的实体字段
if (body != null) {
    val contentType = body.contentType()
    if (contentType != null) {
        // 报文主体的数据格式
        requestBuilder.header("Content-Type", contentType.toString())
    }

    val contentLength = body.contentLength()
    if (contentLength != -1L) {
        // 报文主体的大小（单位：字节）
        requestBuilder.header("Content-Length", contentLength.toString())
        requestBuilder.removeHeader("Transfer-Encoding")
    } else {
        // 报文主体的传输编码方式
        requestBuilder.header("Transfer-Encoding", "chunked")
        requestBuilder.removeHeader("Content-Length")
    }
}

if (userRequest.header("Host") == null) {
    // 服务器域名
    requestBuilder.header("Host", userRequest.url.toHostHeader())
}

if (userRequest.header("Connection") == null) {
    // 默认 TCP 长连接
    requestBuilder.header("Connection", "Keep-Alive")
}

// cookie处理，需要在初始化 OkhttpClient 时配置 CookieJar
val cookies = cookieJar.loadForRequest(userRequest.url)
if (cookies.isNotEmpty()) {
    requestBuilder.header("Cookie", cookieHeader(cookies))
}

if (userRequest.header("User-Agent") == null) {
    // HTTP 客户端的信息
    requestBuilder.header("User-Agent", userAgent)
}
````

接收响应后：

```kotlin
// 将响应头的 cookie 存入 CookieJar（如果有的话）
cookieJar.receiveHeaders(userRequest.url, networkResponse.headers)

val responseBuilder = networkResponse.newBuilder().request(userRequest)

// 如果是请求头的 Accept-Encoding 为 gzip，并且响应头的 Content-Encoding 也是 gzip，这里将保温主体包装一层 GzipSource 方便后面解压缩用
if (transparentGzip && "gzip".equals(networkResponse.header("Content-Encoding"), ignoreCase = true) &&
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
```

通过 `CookeiJar` 可以非常简单地实现对 `Cookie` 的管理

```kotlin
interface CookieJar {
    /**
     * 保存响应头的所有 Set-Cookie
     */
    fun saveFromResponse(url: HttpUrl, cookies: List<Cookie>)

    /**
     * 通过 url 获取相关的所有 Cookie（这时一般加上清除过期 Cookie 的操作）
     */
    fun loadForRequest(url: HttpUrl): List<Cookie>

}
```

> `CacheInterceptor`，负责读取缓存与缓存

关于 `HTTP` 缓存机制，参考[Note/计算机网络/协议-应用层_HTTP.md at master · WeYan1223/Note (github.com)](https://github.com/WeYan1223/Note/blob/master/计算机网络/协议-应用层_HTTP.md#缓存机制)

注意，`OkHttp` 默认只支持 `GET` 请求的缓存

> `ConnectInterceptor`，负责建立连接

内部会维护一个连接池，负责连接复用、创建连接（三次握手等等）、释放连接以及创建连接上的socket流

```kotlin
object ConnectInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val realChain = chain as RealInterceptorChain
        // 获取一个 Exchange 实例，然后往下传递，Exchange 可以理解为一个 TCP 连接
        val exchange = realChain.call.initExchange(chain)
        val connectedChain = realChain.copy(exchange = exchange)
        return connectedChain.proceed(realChain.request)
    }
}
```

这里需要了解两个地方：

* 如何复用 `TCP` 连接？

  这部分逻辑主要在 `ExchangeFinder#findConnection()` 中

  1.  首先尝试使用已分配给该请求的连接（重定向时的再次请求）
  2.  若没有已分配连接，则尝试从连接池中获取。因为此时没有路由信息，所以匹配条件：address 一致—— host、port、代理等一致，且匹配的连接可以接受新的请求
  3.  若从连接池没有获取到，则传入 routes 再次尝试获取，这主要是针对 Http2.0 的一个操作, Http2.0 可以复用 square.com 与 square.ca 的连接
  4.  若第二次也没有获取到，就创建 `RealConnection` 实例，进行 `TCP + TLS` 握手，与服务端建立连接
  5.  此时会第三次从连接池匹配（因为新建立的连接的握手过程是非线程安全的，所以此时可能连接池新存入了相同的连接）
  6.  第三次若匹配到，就使用已有连接，释放刚刚新建的连接；若未匹配到，则把新连接存入连接池并返回

* 如何清除空闲 `TCP` 连接？

  这部分逻辑主要在 `RealConnectionPool#cleanup()` 中

  1. 将连接加入连接池时就会启动定时任务
  2. 有空闲连接的话，如果空闲时间大于5分钟或空闲的连接数大于5，就清除这个空闲连接；如果空闲时间不大于 5 分钟，就等待到达 5 分钟再来清理
  3. 没有空闲连接就等5分钟后再尝试清理
  4. 没有连接不清理

> `CallServerInterceptor`，负责发送数据与读取数据

在前置准备工作完成后，真正发起了网络请求，进行网络 `IO` 的读写

> `interceptors` 与 `networkInterceptors` 的区别为它们处于责任链的位置不同

`interceptors` 位于责任链的头部，当发送请求时，可以对请求进行修改、记录日志、添加认证信息等操作；响应时进行处理、解析、修改等操作

`netnetworkInterceptors` 位于责任链的倒数第二的位置，可以得到最原始的响应信息