---
title: HashMap源码分析
date: 2019-04-11 20:22:56
tags: [JAVA,集合]
categories: JAVA
---
`HashMap`的实现有分为1.7、1.8两种，1.7的实现为hash桶+链表，而1.8中需要判断整个hashmap元素个数来选择使用1.7的结构，或1.8中新增的红黑树结构。

<!-- more -->

# HashMap源码分析

`HashMap`继承自`AbstractMap`,实现了`Map`接口。在读取时，平均可以做到O(1)时间复杂度。

## 构造函数&&成员变量

#### 常量

```java
//默认初始容量(必须为2的次方，以便hash均衡)
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; //16

//最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;

//默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//HashMap是否转化为红黑树的阈值,
static final int TREEIFY_THRESHOLD = 8;

//用于去除红黑树化的阈值
static final int UNTREEIFY_THRESHOLD = 6;

//红黑树的最小容量（最小为4 * TREEIFY_THRESHOLD，以避免冲突）
static final int MIN_TREEIFY_CAPACITY = 64;

//一个基本的bin节点定义
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;//hash值
    final K key;
    V value;
    Node<K,V> next;//下一节点

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

#### 成员变量

```java
//Node数组，会在初次使用的时候进行初始化(数组长度永远为0或者2的平方)。
transient Node<K,V>[] table;

//Entry集合
transient Set<Map.Entry<K,V>> entrySet;

//键值对数量
transient int size;

//结构性改变hashmap的次数
transient int modCount;

//当前 HashMap 所能容纳键值对数量的最大值，超过这个值，则需扩容。默认为(capacity * load factor)
int threshold;

//负载因子
final float loadFactor;
```



#### 构造函数

```java
//使用默认的负载因子
public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; 
}

//创建指定初始容量和默认负载因子的HashMap
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

//创建指定初始容量和指定负载因子的HashMap
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    //如果传入容量大于最大容量，则使用最大容量创建
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    //合法性判断
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
//返回指定目标容量的最小2的幂。(为什么要进行多次或操作?这是为了使得hash更加均衡)
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

//根据给定的继承自Map的对象构造一个HashMap,其中负载因子使用默认负载因子，初始容量使用传入对象的容量
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
//根据传入Map构造HashMap
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        if (table == null) { // pre-size
            float ft = ((float)s / loadFactor) + 1.0F;//传入Map容量+1
            //判断是否越界
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            //如果计算后的容量大于当前阈值，根据传入容量计算当前阈值
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        //如果传入的键值对数量超过当前阈值，调用resize(下文分析)
        else if (s > threshold)
            resize();
        //循环将Map中的键值对放入当前对象中
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);//下文分析
        }
    }
}
```

## 方法

### put()

```java
//将键值对放入HashMap中，如果key已经在HashMap中，则进行替换
public V put(K key, V value) {
    //先获取key的hash值，再进行putVal方法
    return putVal(hash(key), key, value, false, true);
}
//HashMap中的hash函数，其实就是将传入key的hashcode与本身右移16次进行异或操作，
//以获取到更加平滑（或者说平衡）的hash值
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
//将键值对放入HashMap,其中onlyAbsent为是否不替换存在键值对，这在HashMap中实现为替换，
//在HashSet中为不替换
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab;//Node数组 
    Node<K,V> p;//Node节点引用
    int n, i;
    //如果当前HashMap中的Node数组为空，调用resize()构造一个默认的Node数组
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //将传入的hash再与n-1进行与计算，获取到对应下标的Node节点引用。
    if ((p = tab[i = (n - 1) & hash]) == null)
        //如果此节点为空，调用newNode创建新节点；
        tab[i] = newNode(hash, key, value, null);
    //如果节点不为空(已经存在)
    else {
        Node<K,V> e;//Node节点引用
        K k;
        //判断获取到的节点的hash、key是否与传入的相同，如果相同，将p节点赋值给Node节点引用对象e
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //如果获取到的节点是TreeNode节点(红黑树节点)
        else if (p instanceof TreeNode)
            //调用TreeNode中的putTreeVal进行节点操作，返回操作后的节点
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //采用hash桶+链表结构存储的结构
            for (int binCount = 0; ; ++binCount) {
                //判断获取到的节点的下一引用是否为空
                if ((e = p.next) == null) {
                    //将下一节点设置为新节点
                    p.next = newNode(hash, key, value, null);
                    //判断当前节点数量是否大于需要转化为红黑树的阈值，如果大于则进行红黑树化
                    if (binCount >= TREEIFY_THRESHOLD - 1) 
                        treeifyBin(tab, hash);
                    break;
                }
                //判断当前hash桶中，链表上是否会有相同key的节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                //标记已经存在的节点
                p = e;
            }
        }
        if (e != null) { // 存在key映射
            V oldValue = e.value;
            //替换
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);//一些回调函数，HashMap中未实现
            return oldValue;
        }
    }
    ++modCount;//结构性操作次数+1
    if (++size > threshold)//如果当前HashMap节点个数大于阈值，进行resize()进行重新分配
        resize();
    afterNodeInsertion(evict);//一些回调函数，HashMap中未实现
    return null;
}
```

#### Resize()

```java
//初始化Node数组，或翻倍扩容数组
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;//原数组引用
    int oldCap = (oldTab == null) ? 0 : oldTab.length;//原数组容量
    int oldThr = threshold;//原来的阈值
    int newCap, newThr = 0;//定义新容量、新阈值
    //如果原容量大于0(已经创建了数组)
    if (oldCap > 0) {
        //如果原容量大于或等于最大容量
        if (oldCap >= MAXIMUM_CAPACITY) {
            //将原阈值设为最大值
            threshold = Integer.MAX_VALUE;
            //直接返回旧数组
            return oldTab;
        }
        //将原容量*2赋值给新容量newCap，并进行一些合法性判断
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            //新阈值等于原阈值*2
            newThr = oldThr << 1;
    }
    //如果原阈值不为0
    else if (oldThr > 0) 
        newCap = oldThr;//将原阈值赋给新阈值
    //原容量、原阈值都为默认值情况(初始化Node数组)
    else {               
        newCap = DEFAULT_INITIAL_CAPACITY;//新容量即为默认初始容量
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//新阈值
    }
    //处理边界情况
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    //新建一个Node数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    //如果原数组不为空
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;//方便GC
                if (e.next == null)//如果此节点没有后续节点(当前Hash桶中只有这个元素)
                    //以此节点的hash与新数组长度进行与操作获取下标，在新数组下标处直接放入元素
                    newTab[e.hash & (newCap - 1)] = e;
                //如果节点有后续节点，并且为TreeNode
                else if (e instanceof TreeNode)
                    //使用TreeNode的split进行转移操作
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                //如果没有后续节点，并且为Hash桶+链表结构
                else { 
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