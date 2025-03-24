---
title: HashMap源码解析
categories:
  - Java集合框架
  - HashMap
tags:
  - Java集合框架
  - HashMap
---
# HashMap源码解析

 `HashMap`是Java集合框架中非常重要的一个类，它基于哈希表实现，提供了快速的键值对存储和查找功能。HashMap实现了Map接口，允许放入`key`为`null`的元素，也允许插入`value`为`null`的元素。（本篇文章主要以Java 8 为例）

## 1.HashMap的基本结构

`HashMap`内部使用**数组 + 链表/红黑树**的结构存储数据。

- 数组：`Node<K,V>[] table`是一个数组，每个位置称为一个桶(bucket)，用于存储键值对。
- 链表：当多个键值对的哈希值映射到同一个桶时(即同一个下标)，这些键值对会以链表的形式存储。
- 红黑树：如果链表长度超过一定阈值(默认为8)，且数组长度大于等于64，则链表会转换为红黑树。（Java 8 开始）

## 2.核心属性

一下是`HashMap`中的一些重要的成员变量：

```java
//默认初始容量（必须是2的幂），默认16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

//最大容量（2^30）
static final int MAXIMUM_CAPACITY = 1 << 30;

//默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//链表转红黑树的阈值，默认8
static final int TREEIFY_THRESHOLD = 8;

//红黑树退化为链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;

//链表转为红黑树是，table数组的最小容量
static final int MIN_TREEIFY_CAPACITY = 64;

//存储键值对的数组
transient Node<K,V>[] table;

//当前HashMap中的键值对数量 
transient int size;

//修改次数
transient int modCount;

//扩容阈值（capacity * loadFactor)
int threshold;

//负载因子
final float loadFactor;
```

## 3.Node类

`HashMap`的内部静态类`Node`表示**一个键值对节点**。

