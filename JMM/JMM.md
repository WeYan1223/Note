#### 简述

`JMM (Java Memory Model)` 是 `Java` 语言定义的一套规范：描述多线程并发访问共享变量的行为准则，抽象示意图如下

<img src="https://raw.githubusercontent.com/WeYan1223/Pic/master/JMM/JMM_模型.webp" alt="JMM_模型.webp (425×443) (raw.githubusercontent.com)" style="zoom:80%;" /> 

这里主要关注以下三个抽象概念：

* 主内存 `(Main Memory)`：所有线程共享的内存区域，包含了所有的变量数据
* 工作内存 `(Work Memory)`：每个线程独立拥有的内存区域，用于存储主内存中的部分变量副本
* 主内存与工作内存间的交互：线程在执行过程中，会将主内存中的数据拷贝到工作内存中进行操作，然后将结果写回主内存

通过 `JMM` 定义的一系列规范来控制主内存与工作内存间的交互，从而保证指令执行的正确性（原子性、可见性、有序性）

* 原子性 `(Atomicity)`：一个操作或者多个操作，要么全部执行并且不受到任何因素的干扰而中断，要么都不执行
* 可见性 `(Visibility)`：当一个线程对共享变量进行了修改，那么另外的线程都是立即可以看到修改后的最新值
* 有序性 `(Ordering)`：程序执行的顺序按照代码的先后顺序执行（由于指令重排序问题，代码的执行顺序未必就是编写代码时候的顺序）

`Java ` 提供了一系列和并发处理相关的工具，比如 `volatile`、`synchronized`、`final` 关键字和 `java.util.concurren` 包等，这就是 `JMM` 封装了底层实现后提供给开发者使用的工具

***

#### 原子性

> 一个操作或者多个操作，要么全部执行并且不受到任何因素的干扰而中断，要么都不执行

`Java` 中，借助 `synchronized` 关键字、各种 `Lock` 以及各种原子类实现原子性

***

#### 可见性

> 当一个线程对共享变量进行了修改，那么另外的线程可以立即感知到该修改

`Java` 中，借助 `synchronized`、`volatile` 关键字以及各种 `Lock` 实现可见性

***

#### 有序性

> 程序执行的顺序按照代码的先后顺序执行

由于指令重排序问题，代码的执行顺序未必就是编写代码时候的顺序，`Java` 中，`volatile` 关键字可以禁止指令重排

***

#### 指令重排

> 指编译器和处理器为了优化程序性能而对指令序列进行重新排序的一种手段

常见的指令重排有如下两种情况：

- 编译器优化重排：编译器在不改变单线程程序语义的前提下，重新安排语句的执行顺序
- 指令并行重排：指令级并行技术将多条指令重叠执行，如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序

所以，`Java` 代码会经历<u>编译器优化重排 —> 指令并行重排</u>的过程，最终才变成操作系统可执行的指令序列

指令重排可以保证单线程下的语义一致，但是不能保证多线程间的语义也一致 ，所以在多线程下指令重排序可能会导致一些问题

编译器和处理器的指令重排序的处理方式不一样：

* 对于编译器，通过禁止特定类型的编译器重排序的方式来禁止重排序
* 对于处理器，通过插入内存屏障的方式来禁止特定类型的处理器重排序

下面是一个未完善的双重检测锁（`DCL Double-Check-Lock`）实现单例

````java
public class Singleton {
    private static Singleton INSTANCE = null;
    
    private Singleton() {
        
    }
    
    public static Singleton getInstance() {
        //INSTANCE为空，准备进入synchronized代码块
        if (INSTANCE == null) {
            //此时如果有其他线程进入代码块，那么阻塞自己
            synchronized (Singleton.class) {
                //可能有其他线程已经创建实例了，所以需要二次判断
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
}
````

这样写看起来天衣无缝，但是这里面也有可能出现指令重排

````java
INSTANCE = new Singleton();
````

上面这行代码对应的字节码如下

````java
17: new           #3                  //为Singleton对象分配空间，并把其引用放入操作数栈
20: dup                               //将操作数栈顶元素复制一份，继续压入操作数栈
21: invokespecial #4                  //调用构造方法进行初始化
24: putstatic     #2                  //将引用赋值给INSTANCE
````

对于21、24行的字节码指令，执行顺序是有可能被重新排序的，那么就有可能出现下面这种情况

1. 线程 1 判断 `INSTANCE` 为空，并且获取到 `monitor`，成功进入 `synchronized` 代码块
2. 线程 1 二次判断 `INSTANCE` 依然为空，执行 `INSTANCE = new Singleton();`
3. 当线程 1 执行完第 24 行字节码（由于指令重排，并未执行 21 行），时间片结束
4. 线程 2 判断 `INSTANCE` 发现不为空，直接返回 `INSTANCE` 对象
5. 线程 2 调用 `INSTANCE` 对象的方法，但此时 `INSTANCE` 对象并未被初始化（异常情况）
6. 线程 1 执行第 21 行字节码对 `INSTANCE` 对象进行初始化

所以，需要使用 `volatile` 禁止指令重排

````java
public class Singleton {
    private static volatile Singleton INSTANCE = null;
    
    private Singleton() {
        
    }
    
    public static Singleton getInstance() {
        //INSTANCE为空，准备进入synchronized代码块
        if (INSTANCE == null) {
            //此时如果有其他线程进入代码块，那么阻塞自己
            synchronized (Singleton.class) {
                //可能有其他线程已经创建实例了，所以需要二次判断
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
}
````

