#### 修饰符

`Kotlin` 提供了四个可见性修饰符：`private`、`protected`、`internal` 和 `public`，不写则默认是 `public`

> 顶层声明（函数、属性、类、接口可直接在文件内声明）

- 不使用修饰符，默认为 `public`：随处可见
- `private`：文件内可见
- `internal`：模块内可见
- `protected` 不适用于顶层声明

> 类内部声明

- `private`：类内部可见
- `protected`：类内部可见、子类中可见
- `internal`：模块内可见
- `public`：随处可见

***

#### 数据类型

> 在 `Kotlin` 中，所有东西都是对象

##### 1. 数字

> `Kotlin` 不支持八进制

**整数型**

* `Byte`：1 字节
* `Short`：2 字节
* `Int`：4 字节
* `Long`：8 字节

在没有显式指定变量的类型时，如果没有超过 `Int` 的范围，那么类型就是 `Int`，否则就是 `Long`

对于 `Long`，除了显式指定类型，还可以在值得后面追加 `L` 表示 `Long` 类型

**浮点型**

* `Float`：4 字节
* `Double`：8 字节

````kotlin
val a = 3.14 // Double
val b: Double = 1 // 错误：类型不匹配
val c = 1.0 // Double
val d = 2.7182818284 // Double
val e = 2.7182818284F // Float，实际值为 2.7182817
````

**语法糖**

可以使用下划线提高可读性

````kotlin
val oneMillion = 1_000_000
val creditCardNumber = 1234_5678_9012_3456L
val bytes = 0b11010010_01101001_10010100_10010010
````

**位运算**

> 位运算只能用于 `Int` 与 `Long`

* `shl(bits)` – 有符号左移
* `shr(bits)` – 有符号右移
* `ushr(bits)` – 无符号右移
* `and(bits)` –按位与
* `or(bits)` – 按位或
* `xor(bits)` – 按位异或
* `inv()` – 非

##### 2. 字符

`Char`

##### 3. 布尔

`Boolean`

##### 4. 数组

> 在 `Kotlin` 中，数组使用 `Array` 表示

````kotlin
public class Array<T> {
    public inline constructor(size: Int, init: (Int) -> T)

    public operator fun get(index: Int): T

    public operator fun set(index: Int, value: T): Unit

    public val size: Int

    public operator fun iterator(): Iterator<T>
}
````

使用数组：

````kotlin
val ints: Array<Int> = arrayOf(1, 2, 3, 4)//创建一个大小为4，初始化值为1，2，3，4的数组
val ints2: Array<Int> = Array(12) { 0 }//创建一个大小为12，初始化值为0的数组
````

##### 5. 字符串

`String`，可以直接使用索引访问字符

````kotlin
val str = "abcd"
for (c in str) {
    println(c)
}
````

`Kotlin` 中有两种字符串字面量：

**转义字符串**

转义字符串可以包含转义字符，跟 `Java` 的字符串字面量一样 

**原始字符串**

原始字符串包含换行以及任意文本，使用三个引号`"""`分界符括起来，内部没有转义并且可以包含换行以及任何其他字符

````kotlin
val text = """
    for (c in "foo")
        print(c)
"""
````

**语法糖**

通过 `$` 可以直接在字符串中添加表达式或变量值

````kotlin
val i = 10
println("i = $i") // 输出“i = 10”
val s = "abc"
println("$s.length is ${s.length}") // 输出 "abc.length is 3"
````

##### 6.无符号数

* `UByte`: 无符号 8 比特整数，范围是 0 到 255
* `UShort`: 无符号 16 比特整数，范围是 0 到 65535
* `UInt`: 无符号 32 比特整数，范围是 0 到 2^32 - 1
* `ULong`: 无符号 64 比特整数，范围是 0 到 2^64 - 1

#### 操作符

##### 1. is

> 操作符 `is` 用于类型检测

```kotlin
fun demo(x: Any) {
    if (x is String) {
        print(x.length) // x 自动转换为字符串
    }
}
```

##### 2. as

> 操作符 `as` 用于类型转换

不安全转换（转换操作符会抛出一个异常）：

````kotlin
val x: String = y as String
````

注意，`null` 不能转换为 `String`，所以如果 `y` 为空，上面的代码会抛出一个异常

安全转换：

```kotlin
val x: String? = y as? String
```

安全转换操作符 `as?`，可以在失败时返回 `null`