- `hash`和`key`是`final`修饰的，确保了节点的键和哈希值在创建后不会改变。
- `next`指向下一个节点，当发生哈希冲突时，多个键值对存储在同一个桶中，形成链表或红黑树，`next`用于链接这些节点。

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;  //键的哈希值,通过key.hashCode()计算出来
    final K key;	 //键
    V value;		 //值
    Node<K,V> next;  //指向的下一个节点（链表或红黑树）

    //构造器
    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    //实现 Map.Entry 接口的方法
    public final K getKey()        { return key; } 
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }
	public final V setValue(V newValue) {
        //设置新值返回旧值
        V oldValue = value;
        value = newValue;
        return oldValue;
    }
    //计算节点的哈希码
    public final int hashCode() {
        //通过Objects.hashCode()方法分别计算键和值的哈希码，然后通过异或运算(^)组合起来生成唯一的哈希值
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

	//判断当前节点是否于另一个对象相等
    public final boolean equals(Object o) {
        if (o == this)
            //如果传入的对象o是当前节点本身，返回true
            return true;
        if (o instanceof Map.Entry) {
            //如果传入的是一个Map.Entry对象，则比较两则的键和值是否相等。
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        //前两则都不是，或者键或值不完全相等直接返回false
        return false;
    }
}
```

## 4.方法

### 4.1.构造方法

**无参构造HashMap()**

```java
//默认构造方法
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // 默认负载因子
}
```

**HashMap(int initialCapacity, float loadFactor)**

```java
//指定初始容量和负载因子
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        //如果初始容量小于0，抛出IllegalArgumentException异常
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        //如果初始容量大于最大容量MAXIMUM_CAPACITY(2^30)，直接赋值为MAXIMUM_CAPACITY
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        //如果负载因子小于等于0或者为非数字，抛出IllegalArgumentException异常
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    //调用tableSizeFor(initialCapacity)方法计算扩容阈值
    //扩容公式为：threshold = capacity * loadFactor，但实际的capacity是经过tableSizeFor调整后的值
    this.threshold = tableSizeFor(initialCapacity);
}
```

- **tableSizeFor(int cap)**：`tableSizeFor`方法的作用是将输入的容量`cap`调整为最接近的 2 的幂次方数。

例如：cap = 10，`tableSizeFor`的执行过程：

1. 初始值：`n = cap - 1 = 9`（二进制为 `1001`）。
2. `n |= n >>> 1`：`1001 | 0100 = 1101`（二进制为 `1101`）。
3. `n |= n >>> 2`：`1101 | 0011 = 1111`（二进制为 `1111`）。
4. `n |= n >>> 4`：`1111 | 0000 = 1111`（不变）。
5. `n |= n >>> 8` 和 `n |= n >>> 16`：同样不变。
6. 最终结果：`n = 1111`，加 1 后为 `10000`（二进制表示为 `16`）。

```java
static final int tableSizeFor(int cap) {
    //首先减去1，为了避免cap本身已经是2的幂次方数时，结果被错误的放大一倍。
	int n = cap - 1;
    //位运算，将n的最高位扩展到右侧相邻的位，目的是将n的所有低位都设置为1，从而形成一个连续的1的掩码
	n |= n >>> 1; //等价于n = n | n>>>1
	n |= n >>> 2;
	n |= n >>> 4;
	n |= n >>> 8;
	n |= n >>> 16;
	return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

- **为什么需要 2 的幂次方？**

`HashMap`底层数据结构是数组，键值对通过哈希值映射到数组的索引位置。为了提高性能，`HashMap`使用以下公式计算索引：

```java
index = (n - 1) & hash; //n是table数组大小
```

`(n - 1)`：如果n是2的幂次方数，则n-1的二进制形式是一个全1的掩码。比如：n = 16,n-1=15（二进制1111）
`& hash`：通过按位与运算，快速计算哈希值对应的数组索引，这种方式比取模运算(hash % n)更快。

**HashMap(int initialCapacity)**

```java
//指定初始容量
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);//调用HashMap(int initialCapacity, float loadFactor)构造器
}
```

**HashMap(Map<? extends K, ? extends V> m)**

```java
//使用另一个Map初始化
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR; //默认负载因子
    putMapEntries(m, false);
}
```

### 4.2.put方法

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
	int h;
	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
	return new Node<>(hash, key, value, next);
}
/**
 * @param hash key的哈希值
 * @param key 键
 * @param value 值
 * @param onlyIfAbsent 如果是 true，那么只有在不存在该 key 时才会进行 put 操作
 * @param evict 如果为false,则table处于创建模式,这里不关心
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //判断数组table是否为null,即是否为第一次put值，如果是调用resize()初始化为默认容量 16 或自定义的初始容量。
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //找到具体的数组下标，如果此位置没有值，直接通过newNode()方法，初始化一个节点Node<K,V>,并放置在这个位置。
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {  //数组该位置有数据
        Node<K,V> e; K k;
        //判断该位置的第一个数据和我们要插入的数据，key是不是相等的，如果是，取出该节点。
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //如果该节点代表红黑树，调用红黑树的插入方法，这里不展开说红黑树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //到这一步说明table数组该位置上是一个链表
            for (int binCount = 0; ; ++binCount) {
                //循环遍历，插入到链表的最后面（java7中是插入到链表的最前面）
                if ((e = p.next) == null) {  //这里注意每次循环 e 都指向了下一个节点
                    p.next = newNode(hash, key, value, null);
                    
                    //这里判断该链表插入的值在链表中是否是第 8 个，如果是就调用treeifyBin()方法转为红黑树
                    //TREEIFY_THRESHOLD = 8
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                //如果在该链表中找到了相等的 key
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    //直接break，这时e为链表中 与要插入的新值的key相等 的node
                    break;
                p = e;
            }
        }
        
        //e != null 说明存在旧值的key与要插入的新值的key相等
        //这时直接值覆盖，然后返回旧值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //如果HashMap中由于新插入的值导致 size 超过了阈值，调用resize()扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

### 4.3.resize方法

该方法用于初始化table数组或数组扩容，每次扩容为原来的 2 倍，并进行数据迁移。

```java
final Node<K,V>[] resize() {
    //旧数组
    Node<K,V>[] oldTab = table;
    //旧数组大小
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //旧扩容阈值
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {  //对应数组扩容
        //如果老容量大于等于最大容量MAXIMUM_CAPACITY(2^30)，直接返回旧数组
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //将数组大小扩大一倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            //将扩容阈值扩大一倍
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // 对应使用 new HashMap(int initialCapacity) 初始化后，第一次put时
        newCap = oldThr; //使用扩容阈值作为新容量
    else { 
        // 对应使用 new HashMap() 初始化后，第一次使用put时
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY); //0.75f * 16 = 12
    }
    
    // 如果新阈值为 0，则重新计算
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    
    //用新的数组大小初始化新的数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;  //如果时初始化数组，到这就结束了，返回newTab
    
    if (oldTab != null) {
        //遍历原数组，数据迁移
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //如果该位置有数据，取出并清空该位置的数据 oldTab[j] = null
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //如果该位置只有单个元素，直接赋值过去就行了
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                //如果是红黑树，采用红黑树的方式，这里不具体简绍
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { 
                    //链表的迁移，并且保留了原先的顺序
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
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
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
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

### 4.4.get方法 

```java
public V get(Object key) {
    //创建一个局部变量节点e
    Node<K,V> e;
    //判断key所对应的节点是否为null,为null返回null,不为null返回key所对应的值
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k; 
    //首先判断数据是否为null,为null直接返回null
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //判断第一个节点是否是就是需要的值
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //判断该链表第二个节点是否为null
        if ((e = first.next) != null) {
            //判断是否是红黑树
            if (first instanceof TreeNode)
                //是的话通过红黑树的方式获取
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            
            //链表遍历
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

## 5.总结

- 哈希冲突处理：通过链表或红黑树解决哈希冲突。
- 扩容机制：当`size`超过`threshold`时，`HashMap`会自动扩容，容量变为之前的2倍。
- 性能优化：在 Java 8 中引入了红黑树，当链表超过阈值(默认8)，且table长度大于等于64时，链表会转换为红黑树，从而提高查询效率。