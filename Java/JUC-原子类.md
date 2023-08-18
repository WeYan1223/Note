#### 原子类

> 原子类提供了一些修改数据的方法，这些方法都是原子性的，在多线程情况下可以确保被修改数据的正确性
>
> `JUC` 提供的原子类都是依靠 `volatile` 关键字、`CAS` 算法、`Unsafe` 类来配合实现，期间不涉及加锁操作

`Unsafe` 类提供了一些底层、不安全操作的方法，如访问系统内存资源、自主管理内存资源、`CAS` 操作等，但是普通开发者不能直接用 `Unsafe` 类

这里以原子类 `AtomicInteger` 为例

````java
public class AtomicInteger extends Number {
    private static final Unsafe U = Unsafe.getUnsafe();

    /**
     * 原子性地自增 1 并返回自增后的值
     */
    public final int incrementAndGet() {
        return U.getAndAddInt(this, VALUE, 1) + 1;
    }
}
````

````java
public final class Unsafe {
    /**
     * 原子性地增加指定的值，并返回增加前的值
     */
    public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        // 不断尝试 CAS 直到成功
        do {
            // 确保从主内存中获取变量最新的值
            v = getIntVolatile(o, offset);
        } while (!weakCompareAndSetInt(o, offset, v, v + delta));
        return v;
    }
}
````
