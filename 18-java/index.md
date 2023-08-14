# 

[toc]

被迫营业Java的第一天 [2023/05/09]

# 一、集合

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230519144406652.png" alt="image-20230519144406652" style="zoom: 50%;" />

## Collection

### List

#### ArrayList

> #### 问：ArrayList底层的实现原理是什么？

- ArrayList 底层是用动态数组实现的；

- 若不指定初始化容量，ArrayList 初始容量为 0，当第一次添加数据的时候才会初始化容量为 10；若指定初始容量，那么就直接分配该容量大小的数组，不进行扩容；

- ArrayList 再进行扩容的时候是原来容量的 1.5 倍，每次扩容都需要拷贝数组；

- ArrayList 再添加数据的时候：

  - 确保数组已使用长度(size)+1后足够存下下一个数据；
  - 计算数组的容量，如果当前数组已使用长度+1后 > 当前数组长度，则调用 grow 方法扩容(原来的1.5倍)；
  - 确保新增的数据有地方存储后，则将新元素添加到位于 size 的位置上；
  - 返回添加成功的布尔值。

  

## Map

### HashMap 底层实现

和 Go 也大差不差，记几个关键词即可：

- 拉链法；
- 扰动函数减少 hash冲突；
- 与运算(因此数组长度只能是2的整数次幂，JavaGuide 解释的非常不好。。)
- 转换红黑树。

```java
    static final int hash(Object key) {
      int h;
      // key.hashCode()：返回散列值也就是hashcode
      // ^：按位异或
      // >>>:无符号右移，忽略符号位，空位都以0补齐
      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
```

JDK1.8 之后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）时，会判断是否要将链表转化为红黑树：

- 当前**数组的长度小于 64**，那么会选择先进行**数组扩容**；
- 否则，**转成红黑树**，以减少搜索时间。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/jdk1.8_hashmap.png" alt="jdk1.8之后的内部结构-HashMap" style="zoom: 67%;" />

### HashMap 有哪些遍历方式?

先说结论：**存在阻塞时 parallelStream 性能最高, 非阻塞时 parallelStream 性能最低** 。

