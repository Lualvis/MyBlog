---
title: ArrayList源码解析
tages:
  - ArrayList
categories:
  - ArrayList
---

# ArrayList源码解析

## 1.概述

`ArrayList`是 Java 集合框架中基于动态数组实现的顺序容器，允许放入`null`元素，底层通过**数组实现，非线性安全**。

**特性：**

- 元素有序且可重复。
- 支持随机访问。
- 动态扩容机制。

每个`ArrayList`都有一个容量，表示底层数组的实际大小，容器内存储元素的个数不能多余当前容量。当向容器内添加元素时，如果容量不足，容器会自动进行扩容，每次扩容大约为原来的1.5倍。

**继承体系**

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250323001513.png)

`ArrayList`实现了`List<E>`,`RandomAccess`, `Cloneable`, `java.io.Serializable`这些接口。

- `List<E>`：提供了基础的添加、删除、遍历等操作，并且可以通过下标进行访问。

- `RandomAccess`：提供了随机访问的能力。
- `Cloneable`：表示它具有拷贝能力。
- `Serializable`：表示可以被序列化，也就是可以将对象转换为字节流进行持久化存储或网路传输。



## 2.源码解析

### 2.1.**属性**

```java
/**
 * 默认初始容量
 */
private static final int DEFAULT_CAPACITY = 10;  

/**
 * 空数组，如果传入的容量为0时使用
 */
private static final Object[] EMPTY_ELEMENTDATA = {}; 

/**
 * 空数组，默认容量的数组对象，传传入容量是使用，添加第一个元素时会重新初始化为默认容量（10）。
 */
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};  //

/**
 * 存放元素的数组
 */
transient Object[] elementData; // non-private to simplify nested class access   

/**
 * 集合中元素的个数，默认0
 */
private int size;
```

**（1）DEFAULT_CAPACITY**

默认初始容量，当我们通过`new ArrayList()`创建一个`ArrayList`实例时，底层并不会立即分配容量为 10 的数组，而是初始化为一个空数组`DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}`。只有在添加第一个元素时，`ArrayList`才会将底层数组的容量扩展为 10。这种延迟初始化的设计可以节省内存开销，避免不必要的资源浪费 。

**（2）EMPTY_ELEMENTDATA**

空数组，`new ArrayList(0)`创建的`ArrayList`实例。这种情况下，`ArrayList`的底层数组被显示的初始化为空数组，且容量为0。这个与通过无参构造器创建的不同，后者使用的是`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`。

**（3）DEFAULTCAPACITY_EMPTY_ELEMENTDATA**

也是空数组，但是这个专门用于通过无参构造器`new ArrayList()`创建的`ArrayList`实例。与`EMPTY_ELEMENTDATA`的区别在于，当向`ArrayList`中添加第一个元素时，`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`会被初始化为默认容量`DEFAULT_CAPACITY`（即10），而`EMPTY_ELEMENTDATA`则不会触发这种行为。

**（4）elementData**

真正存放元素的地方，他是一个`Object[]`类型的数组，为了优化序列化过程，使用了`transient`关键字修饰，这意味着该字段不会被默认的 Java 序列化机制直接序列化。

这样做的原因是，`ArrayList`只需要序列化实际存储的元素（即前**size**个元素），而不是整个底层数组，这样可以减少序列化数据的大小，从而提高性能。

**（5）size**

集合中元素的个数，而不是`elementData`数组的长度。size 的值始终**小于或等于**`elementData.length`，因为 `elementData` 的容量可能会大于实际存储的元素数量（例如扩容后剩余的空闲空间）。



### 2.2.**构造方法**

**ArrayList(int initialCapacity)**

传入初始容量（initialCapacity），如果大于零就初始化elementData为对应大小的数组，如果为0就是用`EMPTY_ELEMENTDATA`空数组看，小于零就抛出异常。

```java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        //如果传入的初始容量大于零，new一个initialCapacity大小的Object数组
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```

**无参构造ArrayList()**

无参构造，初始化为`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`空数组，在添加第一个元素时扩容为默认初始化容量`DEFAULT_CAPACITY = 10`

