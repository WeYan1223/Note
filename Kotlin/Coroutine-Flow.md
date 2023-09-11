#### Flow

> 引入 `Flow`

`Flow` 于 [2019 年发布](https://www.youtube.com/watch?v=tYcqn48SMT8)，属于 `Kotlin` 协程库的一部分，在项目中需要依赖[协程库](https://github.com/Kotlin/kotlinx.coroutines/blob/master/README.md)：

````kotlin
// 所有平台的通用协程库
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
// Android 平台的通用协程库，当中已经包含了上面的 coroutines-core
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
````



> `Flow` 是什么？

数据流是基于生产者-消费者模型的数据结构，主要由三部分组成：

* 上流（生产者）
* 中间操作符
* 下流（消费者）

数据从上流产生，可能还会经过中间操作符的变换、过滤等操作，最终到达下流被消费



> 冷流、热流

* 冷流：没有消费者，生产者不生产数据，通过 `flow()`、`flowOf()` 等函数创建的就是冷流
* 热流：没有消费者，生产者也可以生产数据，常用的有 `StateFlow` 和 `SharedFlow`



> 基本原理

````kotlin
/**
 * 数据流
 */
public interface Flow<out T> {
    /**
     * 对外提供收集数据的函数
     */
    public suspend fun collect(collector: FlowCollector<T>)
}

/**
 * 收集者
 */
public fun interface FlowCollector<in T> {
    /**
     * 这个函数命名为 emit 确实挺迷的
     * 可以理解为一个回调函数，收集数据时最终都会回调该方法
     */
    public suspend fun emit(value: T)
}
````

````kotlin
suspend fun test() {
    val flow = flow {
        emit(5)
    }
    flow.collect {
        println("value=$it")
    }
}
````

这里由五部分组成：`flow()`、`flow` 闭包、`emit()`、`collect()`、`collect` 闭包，实际流程如下

````sh
flow() -> collect() -> flow闭包 -> emit() -> collect闭包
````

首先看 `flow()` 里面做的什么：

````kotlin
/**
 * 仅仅是保存了接收者类型为 FlowCollector 的函数类型 block()，并返回一个 Flow 对象
 */
public fun <T> flow(block: suspend FlowCollector<T>.() -> Unit): Flow<T> = SafeFlow(block)

private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block()
    }
}
````

再看下 `collect()` 里面做了什么：

````kotlin
/**
 * AbstractFlow 重写了该方法
 * 参数 collector 即为 emit() 的真正实现
 */
public final override suspend fun collect(collector: FlowCollector<T>) {
    // 对 collector 进行包装
    // 可以理解为对 emit() 进行包装，主要是启动协程以及做一些协程相关的工作
    val safeCollector = SafeCollector(collector, coroutineContext)
    try {
        // 最终会执行前面保存的 block()
        collectSafely(safeCollector)
    } finally {
        safeCollector.releaseIntercepted()
    }
}
````



> `SharedFlow`

定义

* 热流，即没有消费者也可以往里面生产数据
* 共享的，可以被多个消费者监听
* 缓存数据，缓存一定数量的最新数据，当新的消费者监听时，它可以获取到一部分缓存的数据，从而避免错过一些重要的更新

使用场景

* `UI` 更新：保存 `UI` 状态，在多个 `UI` 组件中共享状态，使它们能够同时接收到相同的更新
* 数据同步：一个模块更新数据时，其他消费者可以即时获取到这个更新，从而保持数据的一致性
* 数据重放：新订阅者可以获取部分历史数据



> `StateFlow`

定义

* `SharedFlow` 的特殊版，只维护一个值，新值与旧值不同就将旧值覆盖

使用场景

* 适用于只关注最新值的变化（可以说是大部分 `UI` 场景）





