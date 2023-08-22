#### LinkedList

> 基于 `JDK 1.8` 分析

`LinkedList` 底层基于双向链表实现

````java
public class LinkedList<E> {
    transient int size = 0;

    //指向第一个节点的指针
    transient Node<E> first;

    //指向最后一个节点的指针
    transient Node<E> last;

    private static class Node<E> {
        // 具体元素
        E item;
        // 下一个节点
        Node<E> next;
        // 上一个节点
        Node<E> prev;
    }
}
````

> `LinkedList` 插入和删除元素的时间复杂度？

- 头部插入/删除：只需要修改头结点的指针即可完成插入/删除操作，因此时间复杂度为 O(1)
- 尾部插入/删除：只需要修改尾结点的指针即可完成插入/删除操作，因此时间复杂度为 O(1)
- 指定位置插入/删除：需要先移动到指定位置，再修改指定节点的指针完成插入/删除，时间复杂度为 O(n)

