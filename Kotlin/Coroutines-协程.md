# 1. 引入协程

>`Kotlin` 协程在不同平台的底层实现是不一样的，这里是仅针对 `JVM` 上的 `Kotlin` 协程的学习笔记

协程本来是跟线程并列的概念，都是用来管理并发的。有的编程语言使用线程来管理并发、有的是用协程、有的两个都用，`Java` 使用的是线程、`Go` 使用的是协程、`Kotlin` 既有线程又有协程。

所以，`Kotlin` 协程就是个并发管理工具，定位和线程是一样的，例如开启一个后台任务：

````kotlin
// 线程写法
Thread {
    // 执行后台任务
}.start()

// 协程写法
GlobalScope.launch {
    // 执行后台任务
}
````

但是，`Kotlin` 毕竟是基于 `JVM` 的，它的代码最终还是要编译成字节码，而 `JVM` 提供的只有线程那一套，`Kotlin` 作为上层语言，是不可能绕过线程来创建新的并发实现的。所以本质上，<u>`Kotlin` 协程的底层依然是通过 `Java` 的线程来实现的，`Kotlin` 把线程包起来，封装成一套新的 `API` 来管理并发</u>。

`Kotlin` 协程的亮点在于以线性的方式编写一部代码，解决了并发管理中常见的回调地狱的问题。

