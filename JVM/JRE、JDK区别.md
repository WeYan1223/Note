**JRE（Java Runtime Environment）**，是运行编译后的 Java 程序所需的环境，包括 Java 虚拟机、Java 类库、Java 命令和其他基础设施。

**JDK（Java Development Kit）**，Java 开发工具包，除了包含 JRE 的一切，还为开发者提供各种工具如 javac、javap、javadoc 等。



通常下载完 JDK 后，还需要添加环境变量 JAVA_HOME 和 PATH

JAVA_HOME 作用：

* **开发工具配置**：许多开发工具和构建工具如 Gradle、Tomcat 在运行时需要知道 JDK 的安装路径。它们会查找 JAVA_HOME 变量来确定 JDK 的位置，从而正确地编译和运行 Java 程序。
* **PATH配置**：有时候可能需要安装多个不同版本的 JDK，配置 PATH 时使用 JAVA_HOME 作为占位符可以很方便地切换 JDK 为不同的版本，参考[Windows同时安装两个版本JDK，并实现动态切换JAVA8或者JAVA11](https://juejin.cn/post/7167276296152547335?searchId=2024061616543268FC009DC87534B1763C)。

PATH 作用：

* PATH 环境变量决定了操作系统在执行命令时搜索可执行文件的目录。具体来说，将 JDK 的 bin 目录添加到 PATH 中，可以使得在命令行中直接使用诸如 java、javac 等 Java 命令，而无需输入它们的完整路径。



