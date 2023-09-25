#### 简述

> `Binder` 支撑起了 `Android` 系统进程间的通信，而 `Handler` 消息机制支撑起的则是进程内线程间的通信
>
> `Android` 应用程序的运行都是依靠 `Handler` 消息机制的驱动，例如触摸事件的分发、`View` 的绘制流程、屏幕的刷新机制以及 `Activity` 生命周期等等

消息机制，也称为 `Handler` 机制，主要有两个作用

* 将任务切换到某个指定线程中执行
* 将任务延迟至某个时间执行



**Q：**在 `Android` 中，为什么访问 `UI` 的操作只能在主线程中进行？

首先，`Android` 的 `UI` 控件不是线程安全的，意味着并发访问可能会导致 `UI` 控件处于不可预期的状态

其次，为什么系统不对 `UI` 控件的访问加上锁机制呢？缺点有两个

* `UI` 访问的逻辑会变复杂

* 降低 `UI` 访问效率

`Handler`  的主要作用是将一个任务切换到某个指定线程中执行，在日常开发中，通常在子线程中使用 `Handler` 切换到主线程访问 `UI`

***

#### 生产者-消费者

> `Handler` 机制基于生产者-消费者模型

生产者-消费者问题又称有限缓冲问题，该问题描述了生产者线程和消费者线程在共享固定大小缓冲区时会发生的问题：

* 生产者：生成数据放到缓冲区中
* 消费者：从缓冲区读取数据

问题的关键在于保证：生产者不会在缓冲区满时加入数据，消费者也不会在缓冲区中空时读取数据

![Handler-生产者消费者模型.webp (384×296) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/Android/Handler-生产者消费者模型.webp) 

生产者和消费者处于两个不同的线程，但是他们共用了同一个队列：

* 生产者在完成数据生产后通过 `notify` 方法唤醒消费者线程读取数据，当队列满时，生产者线程调用 `wait` 方法来阻塞自身
* 消费者线程在被唤醒后则会从队列中取出数据，并通过 `notify` 方法唤醒生产者线程继续生产数据，当队列为空时，消费者线程调用 `wait` 方法阻塞自身

生产者-消费者模型不仅解决了多个线程共享内存的有限缓冲问题，还解决了线程间通信的问题：<u>由于共用同一个缓冲队列，生产者产生的数据从生产者线程传递给了消费者线程，对数据而言就实现了线程的切换</u>

***

#### 总体框架

##### 1. 缓冲区

`MessageQueue`，消息队列用来存放生产者产生的消息，需要对外提供入队、出队接口

##### 2. 生产者

`Handler`，作为 `MessageQueue` 的包装类，将各种消息封装成统一的 `Message` 并插入到 `MessageQueue` 中

##### 3. 消费者

`Looper`，不断的从 `MessageQueue` 中取出消息

***

#### Message

`Message` 是消息队列的基本单位，由于应用程序运行会有大量 `Message` 插入到 `MessageQueue` 中，所以这里使用到池化技术来解决内存抖动问题，池化技术的核心方法有两个：

````java
public final class Message implements Parcelable {
    /**
     * 从对象池取出 Message
     */
    public static Message obtain() {}
    
    /**
     * 回收用完的 Message
     */
    public void recycle() {}
}
````

`Message` 的核心成员变量：

````java
public final class Message implements Parcelable {
    /**
     * Message 的唯一标识
     */
    public int what;
    
    /**
     * 如果 Message 需要携带少量 int 类型的数据，可以使用arg1
     */
    public int arg1;
    
    /**
     * 如果 Message 需要携带少量 int 类型的数据，可以使用arg2
     */
    public int arg2;
    
    /**
     * 该 Message 应该什么时候被处理，是绝对时间，基于{SystemClock#uptimeMillis()}
     */
    public long when;
    
    /**
     * 携带大量复杂数据时，使用 data
     */
    Bundle data;
    
    /**
     * 指定该 Message 的处理者，target的值在使用 Haldler 将该 Message 插入 MessageQueue 前设置
     */
    Handler target;
    
    /**
     * 提供给 Handler 进行封装的 Runnable 对象
     */
    Runnable callback;
    
    /**
     * 用于 MessageQueue 实现单链表
     */
    Message next;
}
````

***

#### MessageQueue

> 消息队列，说是队列，但内部实现是单链表，主要关注入队与出队
>

##### 1. 入队