协程不是 `Kotlin` 标准库的一部分，而是以官方库的形式存在，在项目中需要依赖[协程库](https://github.com/Kotlin/kotlinx.coroutines/blob/master/README.md)：

````kotlin
// 所有平台的通用协程库
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0-RC")
// 若是Android应用，还需要加上下面的依赖
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.9.0-RC")
````



# 2. 启动协程

并发管理，从线程的角度看有三类内容（协程类似）：

* 切线程
* 在各个线程的执行过程中等待别的线程
* 线程间共享资源访问的安全



首先，为什么需要切线程？

最常见的原因：我有代码要执行，但不希望挡住当前线程，所以得开启子线程执行这段代码。

线程写法：

````kotlin
Thread {
    // ...
}.start()
````

实际开发中通常不会直接创建线程，而是使用线程池中复用的线程：

````kotlin
val executor = Executors.newCachedThreadPool()
executor.execute {
    // ...
}
````

上面的线程通常称为子线程或后台线程，除了开启子线程执行代码，还有一种切线程的场景是：我的代码必须在特定的线程执行，例如 `Android` 必须在主线程更新界面，为此提供了 `Handler` 机制：

````kotlin
val handler = Handler(Looper.getMainLooper())
handler.post {
    // ...
}
````



`Kotlin` 启动协程写法：

````kotlin
val scope = CoroutineScope(EmptyCoroutineContext)
scope.launch {
    // ...
}
````

`CoroutineScope` 与 `CoroutineContext` 是协程的核心知识点，这里先了解 `CoroutineContext` 会提供协程启动所需要的各种上下文信息，比如：线程池。这里启动了一个协程，本质上还是切线程，将 `launch` 代码块的代码提交到线程池去执行。

协程库中，管理任务在哪个线程执行的类为 `ContinuationInterceptor`，作用是代码在往下执行之前先拦截住，切个线程再继续执行。协程库提供了四个直接可用的 `ContinuationInterceptor`，在 `kotlinx.coroutines.Dispatchers` 中：

````kotlin
public actual val Default: CoroutineDispatcher = DefaultScheduler

public actual val Main: MainCoroutineDispatcher get() = MainDispatcherLoader.dispatcher

public actual val Unconfined: CoroutineDispatcher = kotlinx.coroutines.Unconfined

public val IO: CoroutineDispatcher = DefaultIoScheduler
````

`Dispatchers.Default` 适用于 CPU 密集型任务，`Dispatchers.IO` 适用于 IO 密集型任务，他们的区别在于线程池的大小

|                       |     核心线程数      |                             备注                             |
| :-------------------: | :-----------------: | :----------------------------------------------------------: |
| `Dispatchers.Default` | 与设备CPU核心数相同 | 虽然线程数越多效率越高，但当线程数超过CPU核心数时，由于CPU进行线程上下文切换，效率反而会降低 |
|   `Dispatchers.IO`    |         64          | IO密集型任务，CPU大部分时间处于空闲状态，即线程虽然开着，但实际工作的是磁盘或者网卡 |

`Dispatchers.Main` 顾名思义，可以将任务扔到主线程去执行，通常使用 `Dispatchers.Main.immediate`，因为它会先判断当前是否已经处于主线程，若是则直接执行任务，否则才扔到主线程。

`Unconfined` 基本用不上



# 3. 挂起协程

> 协程比线程最大的优势在于：协程支持自动将线程切回来

### 3.1 编译前

`Kotlin` 从语法层面提供了 `suspend` 关键字标识一个函数为挂起函数

```kotlin
suspend fun suspendFunc() {
    ...
}
```

`Android` 应用开发最常见的场景：子线程进行网络请求获取数据，然后在主线程更新界面。最常见的写法应该是回调，在参数 `callback` 中封装切到主线程并更新界面的逻辑：

````kotlin
fun networkFunc(callback :((Response) -> Unit)) {
    okHttpClient.newCall(request).enqueue(object : Callback {
        override fun onResponse(call: Call, response: Response) {
            callback(response)
        }
    })
}

fun main() {
    networdFunc { response ->
        runOnUiThread {
            updateUI(response)
        }
    }
}
````

采用协程 + 挂起函数的方式：

````kotlin
suspend fun networkFunc(): Response = withContext(Dispatchers.IO) {
    okHttpClient.newCall(request).execute()
}

fun main() {
    CoroutineScope(Dispatchers.Main).launch { 
        val response = networkFunc()
        updateUI(response)
    }
}
````

当程序执行到第七行的 `networkFunc()` 时，会将协程体切到子线程执行，`networkFunc()` 执行完后会将协程体自动切回原来的主线程执行，进行剩下的赋值操作、`updateUI()`。

简单来说，当协程体执行到挂起函数，会将协程体切到挂起函数指定的线程执行，挂起函数执行完后再自动切回原来的线程继续执行协程体剩余的代码。

个人理解：

* 所谓挂起，**挂起的是协程体，将协程体从当前线程脱离，挂到别的线程**
* 所谓恢复，**将协程体恢复到原来的线程，挂起的逆操作**



### 3.2 编译后































# 4. 结构化并发











# 5. 最佳实践

> 仅针对 `Android` 应用开发的最佳实践

实际开发中，通常不会使用自定义的 `CoroutineScope` 来启动协程：

````kotlin
val scope = CoroutineScope(Dispatchers.Default)

fun main() {
    scope.launch { 
        ...
    }
}
````

### 5.1 lifecycleScope

`Jetpack` 中，有大量针对 `Kotlin` 语言特性的扩展，即所谓的 [KTX](https://developer.android.com/kotlin/ktx?hl=zh-cnhttps://developer.android.com/kotlin/ktx?hl=zh-cn)。其中 `LifecycleOwner` 提供了扩展属性`lifecycleScope`：

````kotlin
//LifecycleOwner.kt

/**
 * 该CoroutineScope与LifecycleOwner的生命周期绑定；
 * 当LifecycleOwner的生命周期处于Destroyed时，该CoroutineScope会被自动取消；
 * 该CoroutineScope的上下文中的一个元素为Dispatchers.Main.immediate。
 */
public val LifecycleOwner.lifecycleScope: LifecycleCoroutineScope
    get() = lifecycle.coroutineScope
````

`ComponentActivity` 和 `Fragment` 均实现了 `LifecycleOwner`，所以在 `Activity` 和 `Fragment` 中可以直接用 `lifecycleScope`，而不需要手动创建的销毁 `CoroutineScope`：

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        lifecycleScope.launch {
            ...
        }
    }
}
```

### 5.2 viewModelScope

对于 `ViewModel`，需要先添加其 [KTX](https://developer.android.com/kotlin/ktx?hl=zh-cn#viewmodel) 依赖：

```kotlin
dependencies {
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.3")
}
```

与 `LifecycleOwner` 类似，`ViewModel` 也提供了扩展属性 `viewModelScope`：

````kotlin
// ViewModel.kt

/**
 * 该CoroutineScope与ViewModel的生命周期绑定；
 * 当清楚ViewModel时，该CoroutineScope会被自动取消；
 * 该CoroutineScope的上下文中的有Dispatchers.Main.immediate和SupervisorJob。
 */
public val ViewModel.viewModelScope: CoroutineScope
    get() = synchronized(VIEW_MODEL_SCOPE_LOCK) {
        getCloseable(VIEW_MODEL_SCOPE_KEY)
            ?: createViewModelScope().also { scope -> addCloseable(VIEW_MODEL_SCOPE_KEY, scope) }
    }
````

同样在自定义 `ViewModel` 中可以直接用 `viewModelScope`，而不需要手动创建的销毁 `CoroutineScope`：

```kotlin
class MainViewModel: ViewModel() {
    fun doSomething() {
        viewModelScope.launch { 
            ...
        }
    }
}
```







# CoroutineScope/-Context