```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

**ArrayList(Collection<? extends E> c)**

传入集合参数并将集合转为数组，根据数组长度设置size，并判断集合是否为null，如果不是null，判断是否是ArrayList类型，如果是就直接赋值给elementData，因为 ArrayList 的底层数组已经是 `Object[]` 类型，无需额外拷贝；如果集合是null，直接将elementData赋值为EMPTY_ELEMENTDATA空数组。

```java
public ArrayList(Collection<? extends E> c) {
    //将任意集合类型转为数组形式（不同集合类型底层数组类型可能不一样）
    Object[] a = c.toArray();
    if ((size = a.length) != 0) { // 将size设为a.length，并判断集合c是否为null
        if (c.getClass() == ArrayList.class) { //判断集合c是否是ArrayList类型
            elementData = a; //如果是，直接赋值给elementData
        } else {
            //如果不是，使用Array.copyOf()方法复制c里的数据到新的Object[]数组中，然后赋值给elementData
            elementData = Arrays.copyOf(a, size, Object[].class);  
        }
    } else {
        // replace with empty array.
        elementData = EMPTY_ELEMENTDATA;  //如果集合c为null，则初始化为空数组EMPTY_ELEMENTDATA
    }
}

```

### 2.3.新增方法

**add(E e)**

添加元素到末尾，平均时间复杂度为O(1)。

- 首先调用`ensureCapacityInternal(int minCapacity)`方法检查是否需要扩容
- 如果elementData等于DEFAULTCAPACITY_EMPTY_ELEMENTDATA空数组，则初始化为DEFAULT_CAPACITY
- 通过`grow(int minCapacity)`方法进行扩容，**新容量是老容量的1.5倍**（oldCapacity + (oldCapacity >> 1)）；如果新容量还是比**所需最小容量**（minCapacity）小，则以需要的容量为准。
- 通过`Arrays.copyOf()`方法创建新容量的数组，并将老数组拷贝到新数组。

```java
public boolean add(E e) {
    //siez + 1 检查是否需要扩容
    ensureCapacityInternal(size + 1); //size+1 所需最小容量
    //将元素插入最后一位
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
    //判断elementData是否是DEFAULTCAPACITY_EMPTY_ELEMENTDATA空数组，如果是初始化为默认大小10
	if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    //返回最小需求容量
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    if (minCapacity - elementData.length > 0)
        //扩容
        grow(minCapacity);
}

