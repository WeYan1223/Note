#  1. Distributions

`Gradle` 是由纯 `Java` 编写的程序，在 [Gradle Distributions](https://services.gradle.org/distributions/) 中可以下载所有版本的 `Gradle`，每个版本都有三种格式的产物：

* `src`: `Gradle` 的源代码，不可运行
* `bin`: `Gradle` 源代码编译后的二进制发布版，可运行（最常用的格式）
* `all`: 比 `bin` 多了用户文档与使用样例

以 `gradle-8.6-bin` 为例，下载的产物结构如下：

````sh
.
├─bin
│    gradle     // Unix系统下的脚本文件
│    gradle.bat // Windows系统下的脚本文件
├─init.d
└─lib           // Gradle运行所需要的各种jar包
    ├─agents
    └─plugins
````

启动 `Gradle` 的本质是执行脚本（`Window`下为 `gradle.bat`），脚本的内容为：启动 `JVM` 并加载 `lib` 目录下的 `jar` 包



# 2. GradleUserHome

`Gradle User Home` 是一个目录，默认是 `~/.gradle` 可以通过设置环境变量 `GRADLE_USER_HOME` 更改，用于存储 `Gradle` 运行所需要的文件：

````sh
.
├─.tmp
├─caches        // 存放各种依赖包
├─daemon
├─jdks
├─native
├─notifications
└─wrapper       // 存放Wrapper下载的特定版本的Gradle
````

`Gradle` 运行时只跟两个目录打交道：

* 项目所在目录
* `Gradle User Home`



# 3. Wrapper

`Wrapper` 是对 `Gradle` 的封装，目的在于保证 `Gradle` 的版本跟着项目走，方便项目在不同的设备上运行，其中涉及到的文件如下：

````sh
.
│  gradlew     // Unix系统下的脚本文件，执行Wrapper
│  gradlew.bat // Windows系统下的脚本文件，执行Wrapper
├─gradle
│  └─wrapper
│          gradle-wrapper.jar        // Wrapper运行所需的jar包，其中包含了下载Gradle的代码
│          gradle-wrapper.properties // Wrapper运行所需的配置文件，指定Gradle的下载地址和存放目录
````

关于 `Wrapper` 的运行机制，引用[官网](https://docs.gradle.org/current/userguide/gradle_wrapper.html)的图：

![wrapper workflow](https://docs.gradle.org/current/userguide/img/wrapper-workflow.png)`Windows` 系统下，在命令行中使用 `Wrapper` 的方式：

````sh
./gradlew --version
````



# 4. Daemon

 先提一提 `Maven`：`Maven` 每次构建都会启动 `JVM` 进程，构建结束后再关闭此进程，而每次启动 `JVM` 加载 `jar` 文件是一个相当耗时的操作。

`Gradle 3.0` 之后，默认使用 `Deamon` 模式：每次打开项目都会启动一个 `Daemon` 的 `JVM` 进程（默认保留三个小时）。而真正开始构建时，则会启动一个非常轻量的 `Client` `JVM` 进程，只用于和 `Daemon` 进程通信，`Client` 进程将当前构建的一些上下文信息发送给 `Daemon` 进程，`Daemon` 进程负责真正的构建，构建完成再关闭掉 `Client` 进程。

可以通过 `./gradlew --status` 查看项目对应 `Gradle` 版本的 `Daemon` 进程：

````sh
D:\Develop\IntellJIDEA\Projects\Tmp> ./gradlew --status    
   PID STATUS   INFO                                        
 15096 IDLE     7.4
Only Daemons for the current Gradle version are displayed. See https://docs.gradle.org/7.4/userguide/gradle_daemon.html#sec:status
````

构建时可以添加参数 `--no-daemon` 来指定本次构建不使用 `Daemon` 进程



# 5. buildSrc

运行 `Gradle` 时会检查项目中是否存在一个名为 `buildSrc` 的目录，然后 `Gradle` 会自动编译并测试这段代码，并将其放入<u>构建脚本的类路径</u>中, 对于多项目构建，只能有一个 `buildSrc` 目录，该目录必须位于根项目目录中, `buildSrc` 是 `Gradle` 项目根目录下的一个目录，它可以包含我们的构建逻辑，与脚本插件相比，`buildSrc` 应该是首选，因为它更易于维护、重构和测试代码。



# 6. Plugins

`Gradle` 构建的绝大多数功能是通过应用 `Plugin` 来实现的。在 `build.gradle` 文件中创建 `Task` 和编写配置代码会使得文件非常混乱且难以复用，`Plugin` 的作用主要是将这些可复用的 `Task` 和 配置逻辑打包在一起。

`Plugin` 的来源主要有三种：

* [Gradle 内置插件](https://docs.gradle.org/current/userguide/plugin_reference.html#plugin_reference)：不需要下载，直接可以应用
* [社区插件](https://plugins.gradle.org/?_gl=1*1m7kl0e*_gcl_au*MTY5MjgyMjUzNS4xNzE4NjEwMDM5*_ga*MTIwMDMyMjY5NS4xNzE4NjEwMDM4*_ga_7W7NC6YNPT*MTcxODczMjc0Ny42LjEuMTcxODczNjIzNi42MC4wLjA.)
* 本地插件



# 7. Build Lifecycle

> [Build Lifecycle](https://docs.gradle.org/current/userguide/build_lifecycle.html)

![gradle build lifecycle](https://docs.gradle.org/current/userguide/img/gradle-build-lifecycle.png) 

**1. Initialization**

初始化阶段，`Gradle` 通过 `settings.gradle` 文件确认有哪些项目参与本次构建，并为每个项目创建相应的 `Project` 实例

**2. Configuration**

配置阶段，执行每个项目的 `build.gradle` 文件来配置相应的 `Project` 实例，创建 `Task` 的有向无环图

**3. Execution**

执行阶段，`Gradle` 执行指定的 `Task`（注意 `Task` 之间存在依赖关系，以下图为例，执行 `jar` 时，必须先执行 `compile`、`processResources`、`classes`）

![task dag examples](https://docs.gradle.org/current/userguide/img/task-dag-examples.png)



# 8. Configuring Build Environment

> [Configuring the Build Environment](https://docs.gradle.org/current/userguide/build_environment.html)

通过配置各种属性来自定义 `Gradle` 的构建过程，属性配置的优先级如下（从高到低）：

**1. 命令行参数**

通过命令行启动 `Gradle` 时指定参数，通常使用这种方式来指定 `Gradle` 的运行行为而不是配置属性，这里不分析采用命令行的方式配置属性，细节参考[Command-Line Interface Reference](https://docs.gradle.org/current/userguide/command_line_interface.html#command_line_interface)

**2. 系统属性**

存放项目根目录下的 `gradle.properties` 中前缀为 `systemProp.` 的属性，例如

````groovy
// gradle.properties

systemProp.gradle.wrapperUser=myuser
systemProp.gradle.wrapperPassword=mypassword
````

系统属性属于 `JVM` 层面，同时 `Gradle` 构建过程也可以访问这些属性，例如

````groovy
// build.gradle

println('my name is ' + System.getProperty('gradle.wrapperUser'))
````

具体有哪些系统属性参考[System properties](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_system_properties)

**3. 项目属性**

存放在 `GRADLE_USER_HOME` 或项目目录下的 `gradle.properties` 中的属性，通常采用这种方式来集中管理依赖的版本号

**4. Gradle 属性**

`Gradle` 属性用于控制 `Gradle` 运行时的行为，有两种方式进行配置：命令行、`gradle.properties`

具体有哪些 `Gradle` 属性参考[Gradle properties](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_configuration_properties)

**5. 环境变量**

通常只需设置 `GRADLE_USER_HOME` 环境变量即可，其他环境变量及其作用参考[Environment variables](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_environment_variables)



# 9. Dependency Management

> [Dependency Management](https://docs.gradle.org/current/userguide/core_dependency_management.html)

关于添加依赖，最简单的例子就是将一个 `jar` 包的路径添加到项目的 `classpath` 中

`Gradle` 具有一套灵活的依赖管理功能，主要关注一下几点：

* 通过声明的方式为项目添加依赖，不同的声明方式可以满足不同场景的需求（如`compile`、`testCompile`、`runtime`等）
* 处理依赖传递
* 解决版本冲突

详细参考文章[Gradle 爬坑指南 -- 依赖管理](https://juejin.cn/post/6895299152226615309)



# 参考资料

* [官网](https://docs.gradle.org/current/userguide/quick_start.html)
* [来自Gradle开发团队的Gradle入门教程](https://www.bilibili.com/video/BV1DE411Z7nt/?spm_id_from=333.337.search-card.all.click)
* [Gradle 爬坑指南 -- 依赖管理](https://juejin.cn/post/6895299152226615309)