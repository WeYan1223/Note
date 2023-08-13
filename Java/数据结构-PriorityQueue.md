#### PriorityQueue

> 优先队列的实现基于优先级堆，它能够在 `O(log n)` 的时间复杂度内实现元素的插入和删除操作，并且自动维护队列中元素的优先级顺序

通俗来说，`PriorityQueue` 就是一个队列，但是它不是先进先出的，而是按照元素优先级进行排序的

* 入队时，根据元素的优先级将其插入到合适的位置
* 出队时，将优先级最高的元素出队（第一个元素），同时维护队列中元素的优先级顺序

````java
public PriorityQueue(int initialCapacity, Comparator<? super E> comparator) {
    if (initialCapacity < 1) throw new IllegalArgumentException();
    this.queue = new Object[initialCapacity];
    this.comparator = comparator;
}
````

如果没有设置 `Comparetor`，那么默认是升序排序，那么每次出队的元素（第一个元素）是数值最小的元素，所以默认是小顶堆

具体的源码分析：[Java 优先级队列PriorityQueue详解：从源码分析到实践应用 | Java程序员进阶之路 (tobebetterjavaer.com)](https://tobebetterjavaer.com/collection/PriorityQueue.html)