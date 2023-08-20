#### AQS

> `AQS` 全称 `AbstractQueuedSynchronizer`

`AQS` 是一个强大且灵活的框架，基于它可以实现各种锁、同步器，`JUC` 中的 `ReentrantLock`、`Semaphore` 等并发类均是基于 `AQS` 实现的

核心思想：

* 如果共享资源空闲，那么就将请求资源的线程设置为有效的线程，并将共享资源设置为锁定状态
* 如果共享资源被占用，就需要一套 "等待 / 唤醒" 机制来保证锁分配，将获取不到锁的线程加入到队列中

![JUC_AQS模型.webp (681×181) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/Java/JUC_AQS模型.webp) 

这里从以下几个方面来理解 `AQS`

> 线程如何获取资源（锁）？

`AQS` 内部通过一个 `volatile` 变量 `state` 来标记资源数量

````java
private volatile int state;

protected final int getState() {
    return state;
}

/**
 * 原子性地修改 state
 */
protected final boolean compareAndSetState(int expect, int update) {
    return STATE.compareAndSet(this, expect, update);
}
````

`state` 的含义以及初始值不同，可以实现不同的锁

* 独占锁：`ReentrantLock` 中，`state` 表示当前锁被持有的次数，`state` 为 0 表示资源空闲，同一个线程获取锁时 `state++` 、释放锁时 `state--`
* 信号量：`Semaphore` 中，`state` 表示当前可用资源的数量，以实现类似操作系统中的信号量功能

> 获取不到资源（锁）的线程该如何处理？（这里只分析独占模式）

这段逻辑从 `AQS` 的 `acquire(int)` 开始

````java
/**
 * 以独占模式获取锁，忽略中断（即线程等待锁的期间不会响应中断信号）
 * 通过 tryAcquire() 尝试获取锁：
 * 		获取成功，直接返回返回
 *  	获取失败，线程插入等待队列的队尾
 */
public final void acquire(int arg) {
    // 这里涉及到三个方法
    // 1. tryAcquire()，模版方法由子类实现，描述线程如何获取资源（锁）
    // 2. addWaiter(mode)，将线程封装成 Node 插入等待队列的队尾（这个插入操作是线程安全的，内部采用 CAS 算法）
    // 3. acquireQueued(node, arg)，下面着重分析
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {
        selfInterrupt();
    }
}
````

等待队列的模型大致如下

![JUC_等待队列模型.webp (765×205) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/Java/JUC_等待队列模型.webp) 

`acquireQueued(node, arg)` 比较核心，线程等待的逻辑和线程唤醒后的逻辑都在这里

````java
/**
 * 以独占非中断模式获取已在队列中的线程。由条件等待方法和获取方法使用。
 */
final boolean acquireQueued(final Node node, int arg) {
    boolean interrupted = false;
    try {
        for (;;) {
            // 获取前驱节点
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                // 这里有两个作用：
                // 1. 进入等待队列这段时间，可能锁已经被释放了，所以这里再次尝试获取锁，这样就不用等待了
                // 2. 被唤醒后尝试获取锁
                setHead(node);
                p.next = null;
                return interrupted;
            }
            // 判断线程是否需要进入等待状态
            if (shouldParkAfterFailedAcquire(p, node)) {
                // 使线程进入等待状态
                interrupted |= parkAndCheckInterrupt();
            }
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        if (interrupted)
            selfInterrupt();
        throw t;
    }
}
````

流程图如下：

<img src="https://raw.githubusercontent.com/WeYan1223/Pic/master/Java/JUC_acquireQueued流程.webp" alt="JUC_acquireQueued流程.webp (1000×1400) (raw.githubusercontent.com)" style="zoom: 67%;" /> 

点到为止

> 实现锁应该考虑的问题？

1. 如何获取资源(锁)
2. 获取不到资源的线程如何处理
3. 如何释放资源
4. 资源释放后如何让其他线程获取资源

> 由此可以得出实现一把锁，应该具备哪些逻辑？

- 锁的标识，需要有个标识或者状态来表示锁是否已经被占用
- 线程抢锁的逻辑，多个线程如何抢锁，如何才算抢到锁，已经抢到锁的线程再次抢锁如何处理等等
- 线程挂起的逻辑，线程如果抢到锁自然顺利往下运行了，而那些没有抢到锁的线程怎么处理呢？如果一直处于活跃状态，cpu肯定是吃不消，那就需要挂起。具体又如何挂起呢
- 线程存储机制，没有抢到锁的线程就挂起了，而且被挂起的线程可能有很多个，这些线程总要放在某个地方保存起来等待唤醒，然而这么多被挂起的线程，要唤醒哪一个呢？这就需要一套保存机制来支撑唤醒逻辑
- 线程释放锁的逻辑，线程在执行完后就要释放锁，跟抢锁逻辑是对应的，其实也是操作锁标识
- 线程唤醒的逻辑，锁释放后，就要去唤醒被阻塞的线程，这就要考虑唤醒谁，如何唤醒，唤醒后的线程做什么事情