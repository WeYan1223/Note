#### LeakCanary

> 官网：[LeakCanary (square.github.io)](https://square.github.io/leakcanary/)

##### 1. 初始化

从 `LeakCanary 2.0` 开始，集成 `LeakCanary` 只需要添加依赖即可，不需要增加任何代码

````groovy
debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.12'
````

这里利用到了 `ContentPovider` 的初始化时机：打包过程中来自不同模块的 `ContentProvider` 最终都会合并到一个文件中，启动 `app` 时大致流程如下

````sh
Application.attachBaseContext() -> ContentProvider.onCreate() -> Application.onCreate() ->Activity.onCreate()
````

在 `leakcanary-object-watcher-android` 模块的 `AndroidManifest.xml` 文件中发现注册了一个名为 `MainProcessAppWatcherInstaller` 的 `Provider`：

````kotlin
internal class MainProcessAppWatcherInstaller : ContentProvider() {
    // 纯粹利用系统回调该方法的时机来做初始化
    override fun onCreate(): Boolean {
        val application = context!!.applicationContext as Application
        // 进行 LeakCanary 初始化操作
        AppWatcher.manualInstall(application)
        return true
    }
    
    // CURD 操作全为空
}
````

##### 2. 检测内存泄漏

`LeakCanary` 的初始化操作中，这里主要关注 `LeakCanary` 是如何检测到内存泄露的（以 `Activity` 为例）

````kotlin
/**
 * Activity 内存泄漏的检测者
 */
class ActivityWatcher(
    private val application: Application,
    private val reachabilityWatcher: ReachabilityWatcher
) : InstallableWatcher {
    private val lifecycleCallbacks = object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
        /**
         * 当 Activity#onDestory() 被调用时，回调该方法
         */
        override fun onActivityDestroyed(activity: Activity) {
            // 大致流程都在这里面了
            reachabilityWatcher.expectWeaklyReachable(
                activity, "${activity::class.java.name} received Activity#onDestroy() callback"
            )
        }
    }

    /**
     * LeakCanary 初始化时调用该方法
     */
    override fun install() {
        application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
    }
}
````

````kotlin
/**
 * 保存正在检测的对象，与 queue 一起使用
 */
private val watchedObjects = mutableMapOf<String, KeyedWeakReference>()

/**
 * 引用队列（如果对象被正常回收，那么其弱引用对象会被加入到引用队列中）
 */
private val queue = ReferenceQueue<Any>()

@Synchronized override fun expectWeaklyReachable(watchedObject: Any, description: String) {
    // 把回收成功的对象从 watchedObjects 移除
    removeWeaklyReachableObjects()
    // 为 watchedObject 生成一个独一无二的 key
    val key = UUID.randomUUID().toString()
    // 这个时间用来记录 watchedObject 存活了多久
    val watchUptimeMillis = clock.uptimeMillis()
    // 将 watchedObject 与弱引用关联，并将该弱引用与引用队列关联
    val reference = KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
    // 把 reference 存放到 watchedObjects 中，说明 watchedObject 正在被检测
    watchedObjects[key] = reference
    checkRetainedExecutor.execute {
        // 开始尝试检测内存泄漏
        moveToRetained(key)
    }
}
````

如果 `GC` 操作完成，会调用 `removeWeaklyReachableObjects()` 尝试把回收成功的对象从 `watchedObjects` 移除，如果 `watchedObjects` 没有清空，则触发内存泄漏处理（当然 5 秒内也不一定会触发 `GC`，所以之后会主动触发 `GC` 再判断一次）

总结：首先为 `Activity` 注册声明周期的监听，当监听到 `onDestory()` 发生时，创建一个弱引用对象指向该 `Activity` 并关联到引用队列中，如果 `GC` 操作完成，且弱引用对象出现在引用队列 `queue` 中，说明 `Activity` 被正常回收，否则手动触发一次 `GC`，由于手动发起 `GC` 并不会立即执行垃圾回收，所以需要在一定时延后确认 `Activity` 是否已经回收，若出现内存泄漏，`LeakCanary` 会对堆栈进行分析，最终会在另一个进程中展示相关的引用链

