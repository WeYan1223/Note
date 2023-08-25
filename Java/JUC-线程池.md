#### 线程池

> 基于 `JDK 1.8` 分析

线程池是 `Java` 中的一种池化思想的实现，优点如下：

* 减少资源损耗：重复利用已创建的线程，降低线程创建和销毁造成的损耗
* 提高响应速度：任务到达时，无需等待线程创建即可立即执行
* 提高线程的可管理性：线程池可以对线程进行统一的分配、调优和监控

核心实现类是 `ThreadPoolExecutor`，继承体系如下：

````sh
Executor
	↑
	ExecutorService
		↑
		AbstractExecutorService
				↑
				ThreadPoolExecutor
````

运行机制如下：

<img src="https://raw.githubusercontent.com/WeYan1223/Pic/master/Java/JUC_线程池运行机制.webp" alt="JUC_线程池运行机制.webp (1085×476) (raw.githubusercontent.com)" style="zoom:80%;" /> 

> 线程池生命周期？

`ThreadPoolExecutor` 内部内部使用<u>一个成员变量维护线程池状态和线程数量</u>

源码中经常要同时判断线程池状态和线程数量，这样可以避免为了维护两者的一致，而占用锁资源

````java
/**
 * 高 3 位存储线程池状态
 * 低 29 位存储线程数量
 * 默认：RUNNING 状态，0 线程
 */
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
````

线程池一共五种状态

````java
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
````

|      状态      |                             描述                             |
| :------------: | :----------------------------------------------------------: |
|  **RUNNING**   |     运行状态：能接收新提交的任务，也能处理阻塞队列的任务     |
|  **SHUTDOWN**  |  关闭状态：不再接收新提交的任务，但能继续处理阻塞队列的任务  |
|    **STOP**    | 停止状态：不接收新任务，也不处理阻塞队列的任务，中断正在处理任务的线程 |
|  **TIDYING**   |            干净状态：所有任务已终止，线程数量为 0            |
| **TERMINATED** |                           结束状态                           |

生命周期如下：

<img src="https://raw.githubusercontent.com/WeYan1223/Pic/master/Java/JUC_线程池生命周期.webp" alt="JUC_线程池生命周期.webp (880×219) (raw.githubusercontent.com)"  /> 



> 线程池如何管理线程？

对于线程池内线程数量，由两个成员变量描述：

```java
/**
 * 核心线程数
 * 当任务提交到线程池时，如果线程池中的线程数量没有超过 corePoolSize
 * 那么就会创建一个新的线程来立刻执行任务，否则将任务添加到任务队列等待被调度
 */
private volatile int corePoolSize;

/**
  * 最大线程数
  * 若任务队列入队失败（任务队列满），如果线程池中的线程数量没有超过 maximumPoolSize
  * 那么就会创建一个新的线程来执行任务，否则执行拒绝策略
  */
private volatile int maximumPoolSize;
```

线程池将线程封装成 `Worker` 类

````java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    /**
     * Worker 持有的线程
     */
    final Thread thread;

    /**
     * 初始化的任务，可以为 null
     */
    Runnable firstTask;	

    Worker(Runnable firstTask) {
        setState(-1);
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }
}
````

`Worker` 工作的入口是 `ThreadPoolExecutor#runWorker()`，里面的逻辑主要如下：

* 死循环不断地通过 `getTask()` 方法获取任务执行
* 如果线程池正在停止，那么要保证当前线程是中断状态，否则要保证当前线程不是中断状态
* 如果 `getTask` 结果为 `null` 则跳出循环，执行 `processWorkerExit()` 方法，销毁线程

> 线程池如何调度任务？

通常通过 `ThreadPoolExecutor#execute(Runnable command)` 来提交任务，流程如下：

![JUC_线程池任务提交流程.webp (584×525) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/Java/JUC_线程池任务提交流程.webp) 

任务拒绝目的在于保护线程池，线程池具有默认的拒绝策略

````java
public static class AbortPolicy implements RejectedExecutionHandler {
    /**
     * 不接收新任务，直接抛出异常
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() + " rejected from " + e.toString());
    }
}
````

线程需要不断从阻塞队列读取任务来执行，这部分逻辑在 `ThreadPoolExecutor#getTask()` 实现，若 `getTask()` 返回 `null`，说明线程需要被销毁了，线程池会移除对该线程的引用

````java
private Runnable getTask() {
    boolean timedOut = false;

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 线程池已停止运行，返回 null
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);
        
        // 是否需要超出存活时间销毁线程
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 线程数量过多或超出存活时间，返回 null
        if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c)) {
                return null;
            }
            continue;
        }

        try {
            Runnable r = timed ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : workQueue.take();
            if (r != null){
                return r;
            }
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
````

> 线程池的核心成员变量

````java
public class ThreadPoolExecutor {
    /**
     * 线程池的核心线程数
     * 当任务提交到线程池时，如果线程池中的线程数量没有超过corePoolSize，那么就会
     * 创建一个新的线程来执行任务，否则将任务添加到任务队列
     */
    private volatile int corePoolSize;
    
    /**
     * 线程池中允许存在的最大线程数
     * 当任务队列满了之后，再有新任务进入线程池，如果线程池中的线程数量没有超过maximumPoolSize，那么就会
     * 创建一个新的线程来执行任务，否则执行拒绝策略
     */
    private volatile int maximumPoolSize;
    
    /**
     * 线程存活时间（时间单位由构造器的TimeUnit参数决定）
     * 当线程池中的线程数超过corePoolSize时，对于多出来的线程，如果没有任务执行
     * 就代表它们是空闲的，此时它们的就存活时间为keepAliveTime
     */
    private volatile long keepAliveTime;
    
    /**
     * 任务队列，类型为阻塞队列
     * 在队列为空时，获取元素的线程会等待队列变为非空；当队列满时，存储元素的线程会等待队列可用
     * 使用不同的队列可以实现不一样的任务存取策略
     * 常见的有ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQueue、PriorityBlockingQueue
     */
    private final BlockingQueue<Runnable> workQueue;
    
    /**
     * 线程工厂，用于创建线程
     * 通过线程工厂，可以方便的为每一个创建的线程设置具有业务含义的名称
     */
    private volatile ThreadFactory threadFactory;
    
    /**
     * 拒绝策略：当任务队列已满，线程数量达到maximumPoolSize后，线程池不会再接收新任务
     * 此时需要使用拒绝策略来决定最终怎么处理该任务
     */
    private volatile RejectedExecutionHandler handler;
}
````

***

#### 参考文章

* [Java线程池实现原理及其在美团业务中的实践 - 美团技术团队 (meituan.com)](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)

