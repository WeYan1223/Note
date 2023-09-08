#### 协程

> 什么是协程？

* 官方文档：协程是轻量级的线程。但是协程属于语言层面，线程属于系统层面，两者没有可比性，不过用的时候可以这么理解
* 本质上，协程是对线程的封装，可以使开发者以同步的方式编写异步代码，解决了传统异步编程时回调过多的问题，利用了 `Kotlin` 语言以及编译器提供的优化，使得开发者可以很方便地以同步的方式写出异步代码



> 在项目中引入协程

协程不是 `Kotlin` 标准库的一部分，而是以官方库的形式存在，在项目中需要依赖[协程库](https://github.com/Kotlin/kotlinx.coroutines/blob/master/README.md)：

````kotlin
// 所有平台的通用协程库
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
// Android 平台的通用协程库，当中已经包含了上面的 coroutines-core
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
````



***

#### 上下文

协程只能在 `CuroutineScope` 中创建，即协程作用域

````kotlin
/**
 * 协程作用域
 */
public interface CoroutineScope {
    /**
     * 协程上下文，保存协程运行所需的一切信息
     */
    public val coroutineContext: CoroutineContext
}
````

协程作用域的一切行为都由 `CoroutineContext` 控制，所以不同的 `Scope` 本质上是 `CoroutineContext` 的差异

`CoroutineContext` 是一种特殊的数据结构（可以简单理解为特殊的 `Map`），保存了定义协程行为的元素

* `Job`：可以理解为协程本体，用于保存协程的信息以及控制协程
* `CoroutineName`：当前协程名称
* `CoroutineDispatcher`：协程调度器，用于指定协程应该运行在哪个线程
* `CoroutineExceptionHandler`：用于处理协程中的异常



> `Job`

`Job` 的主要结构：

````kotlin
public interface Job : CoroutineContext.Element {
    public val isActive: Boolean
    public val isCompleted: Boolean
    public val isCancelled: Boolean
    //...
    
    public fun start(): Boolean
    public suspend fun join()
    public fun cancel(cause: CancellationException? = null)
    //...
}
````

 简单来说，`Job` 是用来记录、控制协程的状态，并且每个协程都对应一个 `Job`

    +-----+ start  +--------+ complete   +-------------+  finish  +-----------+
    | New | -----> | Active | ---------> | Completing  | -------> | Completed |
    +-----+        +--------+            +-------------+          +-----------+
                     |  cancel / fail       |
                     |     +----------------+
                     |     |
                     V     V
                 +------------+                           finish  +-----------+
                 | Cancelling | --------------------------------> | Cancelled |
                 +------------+                                   +-----------+




> 协程的取消

`Job` 有一个最广泛的用途：取消协程，<u>协程的取消是协作式的，其思想与[线程中断](https://github.com/WeYan1223/Note/blob/master/Java/Java-线程.md#终止线程)类似</u>

`kotlin.coroutines` 包中的 `suspend` 函数都是可取消的，它们会检查协程的取消，并在取消时抛出 `CancellationException`

因此，如果一段协程代码内部没有检查取消并抛出 `CancellationException`，那么就算外部调用了 `Job#cancel()`，协程也不会被取消，例如

````kotlin
val job = GlobalScope.launch {
    var nextPrintTime = System.currentTimeMillis()
    var i = 0
    while (true) {//一个执行计算的循环，只是为了占用 CPU
        //每秒打印消息两次
        if (System.currentTimeMillis() >= nextPrintTime) {
            println("job: I'm sleeping ${i++} ...")
            nextPrintTime += 500L
        }
    }
}
Thread.sleep(1000)
job.cancel()
````

上面的代码不会正常退出，正常退出需要协程内部代码的协作，例如

````kotlin
val job = GlobalScope.launch {
    var nextPrintTime = System.currentTimeMillis()
    var i = 0
    //只有在协程处于Active状态才执行
    while (isActive) {
        if (System.currentTimeMillis() >= nextPrintTime) {
            println("job: I'm sleeping ${i++} ...")
            nextPrintTime += 500L
        }
    }
}
Thread.sleep(1000)
job.cancel()
````

上面通过代码顺序执行结束来停止协程，当然也可以抛出 `CancellationException` 来结束协程

````kotlin
val job = GlobalScope.launch {
    var nextPrintTime = System.currentTimeMillis()
    var i = 0
    while (true) {
        if (System.currentTimeMillis() >= nextPrintTime) {
            println("job: I'm sleeping ${i++} ...")
            nextPrintTime += 500L
        }
        if (!isActive) {
            throw CancellationException()
        }
    }
}
Thread.sleep(1000)
job.cancel()
````

这个 `CancellationException` 可以理解为协程取消的标志，协程框架发现该异常，就知道是时候取消协程了



> `CoroutineDispatcher`

协程调度器，用于指定协程应该运行在哪种线程，主要有以下四种选项：

````kotlin
public actual object Dispatchers {
    /**
     * 默认选项，运行在另一个子线程
     */
    public actual val Default: CoroutineDispatcher = DefaultScheduler

    /**
     * 运行在主线程
     */
    public actual val Main: MainCoroutineDispatcher get() = MainDispatcherLoader.dispatcher

    /**
     * 运行在声明协程所在的线程
     */
    public actual val Unconfined: CoroutineDispatcher = kotlinx.coroutines.Unconfined

    /**
     * 运行在另一个子线程
     */
    public val IO: CoroutineDispatcher = DefaultIoScheduler
}
````





> `CoroutineExceptionHandler`

用于处理协程中的异常



***

#### 挂起函数

> `suspend` 修饰的函数称为挂起函数，挂起函数只能在协程中或由其他挂起函数调用

对于函数调用者来说，只需知道 `suspend` 的作用是：<u>提示调用者，协程执行到挂起函数时，该函数不会马上返回，需要将协程挂起，待挂起函数执行完之后再恢复协程的执行</u>



> `CPS`，`Continuation Passing Style`

实际流程与代码一致按序执行的代码风格，称为 `Direct Style`

````kotlin
/**
 * 发送item数据
 */
fun postItem(item: Item) {
    // 发送请求获取token
    val token = requestToken()
    // 根据token与item创建post请求
    val post = createPost(token, item)
    // 发送post请求
    processPost(post)
}
````

`CPS`，即 `Continuation Passing Style`，代码风格跟平时写的 `Callback` 差不多

````kotlin
fun postItem(item: Item) {
    // 发送请求获取token
    requestToken { token ->
        // 根据token与item创建post请求
        val post = createPost(token, item)
        // 发送post请求
        processPost(post)
    }
}
````

对比平时写的 `Callback`

````kotlin
fun postItem(item: Item) {
    // 若 Callback 只有一个方法，那么在语法上简化一下代码就与 CPS 写法一致了
    requestToken(object: Callback {
        override fun onSuccess(token: Token) {
            val post = createPost(token, item)
            processPost(post)
        }

        override fun onFilure() {
            //...
        }
    })
}
````

通过协程与 `suspend` 函数，我们可以方便地以同步的方式编写异步代码，本质上是编译器提供的优化，编译后 `Direct Style` 代码会被转换为 `CPS` 代码，说白了还是回调，例如：

````kotlin
suspend fun createPost(token: Token, item: Item): Post
````

转换后

````java
Object createPost(Token token, Item item, Continuation<Post> cont)
````

这里新参数 `Continuation` 就可以理解为 `Callback` 接口，里面封装了后续需要执行的代码



> 挂起什么？恢复什么？如何挂起？如何恢复？

先看最重要的接口 `Continuation`

````kotlin
/**
 * 翻译为续体，说白了就是一个回调接口
 */
public interface Continuation<in T> {
    // 协程上下文
    public val context: CoroutineContext

    // 当挂起函数执行完成，调用该方法使得协程从挂起点恢复
    public fun resumeWith(result: Result<T>)
}
````

`Continuation` 继承关系

````sh
- Continuation: 续体，提供协程上下文，声明 resumeWith() 恢复协程的执行
    - BaseContinuationImpl: 实现 resumeWith() 方法，控制状态机的执行，定义了 invokeSuspend() 抽象方法
        - ContinuationImpl: 增加 intercepted 拦截器，实现线程调度等
            - SuspendLambda: 封装协程体代码块
                - 协程体代码块生成的子类: 实现 invokeSuspend() ，即状态机流转逻辑
````

编译后的挂起函数会生成一个 `ContinuationImpl` 的子类实例（理解为状态机），实现了 `invokeSuspend()`

挂起和恢复有两个关键方法：`invokeSuspend()` 和 `resumeWith(Result)`：

* 编译器会为挂起函数自动添加一个 `Continuation` 参数，表示调用它的那个协程代码块的续体，在该挂起函数执行完成后，就会调用其 `resumeWith() ` 来返回结果并恢复协程的执行（`resumeWith()` 内部调用了 `invokeSuspend()` 根据状态机的状态来恢复协程的执行)
* `invokeSuspend()` 是对整个挂起函数的变换，以状态机的形式将整个逻辑分为多块，分隔点就是每个挂起点
* 在这之中，每当调用到一个挂起函数时，挂起函数会返回 `COROUTINE_SUSPENDED` 标识，从而 `return` 停掉 `invokeSuspend()` 函数的执行





***

#### 参考文章

* [Kotlin 的协程用力瞥一眼 - 学不会协程？很可能因为你看过的教程都是错的 (rengwuxian.com)](https://rengwuxian.com/kotlin-coroutines-1/)
* [Kotlin 协程的挂起好神奇好难懂？今天我把它的皮给扒了 (rengwuxian.com)](https://rengwuxian.com/kotlin-coroutines-2/)
* [到底什么是「非阻塞式」挂起？协程真的更轻量级吗？ (rengwuxian.com)](https://rengwuxian.com/kotlin-coroutines-3/)
* [Kotlin 协程与架构组件一起使用及底层原理分析 - 掘金 (juejin.cn)](https://juejin.cn/post/6970809644511920165)
* [Kotlin协程之再次读懂协程工作原理 - 掘金 (juejin.cn)](https://juejin.cn/post/7137905800504148004)