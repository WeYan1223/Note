#### ThreadLocal

> `ThreadLocal` 是作用范围为线程内部的数据结构，通过它创建的变量只能被当前线程访问

##### 1. 简单使用

创建

````java
private ThreadLocal<Integer> mThreadLocal = new ThreadLocal<>();
````

`set` 方法

````java
mThreadLocal.set(6);
````

`get` 方法

````java
mThreadLocal.get();
````

##### 2. 工作原理

`set` 方法：

```java
/**
 * 1. 获取当前线程
 * 2. 获取当前线程的 ThreadLocalMap 成员变量
 * 3. 若 ThreadLocalMap 为空，则先初始化，在设值
 */
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

`get` 方法：

```java
/**
 * 1. 获取当前线程
 * 2. 获取当前线程的 ThreadLocalMap 成员变量
 * 3. 若 ThreadLocalMap 为空或值为空，返回初始值，否则返回期望的值
 */
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

`getMap(Thread t)` 返回的是 `Thread` 对象中的 `threadLocals` 字段，并且该字段的初始化操作只在 `ThreadLocal` 的 `createMap()` 中完成

````java
public class Thread {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
````

`ThreadLocalMap` 是 `ThreadLocal` 的静态内部类

````java
static class ThreadLocalMap {
    private Entry[] table;
    
    
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    
    //...
}
````

简单来说，`ThreadLocalMap` 是一个特殊的 `Map`，它每个 `Entry` 的 `key (ThreadLocal)` 都是一个弱引用，`Map` 内部使用线性探测法解决哈希冲突

##### 3. 内存泄漏

网上讨论说 `ThreadLocal` 会导致内存泄漏，原因如下：

- 首先 `ThreadLocal` 实例被线程的 `ThreadLocalMap` 实例持有，也可以看成被线程持有
- 如果应用使用了线程池，那么之前的线程实例处理完之后出于复用的目的依然存活
- 所以，`ThreadLocal` 的值被持有，导致内存泄露

虽然这个逻辑是清晰的，但是实际上从 `ThreadLocal` 设计角度来说是不会导致内存泄露的：

首先 `ThreadLocalMap` 中的 `key(ThreadLocal)` 是弱引用，当不存在外部强引用该 `ThreadLocal` 对象时就会被自动回收，引用链如下

````sh
Thread -> ThreaLocalMap -> Entry -> value
````

显然，`ThreadLocal` 对象能被自动回收，但是 `Entry` 对 `value` 仍然具有强引用，此时只能手动清理，`ThreadLocal` 的处理方法是，在 `ThreadLocalMap` 进行`set()`、`get()`、`remove()` 的时候，都会进行清理

当然 `ThreadLocal`并不能 100% 保证不发生内存泄漏，一个良好的习惯是：<u>当不需要这个 `ThreadLocal` 变量时，主动调用 `remove()`</u>

##### 4. 应用场景

> 某些数据是以线程为作用域并且不同线程具有不同的数据副本时

`ThreadLocal` 在 `Android` 中的应用，以 `Looper` 为例

````java
public final class Looper {
    //...
    
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
    
    //...
}
````
