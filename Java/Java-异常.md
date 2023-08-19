#### 简述

异常：指程序在执行过程中出现的非正常情况，最终可能会导致程序非正常停止（注意，异常不是语法错误，语法错误不会编译通过）

<img src="https://raw.githubusercontent.com/WeYan1223/Pic/master/Java/Java_异常体系结构.webp" alt="Java_异常体系结构.webp (1101×431) (raw.githubusercontent.com)" style="zoom:80%;" /> 

`Throwable` 是所有 `Error `和 `Exception` 的父类，只有 `Throwable` 的子类才能够被抛出（另外，带有 `@throw` 注解的类也可以被抛出）

##### 1. Error

`Error` 是程序无法处理的错误，表示程序运行中出现了严重的错误，它们在应用程序的处理能力之外，对于设计合理的应用程序来说，即使确实发生了错误，本质上也不应该试图去处理它

常见的 `Error` 有 `OutOfMemoryError` 与 `StackOverflowError`

##### 2. Exception

通常说的异常就是指 `Exception`，主要分为两部分：非检查异常、检查异常

异常中包含的信息可以有效回答以下问题：

* 出了什么错（异常类型）
* 在哪出的错（异常堆栈）
* 为什么出错（异常信息）

***

#### 异常类型

##### 1. 非检查异常

> 又称运行时异常，`RuntimeException` 及其子类都是运行时异常，通常由程序的逻辑错误引起
>
> 同时，编译器不会对此类异常进行检查，常见的有 `NullPointerException`

````java
public static void main(String[] args) {
    f(); // 程序编译通过
}

private static void f() {
    throw new NullPointerException();
}
````

运行时异常通常由程序逻辑错误引起，应该从逻辑角度尽可能避免这类异常的发生

编码习惯：方法体中如果会抛出异常，最好使用 `throws` 关键字声明可能会抛出的异常并添加相应的注释

##### 2. 检查异常

> `Exception` 的子类中除了非检查异常，其余都是检查异常
>
> 同时，编译器强制要求捕获或抛出的检查异常，常见的有 `IOException`

````java
public void main() {
    f(); // 程序编译不通过，编译器提示：f()有可能会抛出IOException，需要抛出或捕获
}

private void f() throws IOException {
    //...
    throw new IOException();
}
````

检查异常是程序中很容易出现的、情理可容的状况，在一定程度上它的发生是可以预计的，而且一旦发生这种异常状况，就必须采取某种方式进行处理

***

#### 异常处理

在程序中最大限度地使用好异常，需要遵守以下三个规则

* 具体明确

* 提早抛出

* 延迟捕获

##### 1. 具体明确

> 捕获的异常应该是具体的异常

`FileNotFoundException`、`EOFException`、`ObjectStreamException` 都是 `IOException` 的子类，每一种都描述了一类特定的 `IO` 异常：分别是文件丢失、异常文件结尾、错误的序列化对象流

异常越具体，越能回答出了什么错

````java
try{
    readFile(file);
} catch (FileNotFoundException e){
    //Todo
} catch (EOFException e){
    //Todo
} catch (ObjectStreamException e){
    //Todo
} catch (IOException e){
    //Todo
}
````

如上代码，如果捕获了 `FileNotFoundException`，可以提示用户指定另一个文件或者读取默认文件

除前三个 `catch` 块，最后一个 `catch` 块捕获 `IOException` 给用户提供了更泛化的错误信息

这样一来，程序就可以尽可能提供具体的信息，但也有能力处理未预料到的其他异常

##### 2. 提早抛出

异常堆栈信息提供了导致异常出现的方法调用链的精确顺序，包括每个方法调用的类名、方法名、代码文件名甚至行数，以此来精确定位异常出现的位置

````java
public static void main(String[] args) {
    readFile(null);
}

private static void readFile(String filePath) {
    InputStream inputStream = new FileInputStream(filePath);
    //...
}
````

运行以上代码，会打印如下堆栈信息

````sh
Exception in thread "main" java.lang.NullPointerException
	at java.base/java.io.FileInputStream.<init>(FileInputStream.java:149)
	at java.base/java.io.FileInputStream.<init>(FileInputStream.java:111)
	at Main.readFile(Main.java:23)
	at Main.main(Main.java:18)
````

由于 `FileInputStream` 是 `Java API` 的一部分，所以导致该异常的原因很可能是自己编写的代码，而不是 `Java API`

但是，`NullPointerException` 是 `Java` 中信息量最少的异常，根本不知道**为什么出错**

逐步检查代码，会发现原因在于 `filePath` 为 `null`

既然 `readFile()` 不能处理 `filePath` 为 `null` 的情况，此时应该马上检查该条件

````java
private static void readFile(String filePath) throws IllegalArgumentException {
    if (filePath == null) {
        throw new IllegalArgumentException("filePath is null");
    }
    InputStream inputStream = new FileInputStream(filePath);
    //...
}
````

通过提早抛出异常，异常显得清晰又准确

堆栈信息立即反映出出了什么错（非法参数）、在哪出的错（第 23 行）以及为什么出错（`filePath is null`）

````sh
Exception in thread "main" java.lang.IllegalArgumentException: filePath is null
	at Main.readFile(Main.java:23)
	at Main.main(Main.java:18)
````

在检测到异常且无法处理该异常时，立刻抛出异常来实现迅速失败，可以有效避免不必要的对象构造或资源占用，比如文件或网络连接

***

##### 3. 延迟捕获

