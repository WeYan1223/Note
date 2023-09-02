#### 修饰符

`Java` 提供了四种可见性修饰符：`public`、``、`protected` 以及默认修饰（即不加修饰）

> 最外层声明

最外层的只能被 `public` 修饰或不修饰

```java
/**
 * 随处可见
 */
public class Main {

}
```

````java
/**
 * 本包可见
 */
class Main {

}
````

> 类内部声明

- `public`：随处可见
- `private`：子类可见
- `protected`：类内部可见、子类可见
- 不修饰：本包可见

***

#### final

##### 1. 修饰类

当某个类被 `final` 修饰时，表明该类不能被继承

一般使用组合的方式来扩展 `final` 类，例如 `String`

````java
public class MyString {
    private String innerString;
    
    public int length() {
        return innerString.length();
    }
    
    public String toString() {
        return innerString.toString();
    }
    
    // 添加自己的方法
}
````

##### 2. 修饰方法

被 `final` 修饰的方法不可以被重写，但可以重载

````java
final void method() {
    
}
````

##### 3. 修饰参数

在参数列表将参数指明为 `final`

````java
void method(final int a) {
    
}
````

* 对于基本数据类型，无法修改它的值
* 对于引用数据类型，无法修改它的指向

##### 4. 修饰变量

* 对于基本数据类型，初始化之后无法修改它的值
* 对于引用数据类型，初始化之后无法修改它的指向

***

#### 拆装箱

##### 1. 简述

> `JDK 5` 开始，每种基本类型都提供了对应的包装类型

| 基本类型  | 包装类型  |
| --------- | --------- |
| `boolean` | `Boolean` |
| `byte`    | `Byte`    |
| `char`    | `Char`    |
| `short`   | `Short`   |
| `int`     | `Integer` |
| `float`   | `Float`   |
| `long`    | `Long`    |
| `double`  | `Double`  |

##### 2. 装箱

````java
public static void main(String[] args) {
    Integer i = 105;
}
````

反编译后发现这里调用了 `Integer` 的 `valueOf()`，如下

````java
public static void main(String[] args) {
    Integer i = Integer.valueOf(105);
}
````

这个加上 `valueOf()` 的过程，就是自动装箱过程

##### 3. 拆箱

```java
public static void main(String[] args) {
    Integer i = Integer.valueOf(105);
    int n = i;
}
```

反编译后发现这里调用了 `Integer` 的 `intValue()`，如下

````java
public static void main(String[] args) {
    Integer i = Integer.valueOf(105);
    int n = i.intValue();
}
````

这个加上 `intValue()` 的过程，就是自动拆箱

##### 4. 时机

* 赋值运算
* 方法调用
* 算术表达式

##### 5. 问题

* 判断打印结果

  ````java
  public static void main(String[] args) {
      Integer a = 100;
      Integer b = 100;
      Integer c = 200;
      Integer d = 200;
      System.out.println(a == b);//true
      System.out.println(c == d);//false
  }
  ````

  重点在于 `valueOf()`

  ````java
  public static Integer valueOf(int i) {
      if (i >= IntegerCache.low && i <= IntegerCache.high)
          return IntegerCache.cache[i + (-IntegerCache.low)];
      return new Integer(i);
  }
  ````

  可以发现，如果 `-128 <= i <= 127`，则返回的是缓存中的 `Integer` 对象，所以上述代码中 `a` 与 `b` 指向的是同一个对象

  判断其他包装类型是否相等主要关注对应的 `valueOf()`

* 下面两种方式的区别

  ````java
  Integer a = 10;//方式一
  Integer b = new Integer(10);//方式二
  ````

  方式一会触发自动装箱过程，方式二则不会

  方式一不一定会创建新对象，方式二必定创建新对象

* 判断打印结果

  ````java
  public static void main(String[] args) {
      Integer a = 100;
      Integer b = 100;
      Integer c = a + b;
  }
  ````
  

首先 `a + b` 算数表达式会进行拆箱操作，得到 `int` 数值 200，然后进行装箱得到值为 200 的 `Integer` 对象

````java
  public static void main(String[] args) {
      Long c = 200L;
      System.out.println(c.equals(200));//false
      System.out.println(c.equals(200L));//true
  }
````

