#### 开发架构

`Android` 应用开发一般分为三部分：`UI` 逻辑、业务逻辑、数据逻辑，以获取数据并显示到界面上为例：

1. 点击获取数据按钮、展示加载框（`UI` 逻辑）
2. 封装请求、发送请求（业务逻辑）
3. 从本地或网络获取数据（数据逻辑）
4. 解析数据（业务逻辑）
5. 隐藏加载框、显示数据（`UI` 逻辑）

开发架构的出现是为了更好地协调这三者的关系，以达到：各模块职责分明、尽可能降低模块间的耦合度

***

#### MVC

* `Model`：数据来源，本地数据或网络数据等

* `View`：负责与用户交互、显示界面

* `Controllor`：处理业务逻辑

![Android_架构MVC.webp (384×200) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/Android/Android_架构MVC.webp) 

`MVC` 逻辑非常简单：

1. `View` 接受用户请求，将请求传递给 `Controller`
2. `Controller` 进行业务逻辑处理后，向 `Model` 获取数据
3. `Model` 数据更新后，通知 `View` 更新界面显示

**优点**

* 简单，不需要写过多代码来进行解耦

**缺点**

* 各层之间耦合度过高，不利于维护

***

#### MVP

* `Model`：数据来源，本地数据或网络数据等

* `View`：负责与用户交互，定义 `View` 接口向 `Presenter` 提供界面更新的方法

* `Presenter`：`View` 和 `Model` 之间的桥梁，接收来自 `View` 的请求，从`Model` 获得数据后通过 `View` 接口决定如何更新界面

![Android_架构MVP.webp (384×200) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/Android/Android_架构MVP.webp) 

`View` 与 `Presenter` 层通过接口相互持有：

* `View` 层通过接口向 `Presenter` 发送请求
* `Presenter` 层通过接口获取 `View` 层的用户输入、控制 `View` 层的界面更新

**优点**

* `View` 层与 `Model` 层之间完全解耦
* 各层之间的职责更加明确
* 便于测试，由于 `View` 与 `Presenter` 之间通过接口通信，且职责分明，所以不论时测试 `View` 层或是测试 `Presenter` 都比较方便

**缺点**

* 严格按照 `MVP` 规范会导致编写过多接口
* `Presenter` 层负责所有业务逻辑，业务过多可能会导致 `Presenter` 层过大
* `View` 与 `Presenter` 层通过接口相互持有可能会导致内存泄漏

***

#### MVVM

* `Model`：数据来源，本地数据或网络数据等

* `View`：负责与用户交互、监听 `ViewModel` 的数据变化，变化时做出相应的 `UI` 更新

* `ViewModel`：接收 `View` 的请求并处理业务逻辑，将能引起 `View` 层视图变化的数据保存到 `ViewModel`，通过类似于观察者的模式，当数据变化会通知 `View` 层，至于怎么更新界面 `ViewModel` 完全不关心

![Android_架构MVVM.webp (384×200) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/Android/Android_架构MVVM.webp) 

使用 `Jetpack` 中的 `LiveData` 与 `ViewModel` 可以简单地实现 `MVVM`

**优点**

* 相对于 `MVP` 来说，`ViewModel` 不会有 `View` 层的依赖
* 相对于 `MVP` 来说，减少许多接口编写
* 相对于 `MVP` 来说，`ViewModel` 完全不关心界面如何变化，只需要处理业务逻辑以及更新状态数据，职责更明确

**缺点**

* 如果视图状态较多，`ViewModel` 的构建和维护的成本都会比较高