> 不要在程序没有处理异常能力的时候进行捕获异常

这是最常见的犯错：程序没有处理异常能力的时候进行捕获异常

平时编写代码时（特别是访问文件、网络），一旦看到自己写的代码下面冒红线，很顺手地 `Alt + Enter` 选择 `Surround with try/catch`

问题在于，捕获到异常之后该怎么办：

* 什么都不做，令 `catch` 块为空，这是最不应该做的，等于把整个异常丢进黑洞，能够说明何时何处为何出错的所有信息都会永远丢失
* 打印堆栈信息，对开发人员是调试有用的，但是用户使用过程中产生异常，这样做并未能解决实际问题
* 将异常信息写到日志中还稍微好点，至少有记录可查，但是这些日志是给开发人员看的
* 弹出显示错误信息的对话框也不合适，例如在 `readFile()` 中弹出一个对话框未免违背了单一职责原则，使得 `readFile()` 方法过于混乱臃肿

```java
private static void readFile(String filePath) throws IllegalArgumentException {
    if (filePath == null) {
        throw new IllegalArgumentException("filePath is null");
    }
    try {
        InputStream inputStream = new FileInputStream(filePath);
        //...
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    }
}
```

如上代码就是在 `readFile()` 不具能力处理 `FileNotFoundException` 时就将异常捕获

最好的方法就是将异常抛给调用者进行处理

````java
private static void readFile(String filePath) throws IllegalArgumentException, FileNotFoundException, IOException {
    if (filePath == null) {
        throw new IllegalArgumentException("filePath is null");
    }
    InputStream inputStream = new FileInputStream(filePath);
}
````

需要注意，这里的 `IllegalArgumentException` 不是必须声明的，因为它是非检查异常，这样做是为了文档化代码（在 `JavaDocs` 中标注出来）

***

#### 异常表

> 从字节码层面理解异常

每个方法都会包含一个异常表（前提：方法内有 `try/catch/finally`），异常可以简单理解为一个 `List`，其中的每个元素包含以下字段：

* `start_pc`
* `end_pc`
* `target_pc`
* `catch_type`

当在字节码在 `[start_pc, end_pc)` 中出现了 `catch_type` 类型的异常，那么马上执行 `target_pc` 字节码

`Java` 中，结束方法有两种方式：

* 方法正常执行结束，`return`

  ```java
  public static int method() {
      int i = 10;
      return i;
  }
  ```

  对应的字节码如下

  ````sh
  0 bipush 10 # 把数字 10 压入操作数栈
  2 istore_0  # 栈顶元素出栈，放入局部变量表 0 号槽位
  3 iload_0   # 局部变量表 0 号槽位的元素压入操作数栈
  4 ireturn   # 返回栈顶元素
  ````

* 方法出现异常并抛出，`throw`

  ```java
  public static int method() {
      int i = 10;
      throw new NullPointerException();
  }
  ```

  对应的字节码如下

  ````sh
   0 bipush 10     # 把数字 10 压入操作数栈
   2 istore_0      # 栈顶元素出栈，放入局部变量表 0 号槽位
   3 new           # 创建一个 NullPointerException 对象, 并将其引用引用值压入栈顶
   6 dup           # 复制栈顶元素并将复制值压入栈顶
   7 invokespecial # 栈顶元素出栈，作为参数调用 NullPointerException 的构造方法
  10 athrow        # 抛出栈顶元素
  ````

> 分析下面的代码输出什么

````java
private static int f() {
    try {
        return 10;
    } finally {
        return 20;
    }
}
````

如上代码的输出结果为 20，现在分析其字节码

````sh
0: bipush        10 # 把数字 10 压入操作数栈
2: istore_0         # 栈顶元素出栈，放入局部变量表 0 号槽位
3: bipush        20 # 把数字 20 压入操作数栈
5: ireturn          # 返回栈顶元素
6: astore_1         # 栈顶的异常对象出栈，放入局部变量表 1 号槽位
7: bipush        20 # 把数字 20 压入操作数栈
9: ireturn          # 返回栈顶元素
Exception table:
from    to  target type
0       3   6      any
````

可以发现，不论 `try` 中是否发生异常，在 `finally` 有 `return` 语句都不会抛出异常，这是一个非常危险的操作

> 分析下面代码输出什么

````java
public static int f() {
    int i = 10;
    try {
        return i;
    } finally {
        i = 20;
    }
}
````

如上代码的输出结果为 10，现在分析其字节码

````sh
0: bipush        10 # 把数字 10 压入操作数栈
2: istore_0         # 栈顶元素出栈，放入局部变量表 0 号槽位
3: iload_0          # 局部变量表 0 号槽位的元素压入操作数栈
4: istore_1         # 栈顶元素出栈，放入局部变量表 1 号槽位，用来之后返回
5: bipush        20 # 把数字 20 压入操作数栈
7: istore_0         # 栈顶元素出栈，放入局部变量表 0 号槽位
8: iload_1          # 局部变量表 1 号槽位的元素压入操作数栈
9: ireturn          # 返回栈顶元素
10: astore_2        # 栈顶的异常对象出栈，放入局部变量表 2 号槽位，用来之后抛出
11: bipush       20 # 把数字 20 压入操作数栈
13: istore_0        # 栈顶元素出栈，放入局部变量表 0 号槽位
14: aload_2         # 局部变量表 2 号槽位的异常对象压入操作数栈
15: athrow          # 抛出栈顶元素
Exception table:
from    to  target type
3       5   10     any   
````