在上述代码中，200 被装箱成 `Integer`，`200L` 被装箱成 `Long`

***

#### Equals

> `equals()` 是 `Object` 的一个方法，`==` 是操作符

````java
public boolean equals(Object obj) {
    return (this == obj);
}
````

对于不同类型的数据，`==` 的比较作用不同：

* 基本类型：比较的是**值**是否相等
* 引用类型：比较的是**对象所在堆内存地址**是否相同（或者说是否指向同一个对象）

所以，使用 `equals` 判断某个类的实例是否相等，需要注意该类是否重写了 `equals()`

重写 `equals()` 的约定：

* 自反性：对于非空引用 `x`，`x.equals(x) == true`
* 对称性：对于非空引用 `x`、`y`，若 `x.equals(y) == true`，则 `y.equals(x) == true`
* 传递性：对于非空引用 `x`、`y`、`z`，若 `x.equals(y) == true` 且 `y.equals(x) == true`，则 `x.equals(z) == true`
* 一致性：对于非空引用 `x`、`y`，`x.equals(y)` 的多次调用始终返回同一个布尔值（前提是没有修改 `equals()` 中比较操作使用到的信息）
* 非空性：对于非空引用 `x`，`x.equals(null) == false`

同时，**重写 `equals()` 必须重写 `hashCode()`**，关于重写 `hashCode()`，`JavaSE` 给出如下约定：

* 程序执行期间，只要对象的 `equals()` 的比较操作所用到的信息没有被修改，那么对这同一个对象调用多次 `hashCode` 方法，必须始终返回同一个整数
* 两个对象根据 `equals()` 方法是相等的，那么调用这两个对象任一个 `hashCode()` 必须返回同样的整数
* 两个对象根据 `equals()` 方法是不相等，但是两个对象对象的 `hashCode()` 返回结果可能相等

**问题**

* `Integer` 用 `==` 与 `equals()` 区别

  由于 `Integer` 是引用类型，所以 `==` 比较的是 `Integer` 对象所在的堆内存地址是否相同，同时需要注意的是，采用 `Integer.valueOf(int)` 在 `-128 ~ 127` 获取到的 `Integer` 对象是缓存中的对象

  `Integer` 重写了 `equals()`

  ````java
  public boolean equals(Object obj) {
      if (obj instanceof Integer) {
          return value == ((Integer)obj).intValue();
      }
      return false;
  }
  ````

  所以 `equals()` 比较的是 `Integer` 中的 `int` 值是否相等

* `String` 用 `==` 与 `equals()` 区别

  `String` 重写了 `equals()`

  ````java
  public boolean equals(Object anObject) {
      if (this == anObject) {
          return true;
      }
      return (anObject instanceof String aString)
          && (!COMPACT_STRINGS || this.coder == aString.coder)
          && StringLatin1.equals(value, aString.value);
  }
  ````

  当且仅当参数不为 `null` 并且是表示与此对象相同的字符序列的 `String` 对象时，结果才为 `true`

  由于 `String` 是引用类型，所以 `==` 比较的是 `String` 对象所在的堆内存地址是否相同，但是需要注意指向的是字符串常量池中的对象还是堆中字符串常量池外的对象

***

#### 内部类

> 在 `Java` 中，将类定义在另一个类或者方法中，这样的类被称为内部类

##### 1. 成员内部类

```java
class Outer {
    private int value = 0;
    
    class Inner {//成员内部类
        public int i = 0;
    }
}
```

如上，`Inner` 即为成员内部类，可以无条件访问外部类的所有成员变量和成员方法

但是，如果内部类拥有跟外部类同名的成员变量或成员方法时，需要使用以下形式访问外部类

* `外部类.this.成员变量`
* `外部类.this.成员方法`

外部类访问成员内部类，需要先实例化该内部类，再通过引用来访问

````java
public static void main(String[] args) {
    Outer outer = new Outer();
    Outer.Inner inner = outer.new Inner();
}
````

**Q：**为什么成员内部类可以直接访问外部类的变量与方法

**A：**首先通过 `javap -v Outer.Inner` 反编译

````java
public int i;

final Outer this$0;

Outer$Inner(Outer);
````

可以发现，`Inner` 中的成员变量除了在 `Java` 代码中的声明的 `i` 之外，还有一个 `final` 修饰的 `Outer` 类型变量

