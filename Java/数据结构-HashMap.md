#### HashMap

> 基于 `JDK 1.8` 分析

`HashMap` 底层基于数组 + 链表 / 红黑树

##### 1. 初始化

```java
public class HashMap<K,V> {
    /**
     * 默认初始容量，16
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
    
    /**
     * 默认负载因子
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
    /**
     * 负载因子
     */
    final float loadFactor;
    
    /*
     * 阈值 = 当前数组长度 * 负载因子
     */
    int threshold;
    
    /**
     * 默认初始容量、默认负载因子
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    }

    public HashMap(int initialCapacity, float loadFactor) {
        this.loadFactor = loadFactor;
        //由于数组长度必须是2的幂次方，所以这里需要对initialCapacity作调整
        this.threshold = tableSizeFor(initialCapacity);
    }
}
```

> `threshold` 有什么作用？

向 `HashMap` 添加元素时，如果元素个数超过阈值（`threshold`），此时需要对 `HashMap` 进行扩容

而 `threshold = 当前数组长度 * 负载因子`，负载因子的默认值 0.75 是对空间和时间效率的一个平衡选择，一般不需要修改

* 如果内存空间很多而又对时间效率要求很高，可以降低负载因子的值
* 如果内存空间紧张而对时间效率要求不高，可以增加负载因子的值，这个值可以大于1

* 负载因子过高，越容易发生哈希碰撞
* 负载因子过低，则会导致浪费空间

##### 2. 添加元素

> `HashMap#put(K, V)`

`HashMap` 添加元素流程如下：

1. 若数组未初始化，先初始化数组
2. 根据 `key` 计算 `hash` 值得到插入的数组索引 `i`
3. 若 `table[i] == null`，则直接添加节点，否则需要进行以下判断：
   * 若 `table[i]` 与 ` key` 相同，新节点覆盖旧节点
   * 否则，说明发生哈希冲突，进行下面步骤
4. 判断是链表还是红黑树：
   * 链表：尾插法插入链表，插入完成后若链表长度大于 8，需要将链表转为红黑树
   * 红黑树：在红黑树中添加新节点
5. 添加完成后，若元素的数量大于 `threshold`，需要进行扩容

> `HashMap` 如何计算元素所在数组的下标？

通常是把 `hash` 值对数组长度取模运算得到下标，但是取模运算的性能是不太好的

首先看 `HashMap` 如何获取 `hash` 值：

```java
/**
 * 若 key 为 null，返回 0，否则进行下面步骤
 * 第一步：h = key.hashCode() 取 hashCode 值
 * 第二步：h ^ (h >>> 16)  使得高位可以参与之后的位运算
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

获取下标：

````java
/**
 * 由于 HashMap 数组的容量一定是 2 的幂次方
 * 所以 hash & (length - 1) 等价于 hash % length
 * 并且位运算比取模运算效率快很多
 */
int index = (table.length - 1) & hash
````

> `HashMap` 解决哈希冲突的方法？

`HashMap` 采用链地址法：数组加链表的结合，在每个数组元素都是一个链表结构，得到数组下标后，把数据放在对应下标元素的链表上

> `HashMap` 的扩容机制？

`HashMap` 添加元素完成后，如果元素个数超过阈值（`threshold`），此时需要对 `HashMap` 进行扩容，通过 `HashMap#resize()`

大致流程如下：

1. 数组大小扩容为原来的两倍（数组大小必须是 2 的幂次方）

2. 将原数组的元素重新分配到新数组，这里又有很巧妙的逻辑：

   由于数组大小扩容为原来的两倍，且下标的计算方式为 `(table.length - 1) & hash`，所以元素的新下标要么是在原位置，要么是在原位置再移动2次幂的位置

> `HashMap` 什么时候将链表转为红黑树？

尾插法插入链表后，若链表长度大于 8，需要将链表转为红黑树，通过 `HashMap#treeifyBin()`

````java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY) {
        //如果数组长度小于64，则链表不会转为红黑树，而是对数组进行扩容
        resize();
    }
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        //将目标链表转为红黑树
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
````

> 为什么在转为红黑树之前，需要多加一层判断：数组长度是否大于等于 64？

从根本触发，链表转为红黑树的目的是为了解决链表过长，导致查询和插入效率慢的问题

要解决这个问题，也可以通过数组扩容，把链表缩短

所以在数组长度还不太长的情况，可以先通过数组扩容来解决链表过长的问题

***

#### HashTable

基本上不用了，需要线程安全的操作应该使用 `ConcurrentHashMap`

* 底层实现：数组 + 链表
* `key` 不能为空
* 线程安全，但是实现方式是对所有公共方法使用 `synchronized` 修饰，锁住整个对象，效率低

***

#### 参考文章

* [Java 8系列之重新认识HashMap - 美团技术团队 (meituan.com)](https://tech.meituan.com/2016/06/24/java-hashmap.html)