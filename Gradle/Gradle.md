#### 基本结构

> 对于使用 `Gradle` 构建的项目，其根目录下通常有如下结构

````sh
.
├── a-subproject
│   └── build.gradle
├── build.gradle
├── settings.gradle
├── gradle.properties
├── local.properties
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
└── gradlew.bat
````

**build.gradle(项目级)**

> 定义所有子模块公共的配置参数

根目录的 `build.gradle` 又被称为 `root build script`

一般来说，`root build script` 并不是一个实际的模块, 而是用于对子模块进行统一的配置, 所以一般不会有太多的内容，甚至可以为空

**build.gradle(模块级)**

> 定义子模块的配置参数，可以覆盖项目级 `build.gradle` 中定义的配置参数

模块目录的 `build.gradle` 又被称为 `module build script`

`Build script` 作为整个 `Gradle` 中配置最复杂的脚本，实际上仅仅做了两件事：

* 引入插件
* 配置属性

在 `build.gradle` 中应用了某个插件, 就可以使用插件提供的 `DSL` 对其进行配置, 从而影响该模块的构建过程

**setting.gradle**

> 确定项目中有哪些模块参与构建

所有需要被构建的模块都需要在 `settings.gradle` 中注册, 因此它的作用是描述<u>当前构建所参与的模块</u>

**gradle.properties**

> 项目级 `Gradle` 参数配置项

通常在这里配置一些统一的版本号，如 `minSdkVersion`、`targetSdkVersion` 等，在各个子模块中获取以实现复用和统一

注意，不同地方配置的参数优先级不同：命令行参数 > `gradle.properties` 配置的参数 > `Android Studio` 设置中的参数

总的来说，`gradle.properties` 就是一个参数配置文件，与在命令行传递参数是一样的效果

**local.properties**

> 本地属性配置（不加入版本控制）

这里通常用来配置一些不可加入版本控制的信息，例如：`SDK` 安装目录、签名文件位置

**Gradle Wrapper**

> `Gradle Wrapper` 是 `Gradle` 项目自带的工具，主要有以下优点：
>
> * 确保项目在不同机器上 `Gradle` 版本的正确性
> * 减少本机手动安装 `Gradle` 环境的工作量

* `gradle-wrapper.jar`：包含脚本文件执行中需要用到的类库
* `gradle-wrapper.properties`：指定 `Gradle` 版本以及缓存目录
* `gradlew`：`Linux/Mac` 系统的可执行文件，用于执行 `Gradle Wrapper`
* `gradlew.bat`：`Windows` 系统的可执行文件，用于执行 `Gradle Wrapper`

`Gradle Wrapper` 的使用方式类似于本机的 `Gradle`，例如

本机 `Gradle` 查看版本：

````sh
gradle --version
````

`Gradle Wrapper` 查看版本（`Windows` 系统）：

````sh
./gradlew.bat --version
````

如果 `Gradle` 缓存目录不存在当前 `Gradle Wrapper` 需要的 `Gradle` 版本，则会先下载对应版本的 `Gradle`

***

#### 简单介绍

要运行 `Java` 项目，最简单来说有以下步骤：

1. 使用 `javac` 命令将 `.java` 文件编译为 `.class` 字节码文件
2. 使用 `java` 命令运行 `.class` 字节码文件 

有了 `Gradle`，通常我们只需单击一下按钮，以上操作便将自动完成

`Gradle` 是一个 运行在 `JVM` 的通用构建工具，其核心模型是一个由多个 `Task` 组成的有向无环图

<img src="https://raw.githubusercontent.com/WeYan1223/Pic/master/Gradle/Gradle_有向无环图.webp" alt="Gradle_有向无环图.webp (737×439) (raw.githubusercontent.com)"  /> 

***

#### 开发语言

##### 1. Groovy

`Groovy` 基于 `JVM` 的动态语言，其语法与 `Kotlin` 非常相似

##### 2. Kotlin

从 `Android Studio Giraffe` 版本开始，`Gradle` 脚本默认使用 `Kotlin` 语言，[Kotlin DSL is Now the Default for New Gradle Builds](https://android-developers.googleblog.com/2023/04/kotlin-dsl-is-now-default-for-new-gradle-builds.html)

***

#### 生命周期 

> `Gradle` 将构建划分为三个阶段： 初始化 -- 配置 -- 执行

![Gradle_生命周期.webp (1920×1080) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/Gradle/Gradle_生命周期.webp) 

##### 1. 初始化

> `Initialization`

初始化又分为两个子过程

* 执行 `Init Script`，读取全局脚本, 主要作用是初始化一些全局通用的属性
* 执行 `Setting Script`，就是前面提到的 `settings.gradle`

##### 2. 配置

> `Configuration`

配置阶段开始加载项目中所有参与构建的模块的 `Build Script`，即执行 `build.gradle` 中的语句，主要有

* 下载插件和依赖
* 执行脚本代码
* 根据脚本代码创建对应的 `task`
* 最终根据所有 `task` 生成对应的有向无环图

##### 3. 执行

> `Execution`

真正开始执行 `task`

***

#### 参考文章

* [关于 Gradle 你应该知道的知识点](https://juejin.cn/post/7064350945756332040)