除此之外，虽然定义的 `Inner` 类是无参构造方法，但是编译器还是会添加一个参数，该参数为指向外部类的一个引用

##### 2. 局部内部类

> 局部内部类是定义在方法或作用域内，与成员内部类的区别在于局部内部类的访问仅限于方法或作用域

##### 3. 匿名内部类

````java
new Thread(new Runnable() {//匿名内部类
    @Override
    public void run() {
        //...
    }
}).start();
````

**Q：**为什么局部内部类与匿名内部类只能访问 `final` 类型变量

**A：**首先，对于下面的代码，编译器不会报错

````java
public class Main {
    public static void main(String[] args) {
        int a = 0;
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(a);
            }
        }).start();
    }
}
````

这里的 `a` 虽然没有 `final` 修饰，但是编译器会检查声明 `a` 之后是否对 `a` 进行写操作，如果没有

````java
public class Main {
    public Main() {
    }

    public static void main(String[] var0) {
        final byte var1 = 0;
        (new Thread(new Runnable() {
            public void run() {
                System.out.println(var1);
            }
        })).start();
    }
}
````

可以看到，编译后编译器帮忙加上了 `final`

如果之后有对 `a` 的写操作，那么编译器会报错

首先，内部类跟外部类处于同一级别，即使是匿名内部类也不会因为在方法中定义而随着方法的执行完毕而被销毁

这里就会产生一个问题：当方法执行完毕时，局部变量就会被销毁，但是内部类对象可能还存在，内部类对象就会访问一个不存在的变量

为了解决该问题，将局部变量复制一份作为内部类的成员变量，这样当局部变量被销毁后，内部类依然可以访问它，通过反编译可以证明这一点（编译后的匿名内部类的类名为：外部类名$数字）

````java
class Main$1 implements java.lang.Runnable {
    final int val$a;
    descriptor: I;

    other.Main$1();
    descriptor: (I)V;
}
````

可以发现，匿名内部类中确实存在 `final` 类型的 `a`，而且构造方法多了一个 `int` 类型的参数（实际上就是用来初始化 `a` 的）

但是又出现问题了：将局部变量复制为内部类的成员变量时，必须保证这两个变量是一样的（如果我们在方法中修改了局部变量，内部类的成员变量理应改变），这该如何解决呢

解决方法是将局部变量设置为 `final`，初始化后，就不让再修改该变量，保证了内部类的成员变量和方法的局部变量的一致性（实际上也是一种妥协）

关于局部变量的复制

* 若是基本类型，其值是不能改变的，就保证了 `copy` 与原始的局部变量的值是一样的
* 若是引用类型，其引用是不能改变的，保证了 `copy` 与原始的变量引用的是同一个对象

**Lambda表达式**

说到匿名内部类，就不得不提及 `Lambda` 表达式了

观察以下两种写法的字节码区别

````java
//匿名内部类
new Thread(new Runnable() {
    @Override
    public void run() {
        
    }
}).start();

//Lambda表达式
new Thread(() -> {
    
}).start();
````

````sh
//匿名内部类
invokespecial #13                 // Method other/Main$1."<init>":()V
invokespecial #16                 // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
invokevirtual #19                 // Method java/lang/Thread.start:()V

//Lambda表达式
invokedynamic #11,  0             // InvokeDynamic #0:run:()Ljava/lang/Runnable;
invokespecial #15                 // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
invokevirtual #18                 // Method java/lang/Thread.start:()V
````

* 匿名内部类：先实例化匿名内部类 `Main$1`（实际上是 `Runnable` 的子类），再实例化 `Thread`，再调用 `start()`
* `Lambda` 表达式：调用某个类（使用 `ASM` 在运行时生成的类）的 `run()` 返回 `Runnable` 对象，实例化 `Thread`，再调用 `start()`

##### 4. 静态内部类

> 静态内部类是定义在另一个类里的 `static` 修饰的类

````java
class Outer {
    private int value = 0;
    
    static class Inner {//静态内部类
        public int i = 0;
    }
}
````

***

#### 语法糖

> 语法糖指 `Java` 编译器在编译成字节码的过程中，自动生成和转换的一些代码，主要是为了减轻开发人员的工作

##### 1. 默认构造

````java
public class Student {
    int id;
}
````

