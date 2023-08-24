#### CopyOnWriteArrayList

> 基于 `JDK 1.8` 分析

`CopyOnWriteArrayList` 是 `JUC` 下的一个线程安全的集合类

核心思想：写时复制，对数组进行写操作时，先创建数组的副本，副本的写操作完之后再将副本赋值回去，这样保证写操作不会影响读操作

使用场景：读多写少的并发场景，能够极大地提高系统的并发性能

缺点：

* 内存占用：每次写操作都需要复制一份原始数据，在数据量比较大的情况下，可能会导致内存资源不足
* 写操作开销：每一次写操作都需要复制一份原始数据，然后再进行修改和替换，所以写操作的开销相对较大，在写入比较频繁的场景下，性能可能会受到影响。
* 数据一致性问题：修改操作不会立即反映到最终结果中，还需要等待复制完成，这可能会导致一定的数据一致性问题

以 `CopyOnWriteArrayList#add(E e)` 为例

````java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 获取原数组
        Object[] elements = getArray();
        int len = elements.length;
        // 创建一个长度为 (len + 1) 的副本，并将原数组的元素复制给副本
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 元素放在新数组末尾
        newElements[len] = e;
        // array 指向副本
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
````

