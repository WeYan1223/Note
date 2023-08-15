#### 简述

`synchronized` 关键字可以保证<u>任意时刻只能有一个线程可以执行某段代码</u>

````java
/**
 * 修饰代码块
 */
synchronized (对象) {
    //...
}

/**
 * 修饰成员方法
 */
public synchronized void method1() {
    //...
} 

/**
 * 修饰静态方法
 */
public static synchronized void method2() {
    //...
}
````

* `synchronized` 修饰代码块时，编译后会在代码块的前后生成 `monitor_enter` 和 `monitor_exit` 字节码指令来实现同步
* `synchronized` 修饰方法时，编译后会在方法名上生成一个 `ACC_SYNCHRONIZED` 标识来实现同步

不论使用哪种方式修饰，本质上都是获取指定对象的 `Monitor`，只有获取到 `Monitor` 的线程，才可以执行方法或代码块，其他获取失败的线程会被阻塞

***

#### 原子性保证

> `synchronized` 可以保证操作的原子性：一个操作或者多个操作，要么全部执行并且不受到任何因素的干扰而中断，要么都不执行

同一时间只能有一个线程执行同步代码块，那么这个操作就是不能被其它线程打断的，所以 `synchronized` 可以保证操作的原子性

***

#### 可见性保证

> `synchronized` 可以保证变量的可见性：当一个线程对共享变量进行了修改，那么另外的线程可以立即感知到该修改

释放锁会将当前线程的工作内存中已写入的数据刷回到主内存中，而获取锁会强制重新加载主内存中的数据到工作内存

所以，虽然锁操作只对同步方法和同步代码块这一块起到作用，但是影响的却是线程执行使用的所有共享变量

如下代码不会停止：

```java
static boolean flag = true;

public static void main(String[] args) throws InterruptedException {
    new Thread(() -> {
        int x = 0;
        while (flag) {
            x++;
        }
    }).start();

    Thread.sleep(100);
    flag = false;
}
```

如下代码会停止（`println()` 内部使用到了 `synchronized`）：

````java
static boolean flag = true;

public static void main(String[] args) throws InterruptedException {
    new Thread(() -> {
        int x = 0;
        while (flag) {
            System.out.println(x);
            x++;
        }
    }).start();

    Thread.sleep(100);
    flag = false;
}
````

***

#### 底层原理

`JVM` 中，对象在内存中的布局分为三个部分

![synchronized_对象结构.webp (252×329) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/JMM/synchronized_对象结构.webp) 

* 对象头：下面会重点分析
* 实例数据：存放对象的成员变量的状态和属性（包括父类的）
* 对齐填充：`JVM` 要求对象起始地址必须是 `8 Byte` 的整数倍，所以这里仅仅是为了字节对齐

> 对象头（这里之分析普通对象，如果是数组对象，对象头还会包含数组长度的字段）

![synchronized_对象头结构.webp (254×119) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/JMM/synchronized_对象头结构.webp) 

* `Mark Word`：下面会重点分析
* `Klass Word`：指向该对象对应的 `Class` 对象

在 32 位的虚拟机下，`Mark Word` 占用 32 位，但是这 32 位空间被压榨到了极致：对象在不同的状态下，`Mark Word` 存储的值的含义完全不同

![synchronized_Mark Word结构.webp (962×288) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/JMM/synchronized_Mark Word结构.webp)  

这里四种锁的状态都是针对 `synchronized` 的，级别从低到高依次是：无锁、偏向锁、轻量级锁和重量级锁（锁状态只能升级不能降级）

##### 1. 偏向锁

> 偏向锁在 `JDK 6` 及以后是默认启用的，在 `JDK 15` 及以后默认关闭并标为过时，可以通过 `JVM` 参数：`UseBiasedLocking` 控制

在大多数情况下，锁总是由同一线程多次获得，不存在多线程竞争，所以出现了偏向锁，使得只有一个线程执行同步代码块时能够提高性能

##### 2. 轻量级锁

> 多个线程在不同时段获取同一把锁，这时也不存在多线程竞争的情况，所以采用轻量级锁来避免线程的阻塞与唤醒

线程访问同步代码块时，发现对象处于偏向锁状态（说明此时可能存在多线程竞争），那么升级为轻量级锁状态，该线程会通过自旋的形式尝试获取锁

如果自旋超过一定的次数依然没有获取到锁，那么轻量级锁会升级成重量级锁

##### 3. 重量级锁

> 在重量级锁状态下，`Mark Word` 会关联一个 `Monitor` 对象

![synchronized_Monitor结构.webp (250×329) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/JMM/synchronized_Monitor结构.webp) 

* `WaitSet`：获取锁之后进入 `Waiting` 状态的线程
* `EntryList`：存放因竞争锁失败而阻塞的线程
* `Owner`：当前持有 `Monitor` 的线程

多线程竞争锁时，只能有一个线程获取到锁，其他线程处于阻塞状态并且被放入 `EntryList` 中等待 `Owner` 释放锁后被唤醒，被唤醒后再次参与锁竞争

当线程获取锁进入同步代码块后，调用 `wait()` 会使当前线程释放持有的 `Monitor`，处于 `Waiting` 状态并进入 `WaitSet` 等待 `notify()`

`synchronized` 是可重入的，由 `Monitor` 的 `counter` 变量保证

***

#### 锁的优化

##### 1. 锁消除

`JVM` 即时编译再运行时，对于代码上要求同步但实际上不可能存在共享变量竞争的锁进行消除，判定依据来源于**逃逸分析**

##### 2. 锁粗化

原则上，加同步锁时应尽可能将作用范围缩小

但如果存在一连串的对同一个对象反复加锁和解锁，这样即使没有线程竞争，频繁的互斥同步操作也会导致不必要的性能消耗

`JVM` 会检测这种操作，将加锁同步的范围扩展