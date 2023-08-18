#### CAS

> `CAS` 是乐观锁的一种实现

`Compare And Swap`，包含三个操作数：内存地址、预期值、新值，伪代码如下：

````c
/**
 * 整个函数是一个原子操作
 */
bool CAS(int* address, int expectedValue, int newValue) {
    int currentValue = *address;
    // 比较address指向的值与expectedValue是否相等
    if (currentValue == expectedValue) {
        // 将newValue存储到address指向的位置
        *address = newValue;
        // 返回true表示CAS操作成功
        return true;
    } else {
        // 返回false表示CAS操作失败
        return false;
    }
}
````

但是，这样不能解决 `ABA` 问题：如果初次读取到变量的值是 `A`，并且在 `CAS` 时检查到它仍然是 `A`，这能说明它的值没有被其他线程修改过了吗？很明显是不能的，因为在这段时间它的值可能被改为其他值 `B`，然后又改回 `A`

`ABA` 问题可以使用版本号解决

> 版本号机制

添加一个版本号来记录变量是否被修改，例如每次修改版本号都自增 1 

此时 `CAS` 就既需要比较值，又需要比较版本号：

* 值不相等：肯定被其他线程改过，不用再比较版本号，`CAS` 提交失败
* 值相等，再比较版本号
  * 版本号相等，则说明没有被改过，`CAS` 提交成功
  * 版本号不等，则就是出现了 `ABA`，`CAS` 提交失败