```java
public final class MessageQueue {
    /**
     * 链表表头
     */
    Message mMessages;

    boolean enqueueMessage(Message msg, long when) {
        synchronized (this) {
            msg.when = when;
            Message p = mMessages;
            if (p == null || when == 0 || when < p.when) {
                // 这是将消息插入链表表头的逻辑
                // 1.链表为空
                // 2.非延迟消息
                // 3.是延迟消息，但预期处理时间比表头消息得预期处理时间小
                msg.next = p;
                mMessages = msg;
            } else {
                // 这种情况下说明要插入的消息是延迟消息，需要遍历链表找到合适的插入位置
                Message prev;
                for ( ; ; ) {
                    prev = p;
                    p = p.next;
                    // 按 Message 的 when 字段升序排序
                    if (p == null || when < p.when) {
                        break;
                    }
                }
                msg.next = p;
                prev.next = msg;
            }
            if (needWake) {
                // 唤醒线程
                nativeWake(mPtr);
            }
        }
        return true;
    }
}
```

上面省略了大部分代码，留下的是 `Message` 插入单链表的逻辑，插入位置是根据消息 `delay` 的时间决定的，`delay` 的时间越长，插入队列时就越靠后，即越晚执行，最后根据是否要唤醒消费者线程来调用 `nativeWake()`，该方法可以理解为生产者-消费者模型中通过 `notify` 唤醒线程的操作

##### 2. 出队

主要关注 `MessageQueue` 的 `next()` 

```java
public final class MessageQueue {
    /**
     * 链表表头
     */
    Message mMessages;

    Message next() {
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
        int nextPollTimeoutMillis = 0;
        for ( ; ; ) { 
            // 阻塞线程，方法的第二个参数表示阻塞的毫秒时间：
            // 如果是 0 表示不阻塞
            // 如果是正数，则阻塞该值的毫秒时长
            // 如果是负数，表示一直阻塞
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                // 省略异步消息得处理

                final long now = SystemClock.uptimeMillis();
                Message msg = mMessages;
                if (msg != null) {
                    if (now < msg.when) {
                        // 还没到当前消息得预期处理时间，设置线程得阻塞时间为：预期时间 - 当前时间
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 出队
                        mMessages = msg.next;
                        msg.next = null;
                        return msg;
                    }
                } else {
                    // 消息队列为空，一直阻塞直到有新消息入队
                    nextPollTimeoutMillis = -1;
                }
                if (mQuitting) {
                    dispose();
                    return null;
                }
            }
        }
    }
}
```

上面省略了大部分代码，暂时忽略了异步消息以及 `IdleHandler` 的处理逻辑，留下的是 `Message` 出队的主要逻辑

当调用了 `Looper#quit()` 或 `Looper#quitSafely()` 时，`MessageQueue` 的 `next()` 会返回 `null`

***

#### Looper

> `Looper` 在生产者-消费者模型中扮演消费者的角色，它会以死循环的形式去 `MessageQueue` 查找是否有新消息，如果有的话就处理消息，否则就一直等待着

`Looper` 只有一个私有的构造方法，在这里对 `MessageQueue` 进行初始化

````java
public final class Looper {
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
}
````

而调用该私有构造方法的地方只有 `Looper#prepare()`

````java
public final class Looper {
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        // 一个线程只能存在一个 Looper 对象
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
}
````

`quitAllowed` 的作用是是否允许退出消息循环，主线程是默认不允许退出的，如果调用主线程 `Looper#quit()` 会抛出以下异常

````java
throw new IllegalStateException("Main thread not allowed to quit.")
````

由于主线程比较特殊，`Looper` 专门提供以下方法给开发者调用

````java
public final class Looper {
    private static Looper sMainLooper;

    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }
}
````

调用 `loop()` 后，真正开启 `Handler` 消息机制

````java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    //...
    for ( ; ; ) {
        if (!loopOnce(me, ident, thresholdOverride)) {
            return;
        }
    }
}

private static boolean loopOnce(final Looper me, final long ident, final int thresholdOverride) {
    // 从 MessageQueue 获取消息
    Message msg = me.mQueue.next();
    if (msg == null) {
        // 前面已经分析了 MessageQueue 返回 null 的情况
        return false;
    }
    // 将消息分派给指定的 Handler 处理
    msg.target.dispatchMessage(msg);
    return true;
}
````

`loop()` 存在死循环，不断调用 `MessageQueue#next()` 尝试获取 `Message`

* 如果 `Message` 为空，此时 `Looper` 已经调用 `quit()` 或 `quitSafely()` 退出，则退出循环
* 如果 `Message` 不为空，则将该 `Message` 交由 `msg.target` 处理

***

#### Handler

> `Handler` 在生产者-消费者模型中扮演生产者（发送消息）的角色
>
> 除此之外，在消息机制中 `Handler` 还得承担处理消息的职责

##### 1. 发送消息

`Handler` 的两个常用的构造方法

````java
public class Handler {
    final Looper mLooper;
    final MessageQueue mQueue;
    final Callback mCallback;
    final boolean mAsynchronous;
    
    public Handler(@NonNull Looper looper) {
        this(looper, null, false);
    }

