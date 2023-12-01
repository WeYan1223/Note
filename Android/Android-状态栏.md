#### 状态栏

> 从 `Window` 的角度看，状态栏是一个层级为 `TYPE_STATUS_BAR` 的系统 `Window`

在日常开发中，与状态栏相关的交互无非就是下面几个：

* 状态栏显示控制
* 状态栏背景颜色控制
* 状态栏字体颜色控制
* 状态栏是否覆盖布局

从 `Android SDK 30` 开始，利用新 `API` 实现以上功能非常简单，参考[【精选】Android 状态栏常规操作(状态栏显示，状态栏颜色，沉浸式状态栏)_安卓状态栏apk-CSDN博客](https://blog.csdn.net/Wait_For_Loading/article/details/119330990)，这里介绍的方式适用于 `API 24` 到 `API 29`

##### 1. 显示控制



##### 2. 背景颜色控制

`Window` 提供了 `setStatusBatColor(int color)` 设置状态栏背景颜色，但是需要先为 `Window` 添加 `FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS` 标志，并且移除 `FLAG_TRANSLUCENT_STATUS` 才能生效

```kotlin
window.run {
    addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS)
    clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS)
    statusBarColor = Color.TRANSPARENT
}
```

##### 3. 字体颜色控制

`Android 6.0` 开始，提供了 `View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR` 标志位，将状态栏文字和图标颜色改为黑色

````java
public static final int SYSTEM_UI_FLAG_LIGHT_STATUS_BAR = 0x00002000;
````

注意，该标志位要生效，必须先为 `Window` 添加 `FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS` 标志，并且移除 `FLAG_TRANSLUCENT_STATUS`

```kotlin
window.run {
    addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS)
    clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS)

    val v = decorView.systemUiVisibility
    decorView.systemUiVisibility = v or View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR
}
```

##### 4. 是否覆盖布局

只需为 `Window` 添加 `FLAG_TRANSLUCENT_STATUS` 即可

```kotlin
window.addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS)
```

当添加该标志时，`View.SYSTEM_UI_FLAG_LAYOUT_STABLE` 和 `View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN` 会被自动添加，状态栏的颜色会在不同机型以及 `Android` 版本中有所不同

布局被状态栏覆盖，有可能导致布局内容也被状态栏覆盖，此时最简单的办法是设置 `fitSystemWindows`

````kotlin
findViewById<FrameLayout>(android.R.id.content).getChildAt(0).fitsSystemWindows = true
````

***

#### 参考文章

* [[Android必备基础\]之状态栏处理 - 掘金 (juejin.cn)](https://juejin.cn/post/6895923667529072648)
* [Android 状态栏常规操作(状态栏显示，状态栏颜色，沉浸式状态栏)_安卓状态栏apk-CSDN博客](https://blog.csdn.net/Wait_For_Loading/article/details/119330990)
* [隐藏状态栏  | Android 开发者  | Android Developers](https://developer.android.com/training/system-ui/status?hl=zh-cn)

