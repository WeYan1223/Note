#### ArrayList

> 基于 `JDK 1.8` 分析

`ArrayList` 底层是基于数组实现的存储结构，与普通数组相比，`ArrayList` 的容量能动态增长

````java
transient Object[] elementData;
````

##### 1. 初始化

`ArrayList` 的最常用的两个构造方法：

````java
/**
 * 默认初始容量
 */
private static final int DEFAULT_CAPACITY = 10;

/**
 * 空数组，作用是决定添加第一个元素时 elementData 要扩容多少
 */
private static final Object[] EMPTY_ELEMENTDATA = {};

/**
 * 具有默认容量的空数组，作用是决定添加第一个元素时 elementData 要扩容多少
 */
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

/**
 * ArrayList 的底层实现
 */
transient Object[] elementData;

public ArrayList() {
    // 初始时为空数组，当添加第一个元素时，数组容量才变成默认初始容量 10
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
    }
}
````

##### 2. 添加元素

> 末尾添加，`ArrayList#add(E e)`

```java
/**
 * 将指定元素追加到列表末尾
 */
public boolean add(E e) {
    // 确保数组容量能装得下新元素
    ensureCapacityInternal(size + 1);
    elementData[size++] = e;
    return true;
}
```

添加元素前需要确保数组能装得下新元素

````java
/**
 * 确保数组容量能装得下新元素
 */
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

/**
 * 计算数组需要得最小容量
 */
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 第一次添加元素时，数组容量才变成默认初始容量 10
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity) {
    // 记录列表结构被修改的次数
    modCount++;

    if (minCapacity - elementData.length > 0) {
        // 容量不够，需要扩容
        grow(minCapacity);
    }
}

private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    // 扩容为原来的 1.5 倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0) {
        // 这种情况通常是 addAll() 导致扩容 1.5 倍还不够
        newCapacity = minCapacity;
    }
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        newCapacity = hugeCapacity(minCapacity);
    }
    // 扩容，内部调用 System.arraycopy() 做数组拷贝
    elementData = Arrays.copyOf(elementData, newCapacity);
}
````

> 中间插入，`ArrayList#add(int index, E e)`

```java
public void add(int index, E element) {
    // 检查数组越界
    rangeCheckForAdd(index);
    // 确保数组容量能装得下新元素
    ensureCapacityInternal(size + 1);
    // 数组拷贝
    System.arraycopy(elementData, index, elementData, index + 1, size - index);
    elementData[index] = element;
    size++;
}
```

##### 3. 删除元素

> 通过下标删除，`ArrayList#remove(int index)`

````java
public E remove(int index) {
    // 检查数组越界
    rangeCheck(index);
    // 记录列表结构被修改的次数
    modCount++;
    
    E oldValue = elementData(index);

    // 需要移动的长度，如果是最后一个元素，则 numMoved 为 0
    int numMoved = size - index - 1;
    if (numMoved > 0) {
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    }
    elementData[--size] = null;

    return oldValue;
}
````

> 通过元素删除，`ArrayList#remove(Object o)`

```java
/**
 * 删除第一次出现的指定元素（如果存在的话）
 */
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++) {
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
        }
    } else {
        for (int index = 0; index < size; index++) {
            // 通过 equals 判断元素是否相等
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
        }
    }
    return false;
}

/**
 * 快速通过下标删除元素
 */
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0) {
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    }
    elementData[--size] = null;
}
```





> `ArrayList` 和 `Vector` 的区别？

- `ArrayList` 是 `List` 的主要实现类，底层使用 `Object[]`存储，线程不安全 
- `Vector` 是 `List` 的古老实现类，底层使用`Object[]` 存储，线程安全

>`ArrayList` 插入和删除元素的时间复杂度？

对于插入：

* 尾部插入：容量未满时，时间复杂度是 O(1)；容量满时，需要做数组拷贝操作，时间复杂度是 O(n)

- 非尾部插入：需要做数组拷贝操作，时间复杂度是 O(n)

对于删除：

- 尾部删除：时间复杂度是 O(1)

- 非尾部删除：需要做数组拷贝操作，时间复杂度是 O(n)