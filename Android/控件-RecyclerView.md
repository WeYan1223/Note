#### RecyclerView

> `RecyclerView` 是什么？

* `RecyclerView` 是 `Android` 常用的列表展示控件
* 使用方面，`RecyclerView` 提供的适配器模式使得我们只需要关注如何将数据转换为 `ViewHolder` 即可
* 性能方面，`RecyclerView` 具有高效的回收复用机制，以提高列表的性能



> `RecyclerView` 回收复用什么？

`RecyclerView` 回收复用的为 `ViewHolder` 对象

`ViewHolder` 主要保存：要展示的 `View` 、在 `RecyclerView` 中的位置信息、自身状态信息

`RecyclerView` 回收复用的核心思想是：对于 `ViewHolder` 来说，能不创建就不创建，能不重新绑定就不重新绑定，尽可能减少不必要的重复工作



> `ViewHolder` 被回收到哪？从哪获取复用的 `ViewHodler`？

`RecyclerView` 的回收复用机制由内部类 `Recycler` 提供，主要有以下核心成员变量

````java
public final class Recycler {
    // 用于屏幕内 ViewHodler 快速复用，不需要重新bindViewHodler
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
    // 用于屏幕内 ViewHodler 快速复用，需要重新bindViewHodler
    ArrayList<ViewHolder> mChangedScrap = null;

    // 保存最近移出屏幕的ViewHolder，默认大小是2（先进先出）
    // 应用场景在那些需要来回滑动的列表中，能直接复用ViewHolder，不需要重新bindViewHodler
    final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();

    // 自定义缓存，默认不实现
    private ViewCacheExtension mViewCacheExtension;
    
    // 看作是Map，key-ViewType，value-List<ViewHolder>
    // 当mCachedViews满，将移出的ViewHolder清除数据并放到Pool中，所以复用时需要重新bindViewHodler
    RecycledViewPool mRecyclerPool;
}
````



> `RecyclerView` 何时回收？何时复用？

简单来说：

在 `RecyclerView` 重新布局 `onLayoutChildren()` 或者填充布局 `fill()` 时，会先把必要的 `ViewHolder` 与屏幕分离或者移除，并做好标记，保存到 `list` 中，在重新布局时，再将 `ViewHolde` 拿出来重新一个个放到新的位置上去

滚动场景：

1. 手指滑动屏幕
2. 将处于屏幕内的 `ViewHodler` 存放到 `AttachScrap/ChangeScrap` 
3. 将即将移除屏幕的 `ViewHodler` 存放到 `CacheView`，若 `CacheView` 满了，则按照先进先出将移出的 `ViewHolder` 清除数据并放入 `RecyclerViewPool` 中
4. 对 `RecyclerView` 内的元素重新布局，获取 `ViewHolder` 的优先级为 `AttachScrap > CachedViews > RecycledViewPool`，获取不到就重新创建

数据更新场景：



***

#### 参考文章

* [深入理解 RecyclerView 的回收复用缓存机制详解（匠心巨作-下）](https://juejin.cn/post/6984974879296585764)

