# 概览



## 1. 布局

布局，即把界面中的控件以正确的尺寸摆放在正确的位置。作为软件开发者，通常我们是不需要关心这个布局过程，因为自带的控件的布局过程算法都是写好的。但是如果自带控件不能满足我的需求，那么就需要对布局过程进行自定义了。

布局分为两个阶段：

* 测量阶段：从上到下递归地调用每个 `View` 或 `ViewGroup` 的 `measure()`，测量它们的尺寸并计算它们的位置
* 布局阶段：从上到下递归地调用每个 `View` 或 `ViewGroup` 的 `layout()`，把测得的尺寸和位置赋值给它们

View 的布局过程（图片截取自[扔物线 HenCoder](https://www.bilibili.com/video/BV1vx411j7Qv/?t=5.832877&spm_id_from=333.1350.jump_directly&vd_source=52b0bdb6507cfe60388049de23d52f75)）：

![自定义View-View布局过程.png](https://github.com/WillisNotFound/Pic/blob/master/Android/%E8%87%AA%E5%AE%9A%E4%B9%89View-View%E5%B8%83%E5%B1%80%E8%BF%87%E7%A8%8B.png?raw=true)

ViewGroup 的布局过程（图片截取自[扔物线 HenCoder](https://www.bilibili.com/video/BV1vx411j7Qv/?t=5.832877&spm_id_from=333.1350.jump_directly&vd_source=52b0bdb6507cfe60388049de23d52f75)）：

![自定义View-ViewGroup布局过程.png](https://github.com/WillisNotFound/Pic/blob/master/Android/%E8%87%AA%E5%AE%9A%E4%B9%89View-ViewGroup%E5%B8%83%E5%B1%80%E8%BF%87%E7%A8%8B.png?raw=true)



## 2. 绘制

### 2.1 作用

作为软件开发者，控件绘制的内容都是由我们来控制的，但是一般情况下，我们并不是直接操作这些绘制内容，而是使用 API 对控件进行设置，然后控件自己会完成绘制内容。

例如，通过 `setText(String)` 为 `TextView` 设置文字内容，`TextView` 自己会负责把文字显示在合适的位置，至于文字的起始坐标在哪、文字从哪换行等细节我们都不用关注。

而自定义绘制就是由我们来接管这些细节的绘制过程，以此来完成一些自带控件显示不出的内容。

### 2.2 实现

Android 里的绘制是在每个 View 的绘制方法里发生的，一个 View 在绘制方法里写了什么代码，他就会绘制什么内容。

而自定义绘制则是重写绘制方法，插入自己的绘制代码来实现，绘制方法不是一个方法，由好几个，其中最常用的是 `onDraw(Canvas)`: 

````java
public class View {
    protected void onDraw(@NonNull Canvas canvas) {
    }
}
````

`onDraw(Canvas)` 这个方法负责 View 的主体绘制，例如 `TextView` 的文字、`ImageView` 的图像，具体执行绘制操作的是 `Canvas` 参数。

`Canvas` 翻译为画布，是画的载体。而 Android 中 `Canvas` 就是一个绘制工具，唯一的功能就是：绘制 。`Canvas` 提供了一系列 `drawXxx()` 来进行绘制：

````java
public void drawCircle(float cx, float cy, float radius, Paint paint) {
    ...
}

public void drawBitmap(Bitmap bitmap, float left, float top, Paint paint) {
    ...
}

public void drawRect(float left, float top, float right, float bottom, Paint paint) {
    ...
}

public void drawText(String text, float x, float y, Paint paint) {
    ...
}

...
````

 在这些 `drawXxx()` 的参数中， 除了绘制内容（例如 text）和位置信息（x、y），都有一种重要参数 `Paint`。

> `Paint` 翻译为颜料，是用来涂色的。而 Android 中的 `Paint` 是加强版的颜料，负责提供绘制使用的颜色和风格，这里的风格信息例如：绘制的圆是空心还是实心、线条的粗细是多少、有无阴影等等。

 另外，除了 `drawXxx()`，`Canvas` 还提供了其他方法用于辅助绘制，主要有两类：

* 绘制范围的裁切，`clipXxx()`
* 绘制内容的几何变换，Matrix

绘制具有顺序，先绘制的内容会被后绘制的内容覆盖，而 `onDraw()` 仅负责主体内容的绘制，还有其他方法用来绘制背景、前景、整体内容。



## 3. 动画

> 动画，即对内容的两个状态进行平滑的过渡

### 3.1 属性动画

属性动画的本质：**不断更新 View 的属性，让 View 表现出动画效果**















## 4. 触摸反馈