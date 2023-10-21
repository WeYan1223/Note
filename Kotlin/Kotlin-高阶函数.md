#### 高阶函数

> 参数类型或返回类型为函数类型的函数，称为高阶函数

````kotlin
// 参数存在函数类型
fun highOrderFunc1(block: (Int) -> String) {
    block(0)
}

// 返回类型为函数类型
fun highOrderFunc2(): (Int) -> String {
    ...
}
````

调用高阶函数（`lambda` 表达式简化写法）

````kotlin
fun main() {
    highOrderFunc1 { num ->
        num.toString()
    }
}
````

由于 `Kotlin` 与 `Java` 是通用的，`Java` 中并没有函数类型，所以 `Kotlin` 需要创建一个对象来作为函数类型变量的载体，让该对象去执行实际的代码

编译后的伪代码大致如下：

````java
//高阶函数
fun highOrderFunc1(block: Function1<Int, String> {
    block.invoke(1)
}

//调用处
fun main() {
    highOrderFunc1(object: Function1<Int, String> {
        override fun invoke(num: Int) {
            return num.toString()
        }
    })
}
````

这与 `Java` 的接口回调的写法基本一致：

````java
public static void post(Callback callback) {
    //post something
    callback.onSuccess()
}

//调用处
public static void main(args: String[]) {
    post(new Callback() {
        @Override
        public void onSuccess() {
            //do something on success
        }
    })
}
````

`Kotlin` 内置了许多 `Function` 接口，结构如下：

![Kotlin_Function继承体系.webp (752×251) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/Kotlin/Kotlin_Function继承体系.webp) 

显然，每一次调用高阶函数，都会产生相应数量的 `Function` 对象，会导致运行时的效率损失

如果在 `N` 次循环里调用高阶函数，整个循环下来就产生了 `N` 个 `Function` 对象

***

#### 函数内联

##### 1. inline

被 `inline` 修饰的函数称为内联函数，内联函数在编译时会进行内联优化：

> 调用处，内联函数在编译的时候会变成代码内嵌的形式

对普通函数的调用

````kotlin
fun hello() {
    println("hello")
}
````

```kotlin
// 源码
fun main() {
    hello()
}

// 编译后的伪代码
fun main() {
    hello()
}
```

对普通内联函数的调用（这里说普通内联函数只是为了跟后面提到的高阶内联函数做一下区别）

````kotlin
inline fun hello() {
    println("hello")
}
````

````kotlin
// 源码
fun main() {
    hello()
}

// 编译后的伪代码
fun main() {
    println("hello")
}
````

> 如果这是高阶函数，那么其函数类型的参数同样会被展开，而不会生成 `Function` 对象

对于高阶函数，`inline` 还可内联其函数类型的参数，也就是说，这个高阶内联函数在编译后是完全铺平展开的，例如

```kotlin
// 高阶内联函数
inline fun hello(postAction:() -> Unit) {
    println("hello")
    postAction()
}
```

```kotlin
// 源码
fun main() {
    hello {
        println("postAction")
    }
}

// 编译后的伪代码
fun main() {
    println("hello")
    println("postAction")
}
```

对于高阶函数，通过 `inline` 可减少 `Function ` 对象的创建，这里对比一下内联与非内联的效率

````kotlin
//内联函数
inline fun inlineFunc(action: () -> Unit) {
    action()
}

//非内联函数
fun noinlineFunc(action: () -> Unit) {
    action()
}

fun main() {
    runBlocking {
        val inlineNanoTime = measureNanoTime {// 计算内联函数运行时间，时间单位纳秒
            repeat(100) {
                inlineFunc { 1 + 1 }
            }
        }
        val noinlineNanoTime = measureNanoTime {// 计算非内联函数运行时间，时间单位纳秒
            repeat(100) {
                noinlineFunc { 1 + 1 }
            }
        }
        println("inlineNanoTime = $inlineNanoTime, noinlineNanoTime = $noinlineNanoTime")
        //打印结果：inlineNanoTime = 1300, noinlineNanoTime = 283400
    }
}
````

一般来说，如果写的是高阶函数，存在函数类型的参数，加上 `inline` 就对了

但是，如果对饱体积大小有严格要求，就得酌情使用了

同时，内联函数有以下限制：公有的内联函数不能访问私有的成员变量

````kotlin
class Student {
    private val name = "Bob"
    
    inline fun hello(action: (String) -> Unit) {
        action(name)//编译器报错：Public-API inline function cannot access non-public-API 'private final val name: String defined
    }
}
````

***

##### 2. noinline

> 局部关闭内联优化：`noinline` 修饰的函数类型的参数，可以使其不参与内联

````kotlin
//高阶内联函数
inline fun hello(preAction: () -> Unit, noinline postAction: () -> Unit) {
    preAction()// 做点前置工作
    println("hello")
    postAction()// 做点后续工作
}
````

````kotlin
// 调用处
fun main() {
    hello(
        { println("pre action") }, 
        { println("post action") }
    )
}

// 编译后的伪代码
fun main() {
    println("pre action")
    println("hello")
    val func = object: Function0<Unit> {
        override fun invoke() {
            println("post action")
        }
    }
    func.invoke()
}
````

> 什么时候需要使用 `noinline` ？

函数类型参数，本质上是一个 `Function` 对象，如果在内联函数中，需要把它当作对象来使用时，需要使用 `noinline`：

* 调用对象的方法
* 作为另一个函数的参数
* 作为返回值

原因也很简单，内联函数把函数类型的参数也展开了，编译后这个参数是不存在了

***

##### 3. crossinline

> 用于解决高阶内联函数的 `lambda` 表达式的非局部返回带来的问题

首先，调用高阶函数可以有以下三种写法

```kotlin
//高阶函数
fun highOrderFunc(action: () -> Unit) {
    action()
}

fun main() {
    // lambda 表达式
    highOrderFunc {
        println("lambda")
    }
    // 匿名函数
    highOrderFunc(fun() {
        println("anonymous function")
    })
    // 具名函数
    highOrderFunc(::namedFunction)
}

fun namedFunction() {
    println("named function")
}
```

对于这三种方式，加上 `return` 语句

```kotlin
fun main() {
    // lambda 表达式
    highOrderFunc {
        println("lambda")
        return // 编译器报错：'return' is not allowed here
    }
    // 匿名函数
    highOrderFunc(fun() {
        println("anonymous function")
        return // 退出当前匿名函数
    })
    // 具名函数
    highOrderFunc(::namedFunction)
}

fun namedFunction() {
    println("named function")
    return // 退出当前函数
}
```

`Kotlin` 中，对于 `lambda` 表达式的 `return`，有如下规定

* 默认情况下，`lambda` 表达式不能使用不带标签的 `return` 来返回，但是可以使用 `return@label` 来显式指定返回的位置

* 如果是内联函数，那么 `lambda` 表达式可以使用 `return` 退出包含它的函数，即<u>非局部返回</u>

  ```kotlin
  //高阶内联函数
  inline fun hello(preAction: () -> Unit) {
      preAction()
      println("hello")
  }
  
  //调用处
  fun main() {
      hello {
          println("pre action")
          return// 实际上 return 的是 main()
      }
  }
  
  // 编译后的伪代码
  fun main() {
      println("pre action")
      return
      println("hello")
  }
  ```

如果内联函数中，函数类型参数是被间接调用的

````kotlin
//高阶内联函数
inline fun hello(afterHello: () -> Unit) {
    println("hello")
    Handler(Looper.getMainLooper()).post {
        afterHello()// 编译器报错：afterHello() 内部可能存在非局部返回，不允许间接调用
    }
}
````

此时，使用 `crossinline` 修饰参数，来向编译器保证其内部不存在非局部返回

````kotlin
//调用处
fun main() {
    hello {
        println("after hello")
        return// 编译器报错: 'return' is not allowed here
    }
}

inline fun hello(crossinline afterHello: () -> Unit) {
    println("hello")
    Handler(Looper.getMainLooper()).post {
        afterHello()// 不报错
    }
}
````

***

#### 参考文章

* [Inline functions | Kotlin Documentation (kotlinlang.org)](https://kotlinlang.org/docs/inline-functions.html)

* [inline, noinline, crossinline — What do they mean? | by Ben Daniel A. | AndroidPub | Medium](https://medium.com/android-news/inline-noinline-crossinline-what-do-they-mean-b13f48e113c2)

* [Kotlin 源码里成吨的 noinline 和 crossinline 是干嘛的？看完这个视频你转头也写了一吨 (rengwuxian.com)](https://rengwuxian.com/kotlin-source-noinline-crossinline/)