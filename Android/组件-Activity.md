#### 简述

`Activity` 是 `Android` 应用的关键组件，提供了窗口给应用在其中绘制界面

对于移动应用，用户与应用的交互并不总是在同一位置开始，例如：如果从主屏幕打开电子邮件应用，可能会进入邮件列表界面，而如果通过社交媒体应用启动电子邮件应用，则可能会直接进入邮件撰写界面，`Activity` 的目的就是促进这种实现

***

#### 启动方式

##### 1. 显式启动

> 显式启动通常用于同一应用的 `Activity` 启动

指定要启动的 `Activity` 类名

````java
Intent intent = new Intent(context, MyActivity.class);
startActivity(intent);
````

##### 2. 隐式启动

> 隐式启动更多用于不同应用的 `Activity` 启动

显示启动与隐式启动的区别：

* 显示启动：告诉系统 “在当前应用启动发送电子邮件 `Activity`”
* 隐式启动：告诉系统 “在任何应用中启动能够发送电子邮件的 `Activity`”（如果有多个，会弹出系统界面询问启动哪个）

要使用此功能，您需要在 `<activity>` 元素中声明 `<intent-filter>` 属性（可以声明多个 `<intent-filter>`）

* [`action`](https://developer.android.com/guide/topics/manifest/action-element?hl=zh-cn)：表示该 `Activity` 的作用，可以声明多个，只需匹配其中一个即可
* [`category`](https://developer.android.com/guide/topics/manifest/category-element?hl=zh-cn)：表示 `Activity` 的附加信息，可以声明多个，但必须全部匹配
* [`data`](https://developer.android.com/guide/topics/manifest/data-element?hl=zh-cn)：表示该 `Activity` 接受的数据类型，可以声明多个，只需匹配其中一个即可

***

#### 生命周期

> * `onCreate` 与 `onDestory` ，分别标 `Activity` 的<u>创建与销毁</u>，并且只可能有一次调用
> * `onStart`  与 `onStop` ，分别标识着 `Activity` 的<u>可见与不可见</u>，可能被多次调用
> * `onResume` 与 `onPause` ，分别标识着 `Activity` 的<u>可交互与不可交互</u>，可能被多次调用

![Activity_生命周期.webp (645×751) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/Android/Activity_生命周期.webp) 

* `onCreate`：`Activity` 正在被创建，此时可以做一些初始化工作，比如：加载界面布局资源、初始化 `Activity` 所需数据等等
* `onRestart`：`Activity` 正在重新启动（当 `Activity` 由不可见重新变为可见状态时，`onStart()` 会被调用）
* `onStart`：`Activity` 正在启动（此时 `Activity` 已经可见了，但无法与用户进行交互）
* `OnResume`：`Activity` 可见并且可以与用户进行交互
* `onPause`：`Activity` 正在停止，此时可以执行存储数据、停止动画等工作，但不能太耗时（原因：必须等旧 `Activity` 的 `onPause()` 执行完，新 `Activity` 的 `onResume()` 才会执行）
* `onStop`：`Activity` 即将停止，此时可以执行一些稍微重量级的回收工作，同样不能太耗时
* `onDestory`：`Activity` 即将被销毁，此时需要执行一些回收工作以及最终的资源释放

##### 1. 正常生命周期

* 启动 `Activity A`
* 在 `Activity A` 按下 `HOME` 键
* 在 `Activity A` 按下返回键
* 在 `Activity A` 启动 `Activity B`
* 在 `Activity B` 按下返回键

##### 2. 异常生命周期

* <u>系统配置改变</u>：

  当系统配置发生改变后，`Activity` 会被销毁并重建，此时属于在异常情况下终止，系统会回调 `onSaveInstanceState()` 方法用于保存状态，该方法的调用时机是 `onStop()` 之前且与 `onPause()` 没有确定的时序关系

  `Activity` 重建后，系统会把 `onSaveInstanceState()` 中保存的 `Bundle` 对象作为参数传递给 `onCreate()` 与 `onRestoreInstanceState()`，从时序关系看，`onRestoreInstanceState()` 的调用时机在 `onStart()` 之后

  虽然说 `onCreate()` 与 `onRestoreInstanceState()` 都有 `Bundle` 参数，但是后者一旦被调用，那么 `Bundle` 参数一定是有值的，而前者则不一定，所以官方是建议在 `onRestoreInstanceState()` 中进行数据恢复

  如果想要系统配置改变后不重建 `Activity`，需要为 `<activity>` 指定 [`configChanges`](https://developer.android.com/guide/topics/manifest/activity-element?hl=zh-cn#config) 属性，多个属性之间使用 `|` 隔开，同时在 `Activity` 中重写 `onConfigurationChanged()`，当忽略的系统配置发生更改时，会回调该方法，此时需要做一些相应的处理

* <u>内存资源不足</u>：

  资源内存不足会导致优先级较低的 `Activity` 被杀死，其数据保存与恢复过程与上述一致

  1. 前台 `Activity`，该 `Activity` 正在和用户进行交互，优先级最高

  2. 可见但非前台 `Activity`，该 `Activity` 可见但是位于后台无法与用户进行交互

  3. 后台 `Activity`，该 `Activity` 已经执行了 `onStop()` 方法，优先级最低

***

#### 任务和返回栈

任务是一系列 `Activity` 的集合，使用栈的方式管理其中的 `Activity`，这个栈被称为返回栈

默认情况下，同一应用的 `Activity` 所属的任务的名称为应用包名，当让可以通过  `<activity>` 指定 `taskAffinity` 属性进行修改

默认情况下，启动的 `Activity` 都放入到一个相同的任务当中，通过一个先进后出的任务栈进行管理，当燃也可以打破这种默认行为，通过在设置 `<activity>` 元素的属性，或者是在启动 `Activity` 时配置 `Intent` 的 `flag` 来实现，具体参考[管理任务](https://developer.android.com/guide/components/activities/tasks-and-back-stack?hl=zh-cn#ManagingTasks)

`Activity` 的四种启动模式：

* `standard`：默认模式，每次启动都会创建新的 `Activity` 实例入栈

* `singleTop`：如果当前任务的栈顶已存在 `Activity` 的实例，则系统会通过调用其 `onNewIntent()` 方法，而不是创建 `Activity` 的新实例

  应用场景：

  * 防止快速点击按钮启动多个 `Activity`
* 消息推送界面：接收到多条推送消息，点开不同消息，均由同一 `Activity` 展示
  
* `singleTask`：如果当前或其他任务的返回栈中已存在 `Activity` 实例，则系统会通过调用其 `onNewIntent()` 方法同时将该 `Activity` 以上的所有 `Activity` 都销毁

  应用场景：

  * 退出整个应用，将主 `Activity` 设为 `singleTask` 模式并重写其 `onNewIntent()` 方法，加上 `finish()` 结束

* `singleInstance`：与 `singleTask` 类似，唯一不同在于该模式下的 `Activity` 只能单独位于一个任务中

  应用场景：
  
  * 呼叫来电界面
  * 社交 `APP` 分享页面

***

#### 启动流程

> 由于 `APP` 的启动流程中包含了 `Activity` 启动流程，所以这里分析 `APP` 的启动流程

1. 在桌面点击图标启动应用，由于桌面属于 `Launcher` 进程的 `Activity`，所以本质上也会调用 `Activity.startActivity()`，最终都会进入 `Instrumentation.execStartActivity()` 中，该方法主要完成如下任务：
   * 将启动 `Activity` 请求发送给 `AMS`（其中的信息主要包括应用包名、`Activity` 名称）
   * `AMS` 处理该请求并返回一个 `int` 结果，通过该 `int` 值判断，如果启动失败则抛出相应的异常，常见的有 `ActivityNotFoundException`
2. `AMS` 收到请求后，发现应用并没有启动，则让 `zygote` 进程 `fork` 出新的子进程
3. 子进程启动后调用 `ActivityThread.main()`，该方法主要做了以下事情：
   * 与 `AMS` 进行关联，把 `IApplicationThread` 对象传到 `AMS`，使得 `AMS` 可以作为客户端与该应用进行通信
   * 初始化 `Application`、`Instrumentation` 对象
   * 开启 `Handle` 消息循环
4. `AMS` 向应用发送 `SCHEDULE_LAUNCH_ACTIVITY` 命令启动 `Activity`，对于应用来说起点为 `performLaunchActivity()`，在这里会通过 `Instrumentation` 对象通过反射创建 `Activity`，并回调 `onCreate()`



该启动流程涉及了如下两个重要的类：

* `Instrumentation`：
  * 负责真正地创建 `Application` 和 `Activity` 以及生命周期的切换
  * 应用只能有一个 `Instrumentation` 实例，同时每个 `Activity` 都会持有该实例
  * 是 `Activity` 与 `AMS` 之间的桥梁
* `ActivityManagerService`：
  * 管理系统中所有 `Activity` 的生命周期
  * 管理任务和返回栈

***





#### 面试题

> A Activity 打开 B Activity 时都有哪些生命周期回调？

需要考虑`B Activity` 的启动模式：

`standard`

* 生命周期回调为：`A.onPause` => `B.onCreate` => `B.onStart` => `B.onResume` => `A.onStop`

`singleTop`

* `B Activity` 不在栈顶：与 `standard` 模式一样
* `B Activity` 在栈顶：`B.onPause` => `B.onNewIntent` => `B.onResume`

`singleTask`

* 任务栈中存在 `B Activity` ：`A.onPause` => `B.onNewIntent` => `B.onRestart` => `B.onStart` => `B.onResume` => `A.onStop` => `A.onDestory`
* 任务栈中不存在 `B Activity` ：与 `standard` 模式一样

`singleInstance`

* 与 `singleTask` 类似

> 弹出 Dialog 对生命周期有什么影响？

首先，生命周期回调都是 `AMS` 通过 `Binder` 通知应用进程的

而弹出 `Dialog`、`Toast` 、`PopupWindow` 本质上都是通过 `WindowManager.addView()` 显示的

所以，弹出 `Dialog` 不会对生命周期有任何影响

而如果是在 `A Activity` 启动一个 `Theme` 为 `Dialog` 的 `B Activity`，则生命周期为：`A.onPause` => `B.onCreate` => `B.onStart` => `B.onResume`

注意，这里 `A Activity` 不会回调 `onStop`，因为只有在 `Activity` 切到后台不可见时才会回调 `onStop`，而弹出 `Theme` 为 `Dialog` 的 `B Activity` 时，`A Activity` 还是可见的，只是失去了焦点而已

启动透明的 `Activity` 的生命周期分析也是一样

> 为什么 Activity 在 onResume 之后才显示？

[来聊聊Activity的显示原理](https://juejin.cn/post/6844904086265937934)

> onActivityResult 在哪两个生命周期之间回调

`onActivityResult` 不属于 `Activity` 的生命周期

如果 `A Activity` 通过 `startActivityForResult()` 启动 `B Activity`，当 `B Activity` 返回时，生命周期如下：`B.onPause()` => `A.onActivityResult()` => `A.onRestart()` => `A.onStart()` => `A.onResume()` => `B.onStop` => `B.onDestory`

> onCreate() 里写死循环会 ANR 吗？

如果 `Android` 应用的主线程处于阻塞状态的时间过长，会触发应用无响应(`ANR`) 错误

`Android` 中产生 `ANR` 的原因

* 当 `activity` 位于前台时，且应用在 5 秒钟内未响应输入事件或 `BroadcastReceiver`（如按键或屏幕轻触事件）
* 虽然前台没有 `activity`，但 `BroadcastReceiver` 用了相当长的时间仍未执行完毕

所以，在 `onCreate()` 写死循环不会发生 `ANR`，但是主线程被死循环一直占用了，所以当再有其他事件产生时，就不能及时响应了，从而导致 `ANR` 发生

