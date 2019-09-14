---
title: LinkedList源码分析
date: 2019-04-07 21:26:17
tags: [JAVA,集合]
categories: JAVA
---
`LinkedList`同样继承自`List`接口，但却是基于链表实现。

<!-- more -->

# LinkedList源码分析

`LinkedList`实现了`List`、`Deque`等接口，同时具有列表与队列的性质。`LinkedList`本质上是一个双向链表，在插入、删除元素上具有O(1)时间复杂度的优势，但是在获取元素时并没有`ArrayList`一样随机读的优势，而是要通过一个个`Node`进行遍历判断得到，时间复杂度为O(n)。 

## 构造函数&&成员变量

#### 成员变量

```java
//初始大小
transient int size = 0;

//头节点
transient Node<E> first;

//尾节点
transient Node<E> last;

//节点的内部类定义
private static class Node<E> {
    E item;//节点包含的数据
    Node<E> next;//下一个节点
    Node<E> prev;//上一个节点

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

#### 构造函数

```java
//默认构造函数，什么都不操作。只有在元素变动的时候进行初始化等操作。
public LinkedList() {
}
//通过实现了collection接口的集合构造
public LinkedList(Collection<? extends E> c) {
    this();
    //将集合c全部加入到链表
    addAll(c);
}
```

## 链表操作

### Add(E e)

```java
//增加一个元素
public boolean add(E e) {
    linkLast(e);
    return true;
}

//在链表尾部增加一个元素，只需要更换引用关系，时间复杂度为O(1)
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);//根据对象构造一个新节点
    last = newNode;//将尾节点替换为新节点
    //如果尾节点为空，则将头节点替换为新节点  (初始时，first与last都为null)
    if (l == null)
        first = newNode;
    //否则，将新节点设置为尾节点的下一个节点
    else
        l.next = newNode;
    size++;
    modCount++;//链表操作次数+1
}
```

------

### Add(int index,E element)

```java
//在指定位置插入元素，更改引用关系，时间复杂度为O(1)
public void add(int index, E element) {
    //检查下标是否合法，否则抛出异常
    checkPositionIndex(index);

    //如果下标与链表大小相同，即在尾部插入元素
    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}

private void checkPositionIndex(int index) {
    if (!isPositionIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
//判断下标是否合法
private boolean isPositionIndex(int index) {
    return index >= 0 && index <= size;
}

//在链表指定元素前插入元素
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    //设置指定元素前节点为指定节点
    succ.prev = newNode;
    //如果指定元素没有前节点，将链表头节点设置为指定节点
    if (pred == null)
        first = newNode;
    //否则，将指定元素前节点设置为新节点
    else
        pred.next = newNode;
    size++;
    modCount++;//链表操作次数+1
}
```



### Remove(Object o)

```java
//移除链表中第一次出现的指定对象，由于需要不断遍历，remove操作需要O(n)时间复杂度
public boolean remove(Object o) {
    //当指定元素为null情况
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
        //不为空情况，使用equals判断
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}

//在链表里移除一个元素
E unlink(Node<E> x) {
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;
    //如果目标元素没有前节点，链表的头节点即为目标节点的后节点(移除链表头)
    if (prev == null) {
        first = next;
        //将目标节点的后节点设置为目标节点的前节点
    } else {
        prev.next = next;
        //方便GC
        x.prev = null;
    }
    //如果目标元素没有后节点，链表的尾节点设置为目标节点的前节点(移除链表尾)
    if (next == null) {
        last = prev;
        //将目标节点的前节点设置为目标节点后节点的前节点
    } else {
        next.prev = prev;
        //GC
        x.next = null;
    }
    //GC
    x.item = null;
    size--;
    modCount++;//链表操作次数+1
    return element;//返回移除元素
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
//根据下标获取元素，需要遍历，时间复杂度为O(n)
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

//判断下标是否存在，不存在则抛出异常
private void checkElementIndex(int index) {
    if (!isElementIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

//判断下标是否存在
private boolean isElementIndex(int index) {
    return index >= 0 && index < size;
}

//在指定下标获取元素
Node<E> node(int index) {
    //如果下标小于链表长度的一半，则从链表头节点开始遍历
    if (index < (size >> 1)) {
        Node<E> x = first;
        //遍历
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
        //下标大于链表长度一半，则从链表尾节点开始遍历
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

------

### Set(int index,E element)

```java
//在指定下标处替换元素
public E set(int index, E element) {
    //判断下标是否存在，不存在则抛出异常
    checkElementIndex(index);
    //获取指定下标元素
    Node<E> x = node(index);
    //替换，并返回旧指
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
```

------

## 队列操作

### 获取头元素 --- peek()、element()、poll()、remove()

```java
//获取队列头元素
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}

//获取队列头元素,如果不存在则抛NoSuchElementException异常
public E element() {
    return getFirst();
}
public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
}

//获取队列头元素，并移除
public E poll() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
}
//移除元素
private E unlinkFirst(Node<E> f) {
    final E element = f.item;
    final Node<E> next = f.next;//目标元素的下一节点
    f.item = null;// help GC
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}

//获取队列头元素，并移除。如果元素不存在则抛NoSuchElementException异常
public E remove() {
    return removeFirst();
}
public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
}
```

------

### 在队列尾部增加元素 --- offer()

```java
public boolean offer(E e) {
    return add(e);
}
```