编译后的 `.class` 文件实际上如下

````java
public class Student {
    int id;

    //这个无参构造方法是编译器帮助加上的
    public Student() {
        super();
    }
}
````

##### 2. [自动拆装箱](#拆装箱)

> 自动拆装箱从 `JDK 5` 开始加入

##### 3. foreach

对于数组：

```java
public static void main(String[] args) {
    int[] nums = new int[]{1, 2, 3, 4};
    for (int num : nums) {
        System.out.println(num);
    }
}
```

编译后的 `.class` 文件实际上如下

```java
public static void main(String[] args) {
    int[] nums = new int[]{1, 2, 3, 4};
    int[] var2 = nums;
    int var3 = nums.length;

    for(int var4 = 0; var4 < var3; ++var4) {
        int num = var2[var4];
        System.out.println(num);
    }
}
```

对于集合：

```java
public static void main(String[] args) {
    List<Integer> nums = new ArrayList<>();
    nums.add(1);
    nums.add(2);
    nums.add(3);
    nums.add(4);
    for (int num : nums) {
        System.out.println(num);
    }
}
```

编译后的 `.class` 文件实际上如下

```java
public static void main(String[] args) {
    List<Integer> nums = new ArrayList();
    nums.add(1);
    nums.add(2);
    nums.add(3);
    nums.add(4);
    Iterator var2 = nums.iterator();

    while(var2.hasNext()) {
        int num = (Integer)var2.next();
        System.out.println(num);
    }
}
```

可以看出，对于集合的 `foreach` 最终是使用迭代器模式实现的

注意，不能再 `foreach` 中对集合中的元素进行添加或删除，原因在于 `Iterator#next()` 中开头有如下代码

````java
public E next() {
    //检查是否修改
    checkForComodification();
    //..
}
````

##### 4. switch-String

```java
public static void f(String s) {
    switch (s) {
        case "a":
            System.out.println("a");
            break;
        case "b":
            System.out.println("b");
            break;
        default:
            break;
    }
}
```

编译后的 `.class` 文件实际上如下

````java
public static void f(String s) {
    byte var2 = -1;
    switch(s.hashCode()) {
        case 97:
            if (s.equals("a")) {
                var2 = 0;
            }
            break;
        case 98:
            if (s.equals("b")) {
                var2 = 1;
            }
    }

    switch(var2) {
        case 0:
            System.out.println("a");
            break;
        case 1:
            System.out.println("b");
    }

}
````

***

#### 三大特性

##### 1. 封装

**定义**

利用抽象数据类型将数据和基于数据的操作封装在一起，使其构成一个不可分割的独立实体，数据被保护在抽象数据类型的内部，尽可能地隐藏内部细节，只保留一些对外接口使其与外部联系

**优点**

* 减少耦合：可以独立开发、测试、优化、使用、修改
* 提高可维护性：调试时可以不影响其他模块
* 有效地调节性能：通过剖析确定哪些模块影响了系统的性能
* 提高可重用性
* 降低构建大型系统的风险：即使整个系统不可用，独立模块却有可能是可用的

##### 2. 继承

继承实现了 `IS - A` 关系，例如 `Cat` 和 `Animal` 就是 `IS - A` 关系，因此 `Cat` 可以继承自 `Animal`，从而获得 `Animal` 的非 `private` 变量和方法

继承应遵循里氏替换原则，只要有父类出现的地方，都可以用子类来替代，而且不会出现任何错误与异常

##### 3. 多态

> 编译时多态：重载是编译时多态的一个例子，编译时多态在编译时就已经确定调用的是哪个方法
>
> 运行时多态：通常说的多态指的是运行时多态，编译时不确定调用的是哪个方法，一直延迟到运行时才能确定
>
> 下面说的多态都是指运行时多态

多态指，针对某个类的方法调用，其真正执行的方法在运行时才能确定

````java
class Person {
    public void run() {
        System.out.println("Person.run");
    }
}

class Student extends Person {
    @Override
    public void run() {
        System.out.println("Student.run");
    }
}
````

两个类的关系如上，现在观察下面代码

````java
public static void main(String[] args) {
    Person p = new Student();
    p.run();//应该打印Person.run还是Student.run
}
````