private void grow(int minCapacity) {
	int oldCapacity = elementData.length;  //老容量
	int newCapacity = oldCapacity + (oldCapacity >> 1);  //新容量，相当于oldCapacity + (oldCapacity / 2)，扩容为原先的1.5倍
	if (newCapacity - minCapacity < 0)
        //如果新容量小于当前所需最小容量（minCapacity），直接赋值为minCapacity
		newCapacity = minCapacity;
	if (newCapacity - MAX_ARRAY_SIZE > 0)
        //如果新容量已经超过最大容量了，则使用最大容量
		newCapacity = hugeCapacity(minCapacity);
    // 以新容量拷贝出来一个新数组
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

**add(int index, E element)**

添加元素到指定位置，平均时间复杂度为O(n)。

- 调用`rangeCheckForAdd(int index)`方法检查索引是否越界
- 检查是否需要扩容
- 把插入索引位置后的元素都往后挪一位
- 在插入索引位置放置插入的元素
- 大小加1

```java
public void add(int index, E element) {
    //检查是否越界
    rangeCheckForAdd(index);
	//检查是否需要扩容
    ensureCapacityInternal(size + 1);
    //将inex及其之后的元素往后挪一位，则index位置处就空出来了
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    //将元素插入到index位置
    elementData[index] = element;
    //大小加1
    size++;
}

private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

`System.arraycopy()`是 Java 中高效的复制数组的一个本地方法。它允许将一个数组中的元素从指定位置复制到另一个数组的指定位置。

```java
public static native void arraycopy(
    Object src,    // 源数组
    int srcPos,    // 源数组的起始位置
    Object dest,   // 目标数组
    int destPos,   // 目标数组的起始位置
    int length     // 要复制的元素数量
);
```

**addAll(Collection<? extends E> c)** 

将一个集合中的元素添加到当前集合，即求两个集合的并集。

```java
public boolean addAll(Collection<? extends E> c) {
    //将集合c转为数组
    Object[] a = c.toArray(); 
    //获取集合c的元素个数
    int numNew = a.length;
    //检查是否需要扩容
    ensureCapacityInternal(size + numNew);  
    //将集合中的元素全部拷贝到elementData数组的最后
    System.arraycopy(a, 0, elementData, size, numNew);
    //增加numNew个大小
    size += numNew;
    //如果集合c不等于null返回true,否则返回false
    return numNew != 0;
}
```

**addAll(int index, Collection<? extends E> c)**

将指定集合c中的所有元素插入到当前ArrayList的指定位置index.

- 检查索引的合法性
- 将集合转为数组并获取其长度
- 检查是否需要扩容
- 将现有元素向后移动，为新元素腾出空间 
- 将集合中的元素复制到目标位置
- 调整列表大小并返回操作结果

```java
public boolean addAll(int index, Collection<? extends E> c) {
    //检查是否越界
    rangeCheckForAdd(index);

    //将集合c转为数组
    Object[] a = c.toArray();
    //获取集合c的元素个数
    int numNew = a.length;
    //检查是否需要扩容
    ensureCapacityInternal(size + numNew); 
	//计算需要移动的元素数量 numMoved，即从 index 开始到列表末尾的元素个数
    int numMoved = size - index;
    if (numMoved > 0)
        //如果 numMoved > 0，则将index及其之后的元素向后移动numNew个位置，为新元素腾出空间
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);
	//将集合c中的元素复制到elementData的指定位置index
    System.arraycopy(a, 0, elementData, index, numNew);
    //size增加numNew个大小
    size += numNew;
    //如果集合c不等于null返回true,否则返回false
    return numNew != 0;
}
```

### 2.4.其他方法

**get(int index)**

通过下标获取元素。

- 首先通过`rangeCheck(index)`方法检查是否越上界，如果越上界抛出`IndexOutOfBoundsException`异常，这里无需检查下界，在Java中，数组访问本身会自动检查下界，例如：

  ```java
  Object[] array = new Object[10];
  array[-1] = "test"; // 这里会抛出 ArrayIndexOutOfBoundsException
  ```

  因此，对于 `ArrayList` 的实现来说，不需要额外检查 `index < 0`，因为即使不检查，JVM 也会在访问底层数组时抛出异常 。

- 返回数组index位置的元素。

```java
public E get(int index) {
    //检查是否越界
    rangeCheck(index);
	//返回数组index位置的元素
    return elementData(index);
}

private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

**remove(int index)**

删除指定索引位置的元素。

- 检查是否越界
- 获取指定位置的元素
- 如果删除的不是最后一位，则index之后的元素往前移一位
- 将最后一位置为null，方便GC回收
- 返回删除元素

```java
public E remove(int index) {
    //检查是否越界
    rangeCheck(index);

    modCount++;
    //获取index位置的元素
    E oldValue = elementData(index);

    //如果index不是最后一位，则将index之后的元素往前挪动一位
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    //将最后一个元素删除
    elementData[--size] = null; // clear to let GC do its work
	//返回旧值
    return oldValue;
}
```

**remove(Object o)**

删除指定元素值的元素，时间复杂度O(n)。

```java
public boolean remove(Object o) {
    if (o == null) {
        //遍历整个数组，找到元素第一次出现的位置，并将其快速删除
        for (int index = 0; index < size; index++)
            //如果要删除的元素为null
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        // 遍历整个数组，找到元素第一次出现的位置，并将其快速删除
        for (int index = 0; index < size; index++)
            //如果要删除的元素不为null，则进行比较，使用equals()方法
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

//这个方法更remove(int index)方法几乎差不多，但是少了检查索引越界的操作，这里通过值删除不需要检查索引，优化了性能
private void fastRemove(int index) {
	modCount++;
	int numMoved = size - index - 1;
    //如果index不是最后一位，则将index之后的元素往前挪动一位
	if (numMoved > 0)
		System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    //将最后一个元素删除
	elementData[--size] = null; // clear to let GC do its work
}
```
