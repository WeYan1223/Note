#### Flow

> 引入 `Flow`

`Flow` 于 [2019 年发布](https://www.youtube.com/watch?v=tYcqn48SMT8)，属于 `Kotlin` 协程库的一部分，在项目中需要依赖[协程库](https://github.com/Kotlin/kotlinx.coroutines/blob/master/README.md)：

````kotlin
// 所有平台的通用协程库
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
// Android 平台的通用协程库，当中已经包含了上面的 coroutines-core
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
````

