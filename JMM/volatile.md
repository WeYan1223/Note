#### 简述

`volatile` 可以保证共享变量的可见性、禁止指令重排以保证有序性

***

#### 可见性

> `volatile` 可以保证变量的可见性：当一个线程对共享变量进行了修改，那么另外的线程可以立即感知到该修改

`volatile` 用于修饰变量

````java
public static volatile int i = 0;
````

工作内存对 `volatile` 变量的读写添加了约束以保证变量的可见性：

* 工作内存在修改 `volatile` 变量之后，会马上把最新的值刷回主内存
* 工作内存在读取 `volatile` 变量之前，每次都会从主内存读取

注意，`volatile` 只是保证可见性，不能保证原子性，所以多线程对 `volatile` 变量修改仍然是不安全的

````java
public static volatile int count = 0;

public static void main(String[] args) throws InterruptedException {
    Thread threrad1 = new Thread(() -> {
        for (int i = 0; i < 10000; i++) count++; // 编译器会提醒：Non-atomic operation on volatile field 'count'
    });
    Thread threrad2 = new Thread(() -> {
        for (int i = 0; i < 10000; i++) count++;
    });
    threrad1.start();
    threrad2.start();
    threrad1.join();
    threrad2.join();
    System.out.println(count); // 打印结果总是小于 20000
}
````

这里从字节码的角度分析，`count++` 对应的字节码如下

````sh
getstatic # 把 count 放入栈顶
iconst_1  # 把 1 放入栈顶
iadd      # 将栈顶两个数出栈并相加，相加结果放回栈顶
putstatic # 将栈顶的值出栈并放入 count
````

虽然 `volatile` 可以保证 `getstatic` 前的 `count` 值是主内存最新的，也可以保证 `putstatic` 之后的 `count` 值能刷回主内存

但是 `iconst_1`、`iadd` 时，其他线程可能已经把 `count` 值修改了，所以 `putstatic` 之后可能把较小的值放回主内存了 

所以，`volatile` 比较适合单线程写、多线程读的场景

***

#### 有序性

> `volatile` 禁止指令重排保证有序性

下面是一个未完善的双重检测锁实现单例（缺少 `volatile`）

````java
public class Singleton {
    private static Singleton INSTANCE = null;
    
    private Singleton() {}
    
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

`volatile` 根据自身特性可以禁止这种指令重排：

- `volatile` 读之后的操作不会被重排到前面
- `volatile` 写之前的操作不会被重排到后面（所以禁止了21、24行的字节码指令被重排）
- 当第一个操作为 `volatile` 写时，第二个操作为 `volatile` 读时，不能重排序