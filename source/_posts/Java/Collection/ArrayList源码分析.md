---
title: ArrayList源码分析
date: 2019-04-07 15:20:01
tags: [JAVA,集合]
categories: JAVA
---
`ArrayList`继承自`List`接口，实现了get()、set()、add()、remove()等通用接口。

<!-- more -->

# ArrayList 源码分析

`ArrayList`是基于数组的容器，具有随机读(Random access)的特点，在执行get()、set()具有O(1)时间复杂度，而在add()、remove()等操作，由于需要将部分数组移动，则需要O(n)时间复杂度。

## 构造函数&&成员变量

#### 成员变量

```java
//默认容量
private static final int DEFAULT_CAPACITY = 10;

//用于构造初始容量为0的空ArrayList
private static final Object[] EMPTY_ELEMENTDATA = {};

//用于创建默认大小的空ArrayList。 
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

//存储ArrayList元素的数组。
transient Object[] elementData; 

//ArrayList元素大小
private int size;
```

#### 构造函数

```java
//构造一个带有指定容量的ArrayList
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

//使用默认空数组构造一个ArrayList
public ArrayList() {
    //首先使用默认的空数组创建
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

//将别的继承自Collection的集合转化为ArrayList
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

------
## 方法

### Add(E e)

```java
//增加一个元素
public boolean add(E e) {
    //判断当前容量是否大于此容器目前可用容量
    ensureCapacityInternal(size + 1); 
    //在数组尾部增加新元素，数组大小加1
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
//计算当前数组容量
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    //判断当前数组是否为默认创建的空组数，如果是则返回默认容量(10)
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
//判断传入的容量
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;//修改数组的次数(AbstractList)

    //如果传入容量大于数组大小，则进行grow判断是否扩容
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

//
private void grow(int minCapacity) {
    //原数组大小
    int oldCapacity = elementData.length;
    //新数组大小
    int newCapacity = oldCapacity + (oldCapacity >> 1);// >> 1为符号数右移，相当于除2
    //如果扩容后的数组容量还是小于传入的数组大小
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    //如果扩容后的数组容量大于最大数组大小
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    //转移旧数组元素到新数组
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

------

### Add(int index,E element)

```java
//在指定位置插入指定元素
public void add(int index, E element) {
    //检查下标是否在数组内，不在就抛出异常
    rangeCheckForAdd(index);
    //判断当前容量是否大于此容器目前可用容量
    ensureCapacityInternal(size + 1);  
    //右移下标开始数组
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    //设置元素
    elementData[index] = element;
    size++;
}

//add()、addAll()使用的数组范围方法
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

------

### Remove(int index)

```java
//将数组中index下标的元素移除，将剩余右边的数组全部左移
public E remove(int index) {
    //检查index下标是否合法，否则抛出异常
    rangeCheck(index);

    modCount++;//操作数组的次数
    E oldValue = elementData(index);
    //需要左移的数组头下标
    int numMoved = size - index - 1;
    if (numMoved > 0)
        //左移 O(n)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    //左移后将剩下的对象设为Null，方便GC
    elementData[--size] = null; 
    //返回旧值	
    return oldValue;
}
//判断合法性
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

------

### Remove(Object o)

```java
//如果存在对象o,则删除
public boolean remove(Object o) {
    //如果传入为null
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
        //如果传入不为null，调用equals判断
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

//跳过边界检查的移除元素方法
private void fastRemove(int index) {
    modCount++;//操作数组次数
    //后续需要左移的数组头下标
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    //便于GC
    elementData[--size] = null;
}
```

------

### Get(int index)

```java
//O(1)时间复杂度获取元素
public E get(int index) {
    //检查范围
    rangeCheck(index);

    return elementData(index);
}
```

------

### Set(int index,E element)

```java
//在指定下标处替换元素
public E set(int index, E element) {
    //检查范围
    rangeCheck(index);

    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}
```