实际上打印结果为 `Student.run`，但是上面的例子一看就知道调用的是 `Student` 的 `run()`，如果编写如下方法呢

````java
public void runWitch(Person p) {
    p.run();
}
````

这时传入的参数为 `Person`，我们无法知道传入的实际类型为 `Person`、`Student` 还是 `Person` 的其他子类

所以，多态性就是运行时才能动态地确定调用哪个方法，调用某个类的方法，执行的实际方法可能是子类重写后的方法

多态的条件：

* 子类继承父类
* 子类重写父类方法
* 父类引用指向子类对象，即 `父类类名 引用名称 = new 子类类名()` 

***

#### 深浅拷贝

> 在 `Java` 中，存在**基本类型**与**引用类型**，在使用 `=` 进行赋值操作的时候
>
> * 对于基本类型，实际上是拷贝它的值
> * 对于引用类型，实际上是赋值对象的引用地址，它们实际上指向的还是同一个对象
>
> 而深浅拷贝都是对对象的拷贝，会在堆中划分一块新的区域，区别在于其引用类型的成员变量的拷贝

首先看 `Object` 的 `clone()`

````java
protected native Object clone() throws CloneNotSupportedException;
````

可以这样理解

````java
protected Object clone() throws CloneNotSupportedException {
    if (!(this instanceof Colneable)) {
        throw new CloneNotSupportedException();
    }
    return internalClone();
}

private native Object internalClone() throws CloneNotSupportedException;
````

所以，这里有一个限制：调用 `clone()` 的对象必须实现 `Cloneable` 接口

````java
public interface Cloneable {
    
}
````

可以看到，`Cloneable` 是一个空接口，可以理解为只是一个标记，开发者允许该对象被拷贝

##### 1. 浅拷贝

> 新对象中的引用类型成员变量跟旧对象的指向的**是**用一个对象

````java
class Father implements Cloneable {
    public int age;
    public Son son;

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}

class Son {

}
````

```java
public static void main(String[] args) {
    Father father = new Father();
    father.age = 22;
    father.son = new Son();
    Father newFather = (Father) father.clone();
    System.out.println("father: " + father + ", son: " + father.son);
    System.out.println("newFather:" + newFather + ", son: " + newFather.son);
}
//打印结果
//father: Father@16b98e56, son: Son@7ef20235
//newFather: Father@27d6c5e0, son: Son@7ef20235
```

##### 2. 深拷贝

> 新对象中的引用类型成员变量跟旧对象的指向的**不是**用一个对象

````java
class Father implements Cloneable {
    public int age;
    public Son son;

    @Override
    protected Object clone() throws CloneNotSupportedException {
        Father father = (Father) super.clone();
        father.son = (Son) son.clone();
        return father;
    }
}

class Son implements Cloneable {
    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
````

````java
public static void main(String[] args) {
    Father father = new Father();
    father.age = 22;
    father.son = new Son();
    Father newFather = (Father) father.clone();
    System.out.println("father: " + father + ", son: " + father.son);
    System.out.println("newFather:" + newFather + ", son: " + newFather.son);
}
//打印结果
//father: Father@16b98e56, son: Son@7ef20235
//newFather: Father@27d6c5e0, son: Son@4f3f5b24
````

虽然这样可以实现深拷贝，一旦引用类型的成员变量很多，每个都需要实现 `Cloneable` 接口、重写 `clone()`，当嵌套对象过多时，这种方法显然不合理

此时更好的做法是使用序列化与反序列化

***

#### 重载重写

##### 1. 重载

方法重载就是在类中可以创建多个方法，它们具有相同的名字，但具有不同的参数个数或参数类型，调用方法时通过参数来决定具体使用哪个方法，这就是编译时多态（注意：无法通过返回类型来区分方法）

重载规则：

* 必须有具有不同的参数列表
* 返回值可以不同
* 访问修饰符可以不同
* 可以抛出不同异常

##### 2. 重写

在继承关系中，子类可以继承父类中非 `private` 修饰的方法，如果子类想对父类中的方法做一定的修改，这时候需要采用方法重写

重写规则：

* 参数列表必须相同
* 返回类型必须相同或者是子类
* 重写方法的访问修饰符必须大于等于原访问修饰符
* 抛出的异常必须是原异常或其子类
* 静态方法不可重写
* `final` 方法不可重写