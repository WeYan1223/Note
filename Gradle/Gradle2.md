#### 核心概念

##### 1. Gradle

从<u>软件</u>的角度看，`Gradle` 是 `Java` 语言编写的软件，是一个通用的构建工具

从<u>构建</u>的角度看，`Gradle` 为整个构建流程提供基础架构，例如严格的生命周期（初始化-配置-执行）、 `Task` 有向无环图的构建

##### 2. Project

`Gradle` 可以

##### 3. Plugin

`Gradle` 为构建任何应用提供了基本框架，具体怎么构建交由上层的 `Plugin` 决定，例如 `AGP` 为打包 `APK` 定义了许多 `Task`

##### 4. Task

`Task` 是 `Gradle` 中的原子性操作，在 `Gradle` 的配置阶段会将所有 `Task` 组成有向无环图（`Task` 内部由多个 `Action` 组成）



***

#### 项目结构

以 `Android Studio` 为例，新建项目后的项目结构如下

````sh
.
├── README.md
├── app
│   └── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
└── gradlew.bat
├── build.gradle
├── local.properties
├── gradle.properties
└── settings.gradle
````

##### 1. wrapper

`Gradle Wrapper` 是对 `Gradle` 的封装，意义在于保证 `Gradle` 的版本跟着项目走，方便项目在不同的设备上运行，主要文件：

* `gradle-wrapper.jar`：`Gradle` 的运行逻辑，包括下载特定版本的 `Gradle`
* `gradle-wrapper.properties`：配置文件，主要指定 `Gradle` 版本以及缓存目录
* `gradlew`：`Linux/Mac` 系统的可执行文件，用于执行 `Gradle Wrapper`
* `gradlew.bat`：`Windows` 系统的可执行文件，用于执行 `Gradle Wrapper`

在命令行中使用 `Gradle Wrapper`（`Windows` 系统）：

````sh
./gradlew --version
````



##### 2. properties

* `local.properties`：定义私有的本地属性，不会被 `Git` 提交到远程

  ````properties
  ## Android Studio 自动生成的 local.properties 文件
  # SDK 路径
  sdk.dir=D\:\\SDK
  ````

  该文件主要保存个人私有的属性，例如用户名、密码、密钥等私密信息，具体使用方式参考[Android 如何從 local.properties, build.gradle 讀取定義的屬性 | by Wayne Chen | Medium](https://medium.com/@waynechen323/如何從-local-properties-build-gradle-讀取定義的屬性-e19aa8a8c12b)

  注意：`local.properties` 文件并不在 `Gradle` 的框架内，但是是处于 `Android Studio` 定义的规范中

* `gradle.properties`：该文件由 `Gradle` 自动生成，定义两种属性——<u>1. `Gradle` 运行所需属性</u>、<u>2. 项目构建所需属性</u>

  ````properties
  ## gradle.properties
  # Gradle 运行时的 JVM 参数
  org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
  
  # 构建 Android 项目相关属性，应该是 AGP 默认提供的
  android.useAndroidX=true
  kotlin.code.style=official
  android.nonTransitiveRClass=true
  ````

  由于该文件由 `Gradle` 自动生成，其内部对属性的读取做了处理，所以在 `build.gradle` 中可以直接读取属性值

* `setting.gradle`：定义项目级代码库设置，`Gradle 7.0` 之后，其内容比较固定，通常不需要修改

  ````groovy
  // Gradle插件管理，指定插件的下载仓库
  pluginManagement {
      repositories {
          google()
          mavenCentral()
          gradlePluginPortal()
      }
  }
  
  // 依赖管理，指定依赖的下载仓库
  dependencyResolutionManagement {
      repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
      repositories {
          google()
          mavenCentral()
      }
  }
  
  // 项目名称
  rootProject.name = "Demo-Gradle"
  
  // 指定哪些模块参与构建
  include ':app'
  include ':baselib'
  ````

  

##### 3. build.gradle

> `build.gradle` 文件是构建的核心

* 项目级 `build.gradle`：对子模块进行统一配置，一般不会有太多内容

  ````groovy
  // 根目录下的 build.gradle
  plugins {
      // 默认只有 plugin 的引用，apply false 表示不将该 plugin 应用于当前项目
      id 'com.android.application' version '8.0.1' apply false
      id 'com.android.library' version '8.0.1' apply false
      id 'org.jetbrains.kotlin.android' version '1.8.20' apply false
  }
  ````

  除了应用插件外，还可以应用自定义 `.gradle` 文件

  ````groovy
  // 根目录下的 build.gradle
  // 将 version.gradle 应用于当前项目以及子项目
  apply from: 'version.gradle'
  ````

* 模块级 `build.gradle`：主要做三件事，应用插件、配置属性、添加依赖

  ````groovy
  // Android Studio 中 app 模块的 build.gradle
  
  // 1.应用插件
  plugins {
      id 'com.android.application'
      id 'org.jetbrains.kotlin.android'
  }
  
  // 2.配置属性
  android {
      namespace 'com.yws.demo_gradle'
      compileSdk 33
  
      defaultConfig {
          applicationId "com.yws.demo_gradle"
          minSdk 26
          //...
      }
      
      //...
  }
  
  // 3.添加依赖
  dependencies {
      implementation 'androidx.core:core-ktx:1.8.0'
      implementation 'androidx.appcompat:appcompat:1.4.1'
      //...
  }
  ````

  在配置属性这一块，具体配置什么属性、怎么配置属性，这与插件密切相关，因此需查看相关插件的使用说明

  对于模块级 `build.gradle` 的编写，最好遵循该文章：[Gradle模块化配置：让你的gradle代码控制在100行以内 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903518411685895)



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

#### 依赖管理





***

#### 参考文章

* [【Gradle-1】入门Gradle，前置必读 - 掘金 (juejin.cn)](https://juejin.cn/post/7155109977579847710)
* [Android 如何從 local.properties, build.gradle 讀取定義的屬性 | by Wayne Chen | Medium](https://medium.com/@waynechen323/如何從-local-properties-build-gradle-讀取定義的屬性-e19aa8a8c12b)
* [Gradle模块化配置：让你的gradle代码控制在100行以内 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903518411685895)
* [关于 Gradle 你应该知道的知识点 - 掘金 (juejin.cn)](https://juejin.cn/post/7064350945756332040)