    public Handler(@NonNull Looper looper, @Nullable Callback callback) {
        this(looper, callback, false);
    }
    
    public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
        // 一个 Handler 只能对应一个 Looper 和一个 MessageQueue
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
}
````

其中，`Callback` 是只有一个方法声明的接口

````java
public interface Callback {
    boolean handleMessage(@NonNull Message msg);
}
````

消息的发送可以通过 `Handler` 的一系列 `sendXxx()` 与 `postXxx()` 来实现

````java
public class Handler {
    // 发送非延迟消息
    public final boolean sendMessage(@NonNull Message msg) {}

    // 发送延迟消息，指定延迟时间（相对时间）
    public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {}

    // 发送延迟消息，指定延迟时间（绝对时间）
    public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {}
}
````

`postXxx()` 可以看作是对 `sendXxx()` 的封装

````java
public class Handler {
    public final boolean post(@NonNull Runnable r) {
        return  sendMessageDelayed(getPostMessage(r), 0);
    }

    public final boolean postAtTime(@NonNull Runnable r, long uptimeMillis) {
        return sendMessageAtTime(getPostMessage(r), uptimeMillis);
    }

    public final boolean postDelayed(Runnable r, long delayMillis) {
        return sendMessageDelayed(getPostMessage(r), delayMillis);
    }
    
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
}
````

以上仅列出常用的几个方法，不论是那个最终都会进入 `Handler#enqueueMessage()` 完成入队操作

````java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg, long uptimeMillis) {
    // 在这里把 Message 的处理者设为当前 Handler
    msg.target = this;
    //真正入队
    return queue.enqueueMessage(msg, uptimeMillis);
}
````

##### 2. 处理消息

当 `Looper` 从 `MessageQueue` 中获取到 `Message` 后，将调用 `msg.target.dispatchMessage(msg)` 处理消息

````java
public class Handler {
    public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            // 一般是通过 postXxx() 发送的消息
            handleCallback(msg);
        } else {
            // mCallback 是实例化 Handler 时传入的 Callback 参数
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    // 在 Callback 处理消息成功，直接返回
                    return;
                }
            }
            // 该方法默认空实现，需要子类重写
            handleMessage(msg);
        }
    }

    private static void handleCallback(Message message) {
        message.callback.run();
    }
}
````

所以，下面的两种写法是有区别的

````java
//写法一，带上 Callback 参数
Handler handler = new Handler(Looper.getMainLooper(), new Handler.Callback() {
    @Override
    public boolean handleMessage(@NonNull Message msg) {
        //...
    }
});

//写法二，重写 handleMessage()
Handler handler = new Handler(Looper.getMainLooper()) {
    @Override
    public void handleMessage(@NonNull Message msg) {
        //...
    }
};
````

***

#### 总体流程

1. 为当前线程设置 `Looper`

   ````java
   Looper.prepare();
   ````

   如果当前线程已存在 `Looper`，会抛出异常

   ````java
   new RuntimeException("Only one Looper may be created per thread");
   ````

   创建 `Looper` 的同时也会创建 `MessageQueue` 对象

2. 开始循环（只有调用 `Looper.loop()` 之后，才能开始处理消息）

   ````java
   Looper.loop();
   ````

3. 创建用来发送消息与处理消息的 `Handler` 对象

   ````java
   private Handler mHandler = new Handler(Looper.myLooper()) {
       @Override
       public void handleMessage(@NonNull Message msg) {
           switch (msg.what) {
               case 0:
                   //...
                   break;
               case 1:
                   //...
                   break;
               default:
                   break;
           }
       }
   };
   ````

4. 在其他线程使用该 `mHandler` 发送消息至 `mHandler` 的 `Looper` 所在线程的处理

   ````java
   /*其他线程*/ {
       Message message = Message.obtain();
       message.what = 0;
       //...
       mHandler.sendMessage(message);
   }
   ````

5. 在不需要处理消息或退出当前 `Activity` 的情况下一定要退出消息循环（有关内存泄漏会在下面分析）

   ````java
   //直接退出
   Looper.myLooper().quit();
   
   //安全退出（等待消息全部处理完）
   Looper.myLooper().quitSafely();
   ````

***

#### 异步消息

`Message` 为开发者提供了一个 `setAsynchronous()` 方法为消息设置一个异步标记，称为异步消息，异步消息拥有最高的执行优先级，但是仅仅将其设为异步消息并没有什么作用，它需要配合同步屏障来执行

<u>同步屏障：也是一个 `Message`，只不过它的 `tager` 为 `null`（通俗来讲就是屏蔽掉同步消息的屏障）</u>

实际上，在上面的分析中： `MessageQueue#next()` 中，省略了以下 `START` 至 `END` 的异步消息处理逻辑

````java
public final class MessageQueue {
    /**
     * 链表表头
     */
    Message mMessages;