#### 关键字

##### 1. lateinit



#### 类与对象

##### 1. 继承

> `Kotlin` 中所有类都有一个共同的超类 `Any`，`Any` 有三个方法：`equals()`、`hashCode()`、`toString()`

声明类：

````kotlin
class Person {
    
}
````

默认情况下，`Kotlin` 中的类是 `final` 的，即不可被继承，可以使用 `open` 关键字标示该类可被继承

````kotlin
open class Person {
    
}
````

类的继承使用冒号 `:` 表示

````kotlin
open class Person {
    
}

class Student: Person() {
    
}
````

成员变量成员方法也同理，使用 `open` 标示可被重写，并且重写时使用 `override` 关键字修饰

```kotlin
open class Shape {
    open val vertexCount: Int = 0
    
    open fun draw() {
        
    }
    fun fill() {
        
    }
}

class Circle() : Shape() {
    override val vertexCount = 4
    
    override fun draw() {
        
    }
}
```

注意：可以用一个 `var` 变量覆盖一个 `val` 变量，但反之则不行，因为一个 `val` 属性本质上声明了一个 `get` 方法， 而将其覆盖为 `var` 只是在子类中额外声明一个 `set` 方法

##### 2. 构造函数

> `Kotlin` 中一个类可以有一个**主构造函数**以及一个或**多个次构造函数**
>
> 构造函数用 `constructor` 关键字标识

**主构造函数**

主构造函数是类头的一部分（主构造函数只包含参数声明，不能包含任何代码语句）

````kotlin
class Person constructor(name: String) {
    //...
}
````

初始化代码可以放到 `init` 初始化块中，在实例化类的过程中，`init` 块与成员变量的声明按照代码中的顺序执行，在这过程中均可使用主构造函数的参数（<u>可以理解为这就是主构造函数的一部分</u>）

````kotlin
class Person constructor(name: String) {
    val firstProperty = "First property: $name"

    init {
        println("First initializer block that prints $name")
    }

    val secondProperty = "Second property: ${name.length}"

    init {
        println("Second initializer block that prints ${name.length}")
    }
}
````

**如果在构造函数中的参数被 val 或 var 修饰，那么该参数也会作为类的成员变量**

```kotlin
class Person constructor(
    val firstName: String,
    val lastName: String,
    var age: Int,) { 
    //...
}
```

**次构造函数**

在类的内部使用 `constructor` 声明次构造函数，`Kotlin` 规定：<u>当一个类既有主构造函数又有次构造函数，所有次构造函数都必须使用`this`关键字直接或间接调用主构造函数</u>

````kotlin
class Person(var name: String, var age: Int) {
    init {
        println("main constructor init")
    }

    constructor(name: String, age: Int, sex: String) : this(name, age) {

    }

    constructor(name: String, age: Int, sex: String, idCard: String) : this(name, age, sex) {

    }
}
````

初始化块中的代码实际上会成为主构造函数的一部分，对主构造函数的委托发生在访问次构造函数的第一条语句，因此所有初始化块与属性初始化器中的代码都会在次构造函数体之前执行，即使该类没有主构造函数，这种委托仍会隐式发生，并且仍会执行初始化块

```kotlin
class Person {
    init {
        println("Init block")//先打印
    }

    constructor(i: Int) {
        println("Constructor $i")//后打印
    }
}
```

与 `Java` 一样，如果一个非抽象类没有声明任何（主或次）构造函数，它会有一个生成的不带参数的主构造函数，构造函数的可见性默认是 `public`

##### 3. 成员变量

> 成员变量可用 `var` 声明为可读可写的，也可用 `va` 声明为只读的

```kotlin
class Address {
    var name: String = "Hello World"
}
```

使用属性时，使用 `对象名.变量名` 进行读写

````kotlin
val result = Address()//Kotlin中没有new关键字
result.name = "new name"//将调用访问器
println(result.name)
````

声明一个成员变量的完整语法如下

```kotlin
var <propertyName>[: <PropertyType>] [= <property_initializer>]
    [<getter>]
    [<setter>]
```

其中 `property_initializer`、`getter`、`setter` 都是可选的，当然也可以自己定义

```kotlin
val isEmpty: Boolean
    get() = this.size == 0
```

一般来说，不论是 `var` 还是 `val`，声明为非空类型的成员变量必须在构造函数中初始化，但往往不能在构造函数提供变量的初始化，此时有如下两种方法：

