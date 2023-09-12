#### KMM

> 官方文档：[Get started with Kotlin Multiplatform for mobile | Kotlin Documentation (kotlinlang.org)](https://kotlinlang.org/docs/multiplatform-mobile-getting-started.html)

`KMM (Kotlin Multiplatform Mobile)`，一套基于 `Kotlin` 的移动端跨平台框架，使 `IOS` 和 `Android` 应用程序之间共享业务逻辑代码，并在必要时可编写平台特定的代码，它的特点是结合了跨平台和原生开发的一种开发模式，简单的理解就是：从纯原生开发变成了 `KMM` + 原生 `UI` 开发

<img src="https://kotlinlang.org/lp/multiplatform/static/e35e8cfc124cd840e3b6d6a6416aa5fd/5a190/baidu-1.png" alt="baidu-1.png (800×409) (kotlinlang.org)" style="zoom:80%;" />  

`KMM` 的出现可以较好地解决以下痛点：

* 编码成本高，需要降本增效：相同的业务逻辑两端需用不同的语言来编码、测试 ，而且很难做到逻辑的实现完全一致
* 后期维护、测试成本较高：业务逻辑变更，需要拉上两端的研发都对齐一遍，然后又各自编码、测试

应用场景：

* 业务逻辑与基建代码进行跨平台，例如网络请求、数据存储，埋点上报等



> `KMM` 项目结构

主要有三个部分：

* `Common` 目录：存放 `Android` 和 `iOS` 公用的代码
* `Android` 目录：编写 `Android` 平台特定的代码
* `iOS` 目录：编写 `iOS` 平台特定的代码



> `KMM` 怎么跨平台

* 将 `Common` 与 `Android` 两个目录下的代码合并编译打包为 `Android` 平台产物（`aar` 文件）
* 将 `Common` 与 `iOS` 两个目录下的代码合并编译打包为 `iOS` 平台的产物（`framework` 文件）



> 怎么使用 `KMM` 项目

对于 `Android` 端来说，集成 `KMM` 项目是非常简单的，跟普通的 `Android` 第三方库集成无异

把 `KMM` 项目编译打包后的 `AAR` 文件发布到远程代码仓库，在 `Android` 项目中对它进行依赖即可