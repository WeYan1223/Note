#### 线程状态

> `Java` 层面的线程状态

`Java` 层面的线程状态对应的是 `Thread` 的枚举类 `State`

````java
public enum State {
    /**
     * 线程被创建但尚未启动
     * new Thread() 但未调用 Thread#start()
     */
    NEW,

    /**
     * 该状态下有两种含义：可运行、正在运行
     */
    RUNNABLE,

    /**
     * 阻塞状态，获取同步代码块的 Monitor 失败进入阻塞队列
     */
    BLOCKED,

    /**
     * 等待状态，有三种情况进入等待状态
     * 1. 线程进入了同步代码块，但是调用了Object#wait()进入等待队列
     * 2. 子线程调用了 Thread#join()
     * 3. LockSupport.park()，出于线程调度的目的禁用当前线程
     */
    WAITING,

    /**
     * 有时限的等待状态
     * 1. Thread.sleep(long millis)
     * 2. 线程进入了同步代码块，但是调用了Object#wait(long millis)
     * 3. 子线程调用了 Thread#join(long millis)
     * 4. LockSupport.parkNanos()
     * 4. LockSupport.parkUntil()
     */
    TIMED_WAITING,

    /**
     * 终止状态，正常终止或抛出异常
     */
    TERMINATED;
}
````

<img src="https://raw.githubusercontent.com/WeYan1223/Pic/master/Java/Java_线程状态图.webp" alt="Java_线程状态图.webp (1155×771) (raw.githubusercontent.com)" style="zoom: 67%;" /> 

***

#### 终止线程

> `Thread#stop()` 弃用原因

父线程在某个时刻要停止子线程，直接调用子线程的 `stop()`，子线程会马上停止，但是此时程序会处于一个非常不安全的状态：

* 立刻停止线程运行，放弃所有未执行的代码，包括 `catch/finally` 语句，可能会导致某些资源无法正常关闭、回收
* 线程正在执行同步代码块，此时线程被终止掉了，那么同步代码块的原子性将无法得到保证，极大可能造成数据混乱

> 如何优雅地终止线程？

核心思想：<u>线程的终止不能完全由外部决定，而是内外协作式地终止，通常是由外部发出终止信号，内部不定时检查该信号再进行终止</u>

对于需要循环执行的任务，可以把任务放到循环中，设置一个 `volatile boolean` 类型共享变量来控制循环：

````java
public class Main {
    // 退出标志，volatile 保证可见性
    public static volatile boolean exit = false;

    public static void main(String[] args) {
        new Thread(() -> {
            while (!exit) {
                // 需要循环执行的任务...
            }
        }).start();

        // 5 秒后更改退出标志的值
        Thread.sleep(5000);
        exit = true;
    }
}
````

但是该方式有严重的缺点：如果任务中出现了长时间的阻塞（例如生产者-消费者的消息队列），此时线程长时间或者永远都不能检查 `exit` 标志

更优雅地终止线程方式：使用 `Thread` 提供的方法以抛异常 `InterruptedException` 的方式终止

````java
public class Thread {
    /**
     * 如果当前线程处于 WAITING 或 TIME_WAITING 状态，抛出 InterruptionException
     * 否则只是设置当前 interrupt 标志为 true
     */
    public void interrupt() {
        //...
    }
}
````

`Thread#interrupt()` 并不能真正的中断线程，需要被调用的线程自己进行配合才行：

*  在正常运行任务时，不定时检查本线程的中断标志位，如果被设置了中断标志就自行停止线程
*  在调用阻塞方法（`Object#wait`、`Thread#join()`、`Thread.sleep()` ···）时正确处理 `InterruptedException` 异常

***

#### 线程控制

> `Java` 提供了一些便捷的方法用于控制线程的执行

##### 1. joinzhuangt

`Thread#join()` 用于等待一个线程的完成，当线程 `A` 调用另一个线程 `B` 的 `join()` 方法时，线程 `A` 将处于 `WAITING`  状态直到线程 `B` 执行完毕

````java
public static void main(String[] args) throws Exception {
    Thread thread = new Thread(() -> {
        Thread.sleep(2000);
        System.out.println("线程 A 结束");
    });
    thread.start();
    thread.join();
    System.out.println("主线程结束");
}

// 打印结果：
// 线程 A 结束
// 主线程结束
````

##### 2. yield

`Thread#yield()`，当前线程主动放弃对 `CPU` 的占用，重新参与抢占 `CPU` 执行权

##### 3. sleep

`Thread.sleep(int millis)`，让当前线程处于 `TIME_WAITING` 状态 `millis` 毫秒

##### 4. 优先级

优先级较高的线程有更高的机会成功抢占 `CPU` 执行权

调用 `Thread#setPriority(int newPriority)`设置线程的优先级，参数范围 1 ~ 10

