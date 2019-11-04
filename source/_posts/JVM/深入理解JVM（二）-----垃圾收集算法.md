---
title: 深入理解JVM（二） --- 垃圾收集算法
date: 2019-08-03 23:24:21
tags: [JAVA,JVM]
categories: JAVA
---

How GC algorithm works

<!-- more -->

# 深入理解JVM（二）---垃圾收集算法

在讲垃圾收集算法之前，我们先考虑如何确定一个对象需要被回收。

#### 如何标记一个需要被回收的对象？

- 引用计数算法

**引用计数**（Reference Count），就是给对象添加一个引用计数器，每当被其他一个地方引用时，计数器值就+1；当引用失效时，计数器值就-1；在任何时刻计数器为0的对象都不可再被使用。

客观来说，引用技术算法的实现简单，效率也高，比如ActionScript、python等都使用了引用计数来进行内存管理，但是再java虚拟机中由于很难解决循环引用的问题，并没有使用RC算法。

```java
public class ReferenceCountingGC{
    public Object instance = null;
    
    public static void testGc() {
        ReferenceCountingGC objA = new ReferenceCountingGC();
        ReferenceCountingGC objB = new ReferenceCountingGC();
        //相互引用，互相的RC都加1
        objA.instance = objB;
        objB.instance = objA;
        
        objA = null;
        objB = null;
        
        //此时，如果发生GC，即便两个对象都指向了null，但是相互的引用关系还是存在的，所以不会被JVM标记回收。
    }
}
```

- 可达性分析算法

在主流的编程语言中（Java、C#、Lisp）等，都是通过**可达性分析**（Reachability Analysis）来判断对象是否存活。算法的基本思路，即是判断对象是否在GC Root引用链上，如果一个对象没有与任何引用链相连，则证明此对象是不可用的。

GC Root对象一般为：

1. 虚拟机栈（栈帧中的本地变量表）中引用的对象

2. 方法区中类静态属性引用的对象（private static Cat cat = new Cat()）
3. 方法区中常量引用的对象（private Logger log = new Logger()）
4. 本地方法栈中的JNI引用的对象。

#### 被标记了就一定会被回收吗？

即使经历过可达性分析中被标记为不可达对象，这个对象也并非是“非死不可”的，真正宣告一个对象的死亡，至少需要经历两次标记：对象在进行可达性分析时，被判断为不可达，此时将会被第一次标记且进行一次筛选过程，筛选的条件是此对象是否有必要执行finalize()方法：

##### 没有必要执行finalize方法

当对象没有覆写过finalize方法，或者finalize方法已经被调用过，虚拟机会将这两种情况都视为“没有必要执行”。

##### 有必要执行finalize方法

首先会将对象放在一个叫“F-Queue“的队列中，并在之后被一个由虚拟机创建的低优先级Finalizer线程执行后续操作，稍后GC会对F-Queue中的对象进行第二次小规模的标记。Finalizer会执行对象的finalize方法，而对象如果想逃脱被回收的命运，只需要在重新与引用链连上即可，此时GC会将此对象移除出队列；如果对象此时还没有逃脱，那就是真的会被回收了。(对象只会被拯救一次，因为一个对象的finalize方法最多只会被系统调用一次)

```java
public class FinalizeEscapeGC{
    public static FinalizeEscapeGC SAVE_HOOK = null;
    
    public void isAlive() {
        System.out.println("I'm still alive");
    }
    
    @Override
    protected void finalize() throws Thorwable{
        super.finalize();
        System.out.println("Execute finalize() method");
        FinalizeEscapeGC.SAVE_HOOK = this;//方法区中类静态属性引用的对象,此时可以连接上GC Root
    }
    
    public static void main(String[] agrs) {
        SAVE_HOOK = new FinalizeEscapeGC();
        
        SAVE_HOOK = null;
        System.gc();
        
        if (SAVE_HOOK != null) {
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("The Object is really dead")
        }
        
        //对象逃脱GC后再次进入GC队列，此时就无法逃脱了
        SAVE_HOOK = null;
        System.gc();
        
        if (SAVE_HOOK != null) {
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("The Object is really dead")
        }
    }
}
```



#### 标记-清除算法

标记-清除算法（Mark-Sweep）如同名字一样，在回收垃圾时有“标记”和”清除“两个阶段：首先标记所有需要回收的对象，之后在标记完成后统一回收所有标记的对象。标记算法实现简单，容易理解，但是却有两个问题：

1. 效率不高，标记和清楚需要扫描大量空间，复杂度高。
2. 空间利用率低，标记清楚后会产生大量不连续的内存碎片，空间碎片太多可能会导致之后分配大对象时无法分配，从而提前出发GC。



#### 复制算法

复制（Copying）算法的出现，是为了解决Mark-Sweep的效率问题，复制算法是将可用内存分成两个相同的部分，每次只使用其中一部分，当其中一块的内存用完了，就将这块内存的存活对象复制到另一块空间上，并且将已使用过的内存空间一次清理掉。

复制算法的实现简单、运行高效，但是代价却太高了，需要牺牲一半的空间。



#### 标记-整理算法

复制算法在对象存活率较高的情况下就会进行较多的复制操作，效率将会变低。标记-整理（Mark-Compact）算法的标记过程和标记-整理的标记一致，但之后并不会直接回收对象而是让对象全都往一端排列，减少内存碎片，之后再将对象外的内存清理干净。