    Message next() {
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
        int nextPollTimeoutMillis = 0;
        for ( ; ; ) { 
            // 阻塞线程，方法的第二个参数表示阻塞的毫秒时间：
            // 如果是 0 表示不阻塞
            // 如果是正数，则阻塞该值的毫秒时长
            // 如果是负数，表示一直阻塞
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                final long now = SystemClock.uptimeMillis();
                //==========START==========
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // 存在同步屏障，需要寻找下一条异步消息
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                //===========END===========

                if (msg != null) {
                    if (now < msg.when) {
                        // 还没到当前消息得预期处理时间，设置线程得阻塞时间为：预期时间 - 当前时间
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 出队
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        return msg;
                    }
                } else {
                    // 消息队列为空，一直阻塞直到有新消息入队
                    nextPollTimeoutMillis = -1;
                }
                if (mQuitting) {
                    dispose();
                    return null;
                }
            }
        }
    }
}
````

所以，同步屏障就是用来屏蔽掉同步屏障之后的所有的同步消息，只处理异步消息，换句话说，同步屏障使 `Handler` 增加了优先级机制：异步消息的优先级要高于同步消息

> 如何发送同步屏障？

在前面的分析中，使用 `Handler#postXxx()`、`Handler#sendXxx()` 发送的消息最终都会走到 `Handler#enqueueMessage()`

````java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg, long uptimeMillis) {
    // 在这之后的 target 肯定不为 null
    msg.target = this;
    //...
}
````

所以这里需要使用 `MessageQueue` 提供的方法来设置同步屏障

````java
public final class MessageQueue {
    @UnsupportedAppUsage
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) {
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
}
````

返回值 `token` 用于之后移除同步屏障（如果不移除，同步消息将被一直屏蔽）

````java
public void removeSyncBarrier(int token) {
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                                            + " barrier token has not been posted or has already been removed.");
        }
        if (prev != null) {
            prev.next = p.next;
        } else {
            mMessages = p.next;
        }
    }
}
````

但是，`MessageQueue#postSyncBarrier()` 并不允许开发者调用(〒︿〒)

**实际应用：**

当调用 `View.resquestLayout()`，会一直往上调用直到 `ViewRootImpl.requestLayout()`，在这之后并不会马上进行绘制任务，而是会给主线程发一个同步屏障，当下一次屏幕刷新信号到来才发送异步消息（消息内容为执行绘制任务）到主线程，来执行绘制任务，这样可以保证在屏幕刷新信号到来时，绘制任务可以被及时执行，不会造成界面卡顿

***

#### 内存泄漏

> 下面是 `Handler` 非常常见的写法（匿名内部类）
>
> ````java
> public class MainActivity extends AppCompatActivity {
>        private Handler mHandler = new Handler(Looper.getMainLooper()) {
>            @Override
>            public void handleMessage(@NonNull Message msg) {
>                //...
>            }
>        };
> }
> ````
>
> 如果现在发送了一条延时消息，在消息处理前关闭了当前 `Activity`，那么此时就发生了内存泄漏

##### 1. 原因

上面分析了无论是使用 `postXxx()` 使用 `sendXxx()` 发送消息，最终都会进入下面这个方法

````java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg, long uptimeMillis) {
    msg.target = this;
    //...
}
````

这里的 `msg.target` 就是发送消息的 `Handler` 对象

所以此时 `Message` 持有 `Handler` 对象的引用

同时，由于匿名内部类持有外部类的引用，所有 `Handler` 又会持有 `Activity` 的引用

所以 `Activity` 不能被及时销毁，造成内存泄漏

引用链大概是这样的：主线程 => `ThreadLocal` => `Looper` => `MessageQueue` => `Message` => `Handler` => `Activity`

##### 2. 解决

**2.1 静态内部类 + 弱引用**

````java
public class MainActivity extends AppCompatActivity {
    private final Handler mHandler = new H(Looper.getMainLooper(), this);

    private static class H extends Handler {
        //弱引用，垃圾回收时被回收
        WeakReference<Activity> mActivity;

        public H(Looper looper, Activity activity) {
            super(looper);
            mActivity = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(@NonNull Message msg) {
            //...
        }
    }
}
````

**2.2 Activity销毁时清空MessageQueue**

````java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onDestory() {
        super.onDestroy();
        mHandler.removeCallbacksAndMessages(null);
    }
}
````

***

#### 参考文章

* [反思 Android 消息机制的设计与实现 - 掘金 (juejin.cn)](https://juejin.cn/post/7110625878320775204)
* [源码茶舍之没有epoll就没有Handler - 掘金 (juejin.cn)](https://juejin.cn/post/6896495861954510861)
* [ActivityThread的理解和APP的启动过程_小河同学的博客-CSDN博客](https://blog.csdn.net/hzwailll/article/details/85339714)