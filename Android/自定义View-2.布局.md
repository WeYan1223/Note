# 1.View

<img src="https://github.com/WillisNotFound/Pic/blob/master/Android/%E8%87%AA%E5%AE%9A%E4%B9%89View-View%E5%B8%83%E5%B1%80%E8%BF%87%E7%A8%8B.png?raw=true" alt="自定义View-View布局过程.png"  /> 

布局分为两个阶段：

* 测量阶段：从上到下递归地调用每个 `View` 或 `ViewGroup` 的 `measure()`，测量它们的尺寸并计算它们的位置
* 布局阶段：从上到下递归地调用每个 `View` 或 `ViewGroup` 的 `layout()`，把测得的尺寸和位置赋值给它们

对于 View 来说，只需要关注测量阶段

### 1.1测量阶段

> 自定义 View 的测量过程：重写 `onMeasure()` 来计算并保存尺寸

View 的测量，即按照 View 的内部逻辑以及**父 View 的约束**来计算出 View 的宽度和高度，并把计算后的宽度和高度保存下来，从代码的角度看如下：

````kotlin
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec)
    // 按照View的内部逻辑以及父View对View的约束来计算出 View 的宽度和高度
    val width = ...
    val height = ...
    // 把计算后的宽度和高度保存下来
    setMeasuredDimension(width, height)
}
````

其中**父 View 的约束**指的是在 View 的绘制过程中，父 View 对 View 测量宽度和高度的限制，在代码中的体现是 `onMeasure()` 中的两个参数：

* `widthMeasureSpec`：对测量宽度的限制
* `heightMeasureSpec`：对测量高度的限制

> `MeasureSpec` 是一个 32 位的整形，高 2 位代表约束模式，低 30 位代表约束大小
>
> ````java
> public static class MeasureSpec {
>     // 给定约束大小和约束模式构造MeasureSpec
>     public static int makeMeasureSpec(int size, int mode) {
>         ...
>     }
> 
>     // 获取Measure的约束模式
>     public static int getMode(int measureSpec) {
>         ...
>     }
> 
>     // 获取Measure的约束大小
>     public static int getSize(int measureSpec) {
>         ...
>     }
> }
> ````
>
> 其中，一共有三种约束模式
>
> ```java
> public static class MeasureSpec {
>     // 不限制
>     public static final int UNSPECIFIED = ...;
> 
>     // 限制为精确值，精确值为约束大小
>     public static final int EXACTLY     = ...;
> 
>     // 设置上限，上限为约束大小
>     public static final int AT_MOST     = ...;
> }
> ```

View 在测量过程中必须遵守父 View 的约束（不遵守的话可能会出 bug），遵守父 View 约束主要有两种写法：

* 按照 View 的内部逻辑计算的同时，考虑三种约束模式
* 按照 View 的内部逻辑计算之后，调用 `View.resolveSize(int size, int measureSpec)`，但这是一个傻瓜式的修正

计算结束后调用 `setMeasuredDimension()` 保存计算结果，该方法的最终实现如下：

````java
private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;
}
````

所以，`mMeasuredWidth` 和 `mMeasuredHeight` 的值是在 View 测量完成后才存在



# 2.ViewGroup

![自定义View-ViewGroup布局过程.png](https://github.com/WillisNotFound/Pic/blob/master/Android/%E8%87%AA%E5%AE%9A%E4%B9%89View-ViewGroup%E5%B8%83%E5%B1%80%E8%BF%87%E7%A8%8B.png?raw=true)

布局分为两个阶段：

* 测量阶段：从上到下递归地调用每个 `View` 或 `ViewGroup` 的 `measure()`，测量它们的尺寸并计算它们的位置
* 布局阶段：从上到下递归地调用每个 `View` 或 `ViewGroup` 的 `layout()`，把测得的尺寸和位置赋值给它们

### 2.1测量阶段

> 自定义 ViewGroup 的测量过程：重写 `onMeasure()`，先测量子 View 的尺寸，再计算自己的尺寸并保存

`onMeasure()` 的重写分三步：

1. 调用每个子 View 的 `measure()`，先进行子 View 的测量过程
2. 根据子 View 的测量尺寸，得出子 View 的位置，并保存子 View 的位置和尺寸（保存不是必须的）
3. 根据子 View 的位置和尺寸计算出自己的尺寸，并调用 `setMeasuredDimension()` 保存

第一步的重点在于如何确定 `measure(int widthMeasureSpec, int heightMeasureSpec)` 的两个参数，通常需要根据 ViewGroup 的可用空间和子 View 的 `LayoutParams` 来进行计算，具体细节参考下面的 `FrameLayout` 源码解析。

在第一步子 View 测量完成后，ViewGroup 就可以获取到子 View 的 `measureWidth` 和 `measureHeight` 了，当所有子 View 都测量完成，ViewGroup 就可以计算自己的尺寸并保存了。

### 2.2布局阶段

> 自定义 ViewGroup 的布局过程：重写 `onLayout()` 来摆放子 View

布局阶段 ViewGroup 只需要做一件事：调用所有子 View 的 `layout(int l, int t, int r, int b)`

### 2.3FrameLayout源码

> 参考[Android常用Layout源码总结—FrameLayout](https://juejin.cn/post/6844904065579614222)

