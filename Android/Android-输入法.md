#### 输入法框架

`Android Input Method Framework (IMF)`，`IMF` 由三部分组成：

* <u>普通应用程序 `(Client)`</u>

  在日常开发中，点击 `EditText` 会唤起当前系统选中的输入法，在输入法中输入字符、提交候选词则会更新到 `EditText` 中，这其中就涉及到了 `EditText` 与输入法服务以及输入法管理服务之间的通信

* <u>输入法服务 `Input Method (IME)`</u>

  输入法服务由系统或其他输入法厂商提供，例如百度输入法、搜狗输入法等，这是一个用户级别的 `Service`，对于输入法开发者来说，`IMF` 提供了 `InputMethodService` 来进行扩展

* <u>输入法管理服务 `Input Method Manager (IMM)`</u>

  这是一个系统级别的 `Service`，用于管理系统内所有的输入法以及与其他系统服务（例如 `WindowManagerService`）进行交互

  

而对于普通开发者来说，与输入法的交互最多就是如下两点：

* 控制输入法的显示与隐藏
* 输入法唤起时当前布局的调整

***

#### 显示与隐藏

```kotlin
/**
 * 显示软键盘，只有当前 Window 以及当前 View 具有焦点才能显示软键盘
 */
fun View.showKeyboard() {
    isFocusable = true
    val inputMethodManager = context.getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
    if (requestFocus()) inputMethodManager.showSoftInput(this, 0)
}

/**
 * 隐藏软键盘
 */
fun View.hideKeyboard() {
    val inputMethodManager = context.getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
    inputMethodManager.hideSoftInputFromWindow(windowToken, 0)
}
```

这里的 `InputMethodManager` 是 `Client` 与 `IME`、`IMM` 通信的桥梁

上面的方式是手动控制软键盘的显示与隐藏，除此之外还可以设置 `Window` 的 `SoftInputMode` 中的 `STATE_XXX` 属性以控制窗口获取焦点时软键盘的显示与隐藏，该方式用的不多，这里不介绍了

***

#### 布局的调整

首先，软键盘本质上是一个系统级别的 `Dialog`，可以从 `InputManagerService#onCreate()` 中看出

````java
// InputManagerSrvice.java

SoftInputWindow mWindow;

@Override public void onCreate() {
    // ...
    
    // SoftInputWindow 继承自 Dialog
    mWindow = new SoftInputWindow(this, mTheme, mDispatcherState);
    // 设置 Dialog 所附属 Window 的属性
    final Window window = mWindow.getWindow();
    final WindowManager.LayoutParams lp = window.getAttributes();
    lp.setTitle("InputMethod");
    lp.type = WindowManager.LayoutParams.TYPE_INPUT_METHOD;
    lp.width = WindowManager.LayoutParams.MATCH_PARENT;
    lp.height = WindowManager.LayoutParams.WRAP_CONTENT;
    lp.gravity = Gravity.BOTTOM;
    window.setAttributes(lp);

    final int windowFlags = WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN
        | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
        | WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS;
    final int windowFlagsMask = windowFlags
        | WindowManager.LayoutParams.FLAG_DIM_BEHIND; 
    window.setFlags(windowFlags, windowFlagsMask);
    
    // ...
}
````

通常设置 `Window` 的 `SoftInputMode` 中的 `ADJUST_XXX` 属性以控制软键盘显示时原窗口的布局调整行为，一共有四种配置

* `ADJUST_NOTHING`：软键盘弹起时，原窗口不做任何调整
* `ADJUST_RESIZE`：软键盘弹起时，若得到焦点的 `View` 被遮挡，则调整 `DecorView` 的 `Padding` 值
* `ADJUST_PAN`：软键盘弹起时，若得到焦点的 `View` 被遮挡，则整个 `ViewTree` 发生位移
* `ADJUST_UNSPECIFIED`：默认配置，若得到焦点的 `View` 被遮挡，则根据实际情况判断是采用 `ADJUST_RESIZE` 还是 `ADJUST_PAN`
  * 若当前可见区域存在可滚动 `View`，则采用 `ADJUST_RESIZE`
  * 否则，采用 `ADJUST_PAN`

***

#### 参考文章

* [Android-软键盘一招搞定(实践篇) - 掘金 (juejin.cn)](https://juejin.cn/post/7012844100994990087)
* [Android-软键盘一招搞定(原理篇) - 掘金 (juejin.cn)](https://juejin.cn/post/7012847888753491976)
* [Android软键盘的全面解析，让你不再怕控件被遮盖 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903489051557902)
* [Android输入法框架分析(1)－三大组件 - 西城铁 - 博客园 (cnblogs.com)](https://www.cnblogs.com/xichengtie/p/3557226.html)

