#### ViewModel

> `ViewModel` 是什么？

从 `MVVM` 架构来说，`ViewModel` 是 `View` 层与 `Model` 层之间的桥梁，使得 `View` 层与 `Model` 层完全解耦，同时 `ViewModel` 存储 `View` 的状态，`View` 监听状态来做相应的 `UI` 更新

从 `Jetpack` 的角度来说，`ViewModel` 就是一个类，可以使得开发者很方便地实现 `MVVM` 架构



> `ViewModel` 有什么用？

* 实现 `MVVM` 架构
* `Fragment` 之间共享数据：同一 `Activity` 下多个 `Fragment` 之间共享 `ViewModel`
* 生命周期比 `Activity` 长：当配置发生改变导致 `Activity` 重建，但 `ViewModel` 并不会重建
* `ViewModel` 出现之前，`Activity ` 可以通过 `onSaveInstanceState()` 方法保存数据，然后从 `onCreate()` 或 `onRestoreInstanceState()` 中的 `Bundle` 恢复数据，但此方式仅适合可以序列化再反序列化的少量数据，`ViewModel` 的出现可以完美解决这个问题



>`ViewModel` 的创建

`ViewModel` 的创建都是通过 `ViewModelProvider` 完成的，`ViewModelProvider` 有如下一个重要的成员变量

````kotlin
// ViewModelProvider.kt

private val store: ViewModelStore
````

从命名可以看出来，`ViewModelStore` 是一个存储 `ViewModel` 的类，内部由 `Map` 实现

````kotlin
// ViewModelStore.kt

private val map = mutableMapOf<String, ViewModel>()
````

获取 `ViewModelStore` 只有一种方式：只有实现了 `ViewModelStoreOwner` 的类，才能提供 `ViewModelStore`

````kotlin
interface ViewModelStoreOwner {
    val viewModelStore: ViewModelStore
}
````

`ComponentActivity` 和 `Fragment` 都实现了 `ViewModelStoreOwner`



> `ViewModel` 的生命周期

生命周期比 `Activity` 长：当配置发生改变导致 `Activity` 重建，但 `ViewModel` 并不会重建

对于 `ComponentActivity` 来说，具体实现如下：

````java
public ComponentActivity() {
    //...
    getLifecycle().addObserver(new LifecycleEventObserver() {
        @Override
        public void onStateChanged(LifecycleOwner source,Lifecycle.Event event) {
            if (event == Lifecycle.Event.ON_DESTROY) {
                // Clear out the available context
                mContextAwareHelper.clearAvailableContext();
                // 如果不是配置改变导致的 Destory，则清空 ViewModelStore，最终会走到每个 ViewModel 的 clear()
                if (!isChangingConfigurations()) {
                    getViewModelStore().clear();
                }
            }
        }
    });
    //...
}
````