* 声明为可空：此时引用该变量时需要频繁的空检测

* 使用 `lateinit` 修饰符：

  ```kotlin
  public class Student {
      lateinit var subject: Subject
  }
  ```

  注意：该变量必须为非空类型，并且不能是原生类型，在初始化前访问 `lateinit` 变量会抛出一个特定异常，该异常明确标识该属性被访问及它没有初始化的事实

  要检测一个 `lateinit var` 是否已经初始化过，使用 `.isInitialized`：

  ```kotlin
  if (subject.isInitialized) {
      println(subject)
  }
  ```

##### 4. 编译期常量

如果只读属性的值在编译期是已知的，那么可以使用 `const` 修饰符标记为编译期常量，编译期常量需要满足以下条件

* 必须位于顶层或 `object` 块中的一个成员
* 必须以 `String` 或原生类型值初始化
* 不能有自定义 `getter`

````kotlin
const val HELLO_WORLD: String = "Hello World"
````

##### 5. 扩展函数



##### 6. 数据类

> 只保存数据的类

`Kotlin` 中，以 `data` 标记数据类：

```kotlin
data class User(val name: String, val age: Int)
```

声明一个数据类时，编译器会自动从主构造函数中声明的所有属性生成一下函数：

- `equals() `/ `hashCode()` 
- `toString()`，格式：`"User(name=John, age=42)"`
- `componentN()` 函数按声明顺序对应于所有属性
- `copy()` 函数

`Kotlin` 标准库中提供了 `Pair`、`Triple` 数据类，这是数据类的标准实现，可以参考

#### 泛型

和 `Java` 泛型一样，`Kolin` 中的泛型本身也是不可变的

- 使用关键字 `out` 来支持协变，等同于 `Java` 中的上界通配符 `? extends`
- 使用关键字 `in` 来支持逆变，等同于 `Java` 中的下界通配符 `? super`





#### 协程





#### 委托

> 委托是 23 种设计模式中的一种，又名[代理模式](..\..\通用\设计模式\结-代理模式.md)
>
> 委托的思想是将一个类的一些具体实现委托给另一个类去完成，在 `Kotlin` 中通过 `by` 关键字实现

##### 1. 类委托

> 一个类的方法不在该类中定义，而是直接委托给另一个对象处理

类委托的语法格式：

```kotlin
class <类名>(b : <基础接口>) : <基础接口> by <基础对象>
```

`Kotlin` 的类委托可以减少 `Java` 中委托模式实现的大部分样板代码，例子：

````kotlin
// 基础接口
interface Base {   
    fun print()
}

// 基础对象
class BaseImpl(val x: Int) : Base {
    override fun print() { print(x) }
}

// 被委托类
class Derived(b: Base) : Base by b

fun main(args: Array<String>) {
    val b = BaseImpl(10)
    Derived(b).print() // 最终调用了 Base#print()
}
````

##### 2. 属性委托

属性委托的语法格式：

````kotlin
val/var <属性名> : <类型> by <基础对象>
````

例子：

````kotlin
class Example {
    // 被委托属性
    var prop: String by Delegate() // 基础对象
}

// 基础类
class Delegate {
    private var _realValue: String = "彭"

    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        println("getValue")
        return _realValue
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("setValue")
        _realValue = value
    }
}

fun main(args: Array<String>) {
    val e = Example()
    println(e.prop)    // 最终调用 Delegate#getValue()
    e.prop = "Peng"    // 最终调用 Delegate#setValue()
    println(e.prop)    // 最终调用 Delegate#getValue()
}
````

对于基础类，如果委托 `val` 属性，必须提供 `getValue()`；如果委托 `var` 属性，必须提供 `getValue()/setValue()`，原因：

````kotlin
源码：
class Example {
    // 被委托属性
    var prop: String by Delegate() // 基础对象
}
--------------------------------------------------------
编译器生成的字节码：
class Example {
    private val prop$delegate = Delegate()
    // 被委托属性
    var prop: String
        get() = prop$delegate.getValue(this, this:prop)
        set(value : String) = prop$delegate.setValue(this, this:prop, value)
}
````

实现属性委托时，除了定义基础类，还可以直接使用 `Kotlin` 标准库中的两个接口 `ReadOnlyProperty / ReadWriteProperty`