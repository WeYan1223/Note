#### 简述

> 历史发展

在 `JDK 1.5` 之前，集合类中可以存放任何类型的元素，对集合中的元素进行统一操作时，非常容易引发 `ClassCastException`

最终解决方案：引入泛型，<u>将具体的类型泛化，声明时用符号代替具体类型，使用时再确定具体类型</u>

注意，`Java` 中的泛型是 `Java` 代码层面的语法糖，字节码层面是不存在泛型的

泛型可以保证：<u>如果程序在编译时没有发出警告，运行时就不会产生 `ClassCastException` 异常</u>

> 泛型类

定义泛型类：

````java
public class Generic<T> {
    private T key;
    
    private Generic(T key) {
        this.key = key;
    }
    
    public T getKey() {
        return key;
    }
}
````

使用泛型类：

````java
// 实例化类时指明泛型的具体类型
Generic<Integer> genericInteger = new Generic<Integer>(1);

// Java 9 之后，可以省略构造方法后的泛型信息，写成如下形式
Generic<Integer> genericInteger = new Generic<>(1);
````

> 泛型接口

定义泛型接口：

````java
public interface Generator<T> {
    public T next();
}
````

实现泛型接口：

````java
// 未指定具体类型，待实例化类时指定
public class FruitGenerator<T> implements Generator<T>{
    @Override
    public T next() {
        
    }
}

// 指定具体类型
public class FruitGenerator implements Generator<String> {
    @Override
    public String next() {
        
    }
}
````

> 泛型方法：调用方法时指明泛型的具体类型

定义泛型方法：

````java
public <T> void f(T t) {
    System.out.println(t);
}
````

区分泛型方法：

````java
public class Generic<T> {
    private T key;

    /**
     * 该方法虽然使用了泛型，但不是泛型方法
     * 这只是泛型类的普通方法
     */
    public T getKey() {
        return key;
    }
    
    /**
     * 这是泛型方法，但这里的 T 与定义泛型类的 T 不是同一个东西
     * 所以为了避免混淆，如果在泛型类中存在泛型方法，两者的泛型参数最好不要同名
     */
    public <T> void println(T t) {
        System.out.println(t);
    }
}
````

***

#### 通配符

> 声明通配符：声明泛型类、接口、方法时使用的通配符

常用的有 `T`、`E`、`K`、`V`，实际怎么样写都行，只不过这几个比较常用，属于是编码是约定俗成的规范

* `T (Type)`，表示具体的 `Java` 类型
* `E (Element)`，表示元素的类型
* `KV (Key, Value)`，表示键值对类型

````java
public class Generic<T> {

}
````

> 使用通配符：使用泛型类、接口、方法时使用的通配符

只有一个：`?`，表示不确定的 `Java` 类型

````java
List<?> list = new ArrayList<>();
````

