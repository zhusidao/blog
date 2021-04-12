# HashMap源码分析

```
最近看看了相关的JAVA集合类，这里是自己的jdk1.8 HashMap的个人学习和理解。
```

## 1.存储结构-字段

### 1.数据结构图如下

从结构实现来讲，HashMap是数组+链表+红黑树（JDK1.8增加了红黑树部分，优化查询速度）实现的，如下如所示：

![img](file:////Users/zhusidao/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image001.png)

### 2.java代码中的数据结构定义如下

jdk源码中定义的数据结构代码如下：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash; //hash值
    final K key;
    V value;
    Node<K,V> next; //下一个hash结点
        .......
}
```

## 2.功能实现

### 1.源码中的put操作流程

HashMap类中定义的变量（用于下面泳图）

```java
transient int size; // Map中的键值对数量
transient Node<K,V>[] table; //hash表
int threshold;         // 所能容纳最大键值对个数
```

put方法执行过程可以通过下图来理解：

![img](file:////Users/zhusidao/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image002.png)

1.首先判断是否为空，为空则直接进行扩容操作。

2.不为空则定位到数组位置，并判断是否为空，

###  2.确定哈希桶数组索引位置

1.hash值的计算，代码如下：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); // 异或运算高16位
}
```

2.定位到纵向数组的位置：

```java
// n为2的指数幂，为扩容之后的数组大小，threshold（容纳最大键值对个数）= loadFactor（负载因子）* n（数组大小）
i = (n - 1) & hash // 这里相当于 hash % n，这种位运算速度更快
```

### 3.扩容操作

1.扩容因子默认是0.75，初始化数组大小为16，不超过容量的情况下：容纳最大键值对个 = 负载因子 * 数组大小

2.扩容策略是每次都会去进行向左移动一位，即容量翻倍。

3.每次扩容之后，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置

4.由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的数组的链表中了，这一块就是JDK1.8新增的优化点。

代码和具体说明如下：

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 超过最大值就不再扩充了
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 没超过最大值，就扩充为原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {  // 第一次扩容操作初始化数组大小（16）、容纳最大键值对个数
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 计算新的resize上限
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) { // 把旧的HashMap的数组迁移新的HashMap中
        for (int j = 0; j < oldCap; ++j) { // 遍历数组
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e; // 放入新的数组中
                else if (e instanceof TreeNode) // 如果是红黑数
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { //遍历链表
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                     // 若之前大小为16，扩容后为32，理论上这里应该是和 0000 0000 0001 1111做位运算（也相当于取32的余数操作）
                     // 但是这里简化了操作0000 0000 0001 0000 直接用去做位运算（低四位做位运算的结果一定一样）去判断是否为0
                     // 若之前的链表在数组2号位，这样便能使之前某行的链表均匀分布在新的18号位和旧的2号位
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 原索引放到扩容后的数组中
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 原索引 + 之前数组大小 放到数组中
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

4.put方法

直接上代码吧，相关说明都在代码中写了注解。

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0) // hash表不存在初始化
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null) // 组数上的该结点为空则直接插入值
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k)))) // 数组结点上的hash值和key相等直接覆盖之前的值
            e = p;
        else if (p instanceof TreeNode) // 若节点为红黑树
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) { // 遍历链表
                if ((e = p.next) == null) { // 链表的长度大于8转为红黑树
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash && // 若遍历到某个链表上的结点hash和key相等则结束遍历
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key  // 覆盖之前的hash和key相同的值
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e); // 这里的方法留给一些HashMap再封装类（如：LinkedHashMap）做后续回调操作
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold) // 大于容量则进行扩容操作
        resize();
    afterNodeInsertion(evict); // 类似于afterNodeAccess该方法，做回调操作
    return null;
}
```

 

 