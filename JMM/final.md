#### 使用

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