单个 `?` 使用没什么意义，通常需要与[上下限](#上下限)配合使用

***

#### 上下限

> 与声明通配符配合使用

````java
public class Generic<T extends Number> {
    
}
````

这规定了实例化 `Generic` 类时，传入的泛型类型必须为 `Number` 或其子类

````java
Generic<Number> a = new Generic<>(); // 编译通过
Generic<Integer> b = new Generic<>(); // 编译通过
Generic<String> c = new Generic<>(); // 编译不通过：String 不是 Number 子类
````

`super` 同理，规定传入的类型必须为 `Number` 或其父类

> 与使用通配符配合使用

````java
public class Generic<T> {
    private T a;

    public T getA() {
        return a;
    }

    public void setA(T a) {
        this.a = a;
    }
}
````

实例化泛型类

````java
//方式一：? extends Number，表示传入的具体类型未知，但是肯定是 Number 或其子类
Generic<? extends Number> g1 = new Generic<>();

//方式二：? super Number，表示传入的具体类型未知，但是肯定是 Number 或其父类
Generic<? super Number> g2 = new Generic<>();
````

对于方式一，访问 `Generic` 的成员变量 `a`

````java
Generic<? extends Number> g1 = new Generic<>();
Number a1 = g1.getA(); // 编译通过：父类引用可以指向子类实例
Integer a2 = g1.getA(); // 编译不通过：Number 有许多子类，可能导致类型错误
g1.setA(Integer.valueOf(1)); // 编译不通过：Number 有许多子类，可能导致类型错误
g1.setA(new Number()); // 编译不通过：Number 有许多子类，子类引用不能指向父类实例
````

对于方式二，访问 `Generic` 的成员变量 `a`

````java
Generic<? super Number> g2 = new Generic<>();
Number a1 = g2.getA(); // 编译不通过：子类引用不能指向父类实例
Integer a2 = g2.getA(); // 编译不通过：子类引用不能指向父类实例
g2.setA(Integer.valueOf(1)); // 编译通过：父类引用可以指向子类对象
g2.setA(new Number()); // 编译通过：父类引用可以指向子类对象
````

上面的编译不通过主要有两点原因：

* 类型强转是不安全的
* 子类引用不能指向父类对象

除此之外，还有一种情况

````java
//相当于<? extends Object>
Generic<?> g1 = new Generic<>();
````

 `?` 通配符在声明变量时是没什么意义的，但是为方法声明参数时，能起到重要作用，具体参考[协变逆变](#协变逆变)

***

#### 协变逆变

##### 1. 协变

> 定义：如果 A 是 B 的子类，那么 F(A) 是 F(B) 的子类

`Java` 数组支持协变，所以如果 `Cat` 是 `Animal` 的子类，那么 `Cat[]` 也是 `Animal[]` 的子类，看如下代码

````java
Animal[] animals = new Cat[2];
animals[0] = new Cat();
animals[1] = new Dog();
````

代码编译没有任何问题，一旦运行 100% 抛出 `ArrayStoreException<u>`

`Java` 数组是协变的，会有些问题，所以泛型是不变，也就是说，`List<Cat>` 并不是 `List<Animal>` 的子类型

````java
List<Animal> list = new ArrayList<Cat>(); // 编译失败
````

泛型是不变，能够更加安全，`List<Cat>` 不是 `List<Animal>` 的子类型，但 `Java` 提供了其它方式来支持泛型的协变，使得 `List<Cat>` 是 `List<? extends Animal>` 的子类型，同时在编译器层面通过禁止写入的方式，保证了协变下的安全

````java
public void printAnimal(List<? extends Animal> list) {
    for (Animal animal : list) { // 每次从这个集合取出来的一定是 Animal 或其子类
        System.out.println(animal);
    }
}
````

##### 2. 逆变

> 与协变相反， `Cat` 是 `Animal` 的子类，但 `List<Animal>` 是 `List<? super Cat>` 的子类型

有一个集合，希望可以往里面写入 `Animal` 及其子类

````java
public void setAnimals(List<? super Animal> list) {
    list.add(new Cat());
    list.add(new Dog());
    list.add(new Animal());
}
````

***

#### 类型擦除

> `Java` 泛型是在 `Java` 代码层面实现的，编译之后不会保留任何泛型信息，代码中定义 `List<Object>` 和 `List<String>` 等类型，编译后都会变成 `List`

##### 1. 为什么要擦除

为了兼容 `JDK5` 之前的项目，否则有大量代码修改的工作

##### 2. 证明类型擦除

* 原始类型相等

  ````java
  public static void main(String[] args) {
      List<String> stringList = new ArrayList<>();
      List<Integer> integerList = new ArrayList<>();
      System.out.println(stringList.getClass() == integerList.getClass());
  }
  //打印结果：true
  ````

* 通过反射添加其他类型元素

  ````java
  public static void main(String[] args) {
      List<String> stringList = new ArrayList<>();
      stringList.add("123");
      stringList.getClass().getMethod("add", Object.class).invoke(stringList, 1234);
      System.out.println(stringList.size());
      for (int i = 0; i < stringList.size(); i++) {
          System.out.println(stringList.getClass().getMethod("get", int.class).invoke(stringList, i));
      }
}
  // 打印结果：
  // 123
  // 1234
  ````
  

##### 3. 擦除后的类型

原来：

````java
public class Pair<T> {  
    private T value;  
    public T getValue() {  
        return value;  
    }  
    public void setValue(T  value) {  
        this.value = value;  
    }  
}  
````

擦除后：

````java
public class Pair {  
    private Object value;  
    public Object getValue() {  
        return value;  
    }  
    public void setValue(Object  value) {  
        this.value = value;  
    }  
}
````

若 `public class Pair<T extends Comparable> {}`，则擦除后的类型为 `Conparable`

***

#### 泛型数组

> 数组元素的类型不能包含泛型变量或泛型形参，除非是无上限通配符

假如 `Java` 支持创建泛型数组，如下

````java
List<String>[] strListArray = new ArrayList<>()[10];//假如这是合法的
Object[] objArray = strListArray;
List<Integer> integerList = new ArrayList<>();
integerList.add(3);
objArray[1] = integerList;
String s = strListArray[1].get(0);
````

最后一行必定引发 `ClassCastException`

***

#### 常见问题

##### 1. 自动类型转换

**Q**：既然所有泛型信息在编译后都会被替换为原始类型，那为什么在获取的时候，不需要进行强制类型转换

**A**：看 `ArrayList.get()` 方法

````java
public E get(int index) {
    Objects.checkIndex(index, size);
    return elementData(index);
}

E elementData(int index) {
    return (E) elementData[index];
}
````

在 `return` 之前，会根据泛型信息进行强转：假设泛型信息为 `String`，那么编译后为

````java
String elementData(int index) {
    return (String) elementData[index];
}
````

##### 2. 多态冲突问题

首先有这样一个泛型类

```java
static class Parent<T> {
    private T value;

    public T getValue() {
        return value;
    }

    public void setValue(T value) {
        this.value = value;
    }
}
```

然后一个子类继承它，并设定父类的泛型类型为 `String`

```java
static class Child extends Parent<String> {
    @Override
    public String getValue() {
        return super.getValue();
    }

    @Override
    public void setValue(String value) {
        super.setValue(value);
    }
}
```

可以看到子类重写了父类的两个方法，从 `@Override` 也可以得出这是重写

实际上，类型擦除后，父类应该是这个样子

````java
static class Parent {
    private Object value;

    public Object getValue() {
        return value;
    }

    public void setValue(Object value) {
        this.value = value;
    }
}
````

此时再回看子类重写的两个方法，发现类型不一样，在普通的继承关系中，这根本不是重写而是重载

````java
public static void main(String[] args) {
    Child child = new Child();
    child.setValue(new String());
    child.setValue(new Object());//编译报错
}
````

但是，如果是重载，上面的代码就不应该编译错误，所以说，这的的确确是子类重写父类方法

为什么会这样呢，首先反编译子类的字节码，会发现里面存在 4 个方法

````java
public reflection.Child();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method reflection/Parent."<init>":()V
       4: return

  public java.lang.String getValue();
    Code:
       0: aload_0
       1: invokespecial #7                  // Method reflection/Parent.getValue:()Ljava/lang/Object;
       4: checkcast     #11                 // class java/lang/String
       7: areturn

  public void setValue(java.lang.String);
    Code:
       0: aload_0
       1: aload_1
       2: invokespecial #13                 // Method reflection/Parent.setValue:(Ljava/lang/Object;)V
       5: return

  public void setValue(java.lang.Object);
    Code:
       0: aload_0
       1: aload_1
       2: checkcast     #11                 // class java/lang/String
       5: invokevirtual #17                 // Method setValue:(Ljava/lang/String;)V
       8: return
           
  public java.lang.Object getValue();
    Code:
       0: aload_0
       1: invokevirtual #22                 // Method getValue:()Ljava/lang/String;
       4: areturn
````

最后两个方法，就是编译器自己生成的桥方法，也就是说子类真正重写父类的两个方法就是这两个我们看不到的桥方法（在 `JVM` 中，函数签名 = 返回值 + 方法名 + 参数）

