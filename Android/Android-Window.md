#### Window

`Android` 手机展示的界面是由一层层 `Window` 叠加而成的，对于平时开发接触最多的 `View`，实质上也是由 `Window` 进行管理，每个 `Window` 都管理着一课 `View Tree`，根节点为 `DecorView`

以 `Activity` 为例，`Window` 没有生命周期这个说法，在 `Window` 外面包一层 `Activity` 就可以决定什么时候展示画面、什么时候隐藏画面



`Window` 的属性由 `WindowManager` 的静态内部类 `LayoutParams` 定义，以下是常见的几个属性：

* `type`：整型值，表示 `Window` 的层级，层级越大的 `Window` 会显示在越上层
* `flag`：整型值，每一位代表一种配置，用于控制 `Window` 的显示以及触摸事件的处理逻辑
* `softInputMode`：处理软键盘弹起时 `Window` 的显示逻辑
* `x/y`：指定 `Window` 的位置
* `alpha`：`Window` 的透明度
* `dimAmount`：当设置了 `FLAG_DIM_BEHIND` 时，`Window` 后面内容的亮度，范围从 1.0（明亮）到 0.0（黑暗）
* `gravity`：`Window` 在屏幕中的位置

***

#### WindowManager

这里关注 `WindowManager` 的三个主要功能：添加 `Window`、更新 `Window`、删除 `Window`，这三个功能对应的是 `ViewManager` 定义的一组方法

````java
//ps.这里关注的在是WindowManager中这些方法的作用
public interface ViewManager {
    /**
     * 添加Window，这里的view是View Tree的根节点
     */
    public void addView(View view,ViewGroup.LayoutParams params);
    
    /**
     * 更新Window
     */
    public void updateViewLayout(View view,ViewGroup.LayoutParams params);
    
    /**
     * 删除Window
     */
    public void removeView(View view); 
}
````

这三个方法的最终实现在 `WindowManagerGlobal` 中，`WindowManagerGlobal` 是全局单例，负责管理 `APP` 的所有 `Window`

````java
public final class WindowManagerGlobal {
    // 存放所有Window的View Tree的根节点
    private final ArrayList<View> mViews = new ArrayList<View>();
    // 存放所有Window对应的ViewRootImpl对象
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    // 存放所有Window的LayoutParams
    private final ArrayList<WindowManager.LayoutParams> mParams = new ArrayList<WindowManager.LayoutParams>();
}
````

`WindowManagerGlobal#addView()`：

````java
public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    ViewRootImpl root;
    View panelParentView = null;
    synchronized (mLock) {
        // 这里实例化一个ViewRootImpl对象
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        // 存储View、ViewRootImpl、LayoutParams
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
        try {
            // 最终通过ViewRootImpl与WMS跨进程通信，将Window添加到屏幕上
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            //...
        }
    }
}
````

`WindowManagerGlobal#updateViewLayout()`

````java
public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
    //将params更新到view中
    view.setLayoutParams(wparams);
    synchronized (mLock) {
        //获取对应的下标
        int index = findViewLocked(view, true);
        ViewRootImpl root = mRoots.get(index);
        mParams.remove(index);
        mParams.add(index, wparams);
        //ViewRootImpl对View进行重新绘制
        root.setLayoutParams(wparams, false);
    }
}
````

`WindowManagerGlobal#removeViewLayout()`

````java
public void removeView(View view, boolean immediate) {
    synchronized (mLock) {
        int index = findViewLocked(view, true);
        View curView = mRoots.get(index).getView();
        removeViewLocked(index, immediate);
        if (curView == view) {
            return;
        }
    }
}
````

***

#### ViewRootImpl

`ViewRootImpl` 可以理解为 `Window` 中 `View Tree` 根节点的管理者，所以它是真正管理整颗 `View Tree` 的

对于 `APP` 来说，事件分发以及 `View` 绘制流程的起点都在 `ViewRootImpl` 中

****

#### 参考文章

* [Android的Window详解 - 掘金 (juejin.cn)](https://juejin.cn/post/7119004719892135966)
* [Android全面解析之Window机制 - 掘金 (juejin.cn)](https://juejin.cn/post/6888688477714841608)