懒得记了，看链接：[HashMap 的 7 种遍历方式与性能分析！](https://mp.weixin.qq.com/s/zQBN3UvJDhRTKP6SzcZFKw)

HashMap **遍历从大的方向来说，可分为以下 4 类**：

1. 迭代器（Iterator）方式遍历；
2. For Each 方式遍历；
3. Lambda 表达式遍历（JDK 1.8+）;
4. Streams API 遍历（JDK 1.8+）。

但每种类型下又有不同的实现方式，因此具体的遍历方式又可以分为以下 7 种：

1. **使用迭代器（Iterator）EntrySet 的方式进行遍历**；
2. 使用迭代器（Iterator）KeySet 的方式进行遍历；
3. 使用 For Each EntrySet 的方式进行遍历；
4. 使用 For Each KeySet 的方式进行遍历；
5. 使用 Lambda 表达式的方式进行遍历；
6. 使用 Streams API 单线程的方式进行遍历；
7. 使用 Streams API 多线程的方式进行遍历。



### ConcurrentHashMap 和 Hashtable 的区别

**底层数据结构**：JDK1.7 的 `ConcurrentHashMap` 底层采用 **分段的数组+链表** 实现，JDK1.8 采用的数据结构跟 `HashMap1.8` 的结构一样，数组+链表/红黑二叉树。

**实现线程安全的方式:star:**：

- 在 JDK1.7 的时候，`ConcurrentHashMap` 对整个桶数组进行了**分割分段(`Segment`，分段锁)**，每一把锁只锁容器其中一部分数据（下面有示意图），多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率；

![Java7 ConcurrentHashMap 存储结构](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/java7_concurrenthashmap.png)

- 到了 JDK1.8 的时候，`ConcurrentHashMap` 已经摒弃了 `Segment` 的概念，而是直接用 `Node`数组+链表+红黑树的数据结构来实现，并发控制使用 `synchronized` 和 `CAS` 来操作。（JDK1.6 以后 `synchronized` 锁做了很多优化） 整个看起来就像是优化过且线程安全的 `HashMap`，虽然在 JDK1.8 中还能看到 `Segment` 的数据结构，但是已经简化了属性，只是为了兼容旧版本；

![Java8 ConcurrentHashMap 存储结构](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/java8_concurrenthashmap.png)

- **`Hashtable`(同一把锁)** :使用 `synchronized` 来保证线程安全，效率非常低下。当一个线程访问同步方法时，其他线程也访问同步方法，可能会进入阻塞或轮询状态，如使用 put 添加元素，另一个线程不能使用 put 添加元素，也不能使用 get，竞争会越来越激烈效率越低。

![Hashtable 的内部结构](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/jdk1.7_hashmap.png)

### ConcurrentHashMap 线程安全的具体实现方式/底层具体实现

##### JDK1.8 之前

![Java7 ConcurrentHashMap 存储结构](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/java7_concurrenthashmap.png)

首先将数据分为一段一段（这个“段”就是 `Segment`）的存储，然后**给每一段数据配一把锁**，当一个线程占用锁访问其中一个段数据时，其他段的数据也能被其他线程访问。

**`ConcurrentHashMap` 是由 `Segment` 数组结构和 `HashEntry` 数组结构组成**。

一个 `ConcurrentHashMap` 里包含一个 `Segment` 数组，`Segment` 的个数一旦**初始化就不能改变**。`Segment` 数组的大小默认是 16，也就是说**默认可以同时支持 16 个线程并发写**。

`Segment` 的结构和 `HashMap` 类似，是一种数组和链表结构，一个 `Segment` 包含一个 `HashEntry` 数组，每个 `HashEntry` 是一个链表结构的元素，每个 `Segment` 守护着一个 `HashEntry` 数组里的元素，当对 `HashEntry` 数组的数据进行修改时，必须首先获得对应的 `Segment` 的锁。也就是说，**对同一 `Segment` 的并发写入会被阻塞，不同 `Segment` 的写入是可以并发执行的**。

##### JDK1.8 之后

![Java8 ConcurrentHashMap 存储结构](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/java8_concurrenthashmap.png)

Java 8 几乎完全重写了 `ConcurrentHashMap`，代码量从原来 Java 7 中的 1000 多行，变成了现在的 6000 多行。

**`ConcurrentHashMap` 取消了 `Segment` 分段锁**，**采用 `Node + CAS + synchronized` 来保证并发安全**。数据结构跟 `HashMap` 1.8 的结构类似，数组+链表/红黑二叉树。**Java 8 在链表长度超过一定阈值（8）时将链表（寻址时间复杂度为 O(N)）转换为红黑树（寻址时间复杂度为 O(log(N))）**。

Java 8 中，锁粒度更细，`synchronized` 只锁定当前链表或红黑二叉树的首节点，这样只要 hash 不冲突，就不会产生并发，就不会影响其他 Node 的读写，效率大幅提升。

### JDK 1.7 和 JDK 1.8 的 ConcurrentHashMap 实现有什么不同？

其实主要就是**把段分的更细了**。

- **线程安全实现方式**：JDK 1.7 采用 `Segment` 分段锁来保证安全， `Segment` 是继承自 `ReentrantLock`。JDK1.8 放弃了 `Segment` 分段锁的设计，采用 `Node + CAS + synchronized` 保证线程安全，锁粒度更细，`synchronized` 只锁定当前链表或红黑二叉树的首节点。
- **Hash 碰撞解决方法** : JDK 1.7 采用拉链法，JDK1.8 采用拉链法结合红黑树（链表长度超过一定阈值时，将链表转换为红黑树）。
- **并发度**:star:：JDK 1.7 最大并发度是 Segment 的个数，默认是 16。JDK 1.8 最大并发度是 Node 数组的大小，并发度更大。

### HashMap 和 HashTable 的区别？

- **线程是否安全**：`HashMap` 是非线程安全的，`Hashtable` 是线程安全的,因为 `Hashtable` 内部的方法基本都经过`synchronized` 修饰。（如果你要保证线程安全的话就使用 `ConcurrentHashMap` 吧！）；
- **效率**：因为线程安全的问题，`HashMap` 要比 `Hashtable` 效率高一点。另外，`Hashtable` 基本被淘汰，不要在代码中使用它；
- **Key是否能为null**：`HashMap` 可以存储 null 的 key 和 value，但 null 作为键只能有一个，null 作为值可以有多个；`Hashtable` 不允许有 null 键和 null 值，否则会抛出 `NullPointerException`。
- **初始容量与扩容**：
  - 创建时如果**不指定容量初始值**：`Hashtable` 默认的初始大小为 11，之后每次扩充，容量变为原来的 2n+1；`HashMap` 默认的初始化大小为 16。之后每次扩充，容量变为原来的 2 倍。
  - 创建时如果**给定了容量初始值**：那么 `Hashtable` 会直接使用**给定的大小**，而 `HashMap` 会将其扩充为 2 的幂次方大小。也就是说 `HashMap` 总是使用 **2 的幂**作为哈希表的大小。
- **底层数据结构**：`HashMap`中，当链表长度大于阈值（默认为 8）时，会判断是否要将链表转化为红黑树(`Hashtable` 没有这样的机制)：
  - 当前**数组的长度小于 64**，那么会选择先进行**数组扩容**；
  - 否则，转成红黑树，以减少搜索时间。

### HashMap 和 HashSet 的区别？

`HashSet` 底层就是基于 `HashMap` 实现的。

| `HashMap`                              | `HashSet`                                                    |
| -------------------------------------- | ------------------------------------------------------------ |
| 实现了 `Map` 接口                      | 实现 `Set` 接口                                              |
| 存储键值对                             | 仅存储对象                                                   |
| 调用 `put()`向 map 中添加元素          | 调用 `add()`方法向 `Set` 中添加元素                          |
| `HashMap` 使用键（Key）计算 `hashcode` | `HashSet` 使用**成员对象**来计算 `hashcode` 值，对于两个对象来说 `hashcode` 可能相同，所以`equals()`方法用来判断对象的相等性 |

### HashMap 和 TreeMap 的区别？

TreeMap 实现`SortedMap`接口，可以对集合中的元素根据**键**排序。

### HashSet 如何检查重复？

当通过`add()`把对象加入`HashSet`时：

- `HashSet` 会先计算对象的`hashcode`值来判断对象加入的位置，同时也会与其他加入的对象的 `hashcode` 值作比较；
- 如果没有相等的 `hashcode`，`HashSet` 会假设对象没有重复出现；
- 但是如果发现有相同 `hashcode` 值的对象，这时会调用`equals()`方法来检查 `hashcode` 相等的对象是否真的相同。如果两者相同，`HashSet` 就不会让加入操作成功。（注意，这是不对的。）

`HashSet`的`add()`方法只是简单的调用了`HashMap`的`put()`方法，并且判断了一下返回值以确保是否有重复元素。

```java
private static final Object PRESENT = new Object();	// PRESENT 其实就是个空 Object
// Returns: true if this set did not already contain the specified element
// 返回值：当 set 中没有包含 add 的元素时返回真
public boolean add(E e) {
        return map.put(e, PRESENT)==null;	// HashSet 并非没有 Val，其实也是 Key-Val，Val 默认为 PRESENT
}

// Returns : previous value, or null if none
// 返回值：如果插入位置没有元素返回null，否则返回上一个元素
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
...
}
```

- 上面的代码可以看出，HashSet 插入时，调用的是 HashMap 的 put()，把 Val 置为默认的 PRESENT；

- 重复插入的时候，也会像 HashMap 一样操作：**对于已存在的 Key，就会更新它的 Val，并返回旧的 Val**。对于 HashSet来说，**插入重复的 Key，也就是会返回 PRESENT**，然后判定其不为 null，返回 false，**"相当于"**插入失败。

也就是说，在 JDK1.8 中，**实际上无论`HashSet`中是否已经存在了某元素，`HashSet`都会直接插入，只是会在`add()`方法的返回值处告诉我们插入前是否存在相同元素**。

# 二、并发编程


