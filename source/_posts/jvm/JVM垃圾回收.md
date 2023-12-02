---
title: JVM垃圾回收
date: 2023-01-02 16:16:60
theme: cyanosis
highlight: atom-one-dark-reasonable
tags:
 - jvm
 - java
 - 垃圾回收
categories:
 - jvm
cover: https://img.fansqz.com/img/195425_86566869162.jpg
---
# jvm垃圾回收

所谓垃圾就是指运行程序中没有任何指针指向的对象

## 标记算法

当一个对象没有被没有被引用的时候就可以被回收。可以使用**引用计数**和**可达性分析**算法来标记一个对象为垃圾回收

### 引用计数

记录对象背引用的情况，如果其他对象引用该对象，引用计数+1；如果引用失效，引用计数-1。如果一个对象的引用计数为0，则该对象可以被回收

**优点**：

- 实现简单，判定效率高，回收没有延长性

**缺点**：

- 需要一个引用计数器，需要额外空间开销

- 每次引用都要更新计数器，增加时间开销

- 无法处理**循环引用问题**，这是一个致命缺陷，导致java不使用这类算法

### 可达性分析

基本思路就是人以根对象集合为起点，按照从上到下的方式搜索根对象集合所连接的目标对象是否可达。搜索走过的路径叫**引用链**，如果一个对象没有任何引用链相连，则是不可达的，标记为垃圾。

**注意**：由于java用的是可达性分析算法，分析工作必须在一个能保证一致性快照中进行。这也是GC为什么要Stop The World的原因

#### GC Roots

java中哪些对象可以被作为GC Roots呢

- 虚拟机栈或本地方法栈中引用的对象

- 方法区静态属性引用的对象

- 方法区中常量引用的对象

- 被同步锁`synchronized`持有的对象

- 虚拟机内部的引用，比如基本类型对应的Class对象，一些常驻的异常对象

#### finalization机制

如果所有的根节点都无法访问到某个对象，说明对象已不在使用，一般来说，该对象需要被回收。但是，其实也并不是非死不可的。他暂时处于缓刑阶段。一个无法触及的对象有可能在某个条件下“复活”自己

#### 对象的三种状态

java语言提供了对象`finalization`机制来允许开发人员提供对象被销毁前自定义处理逻辑，由于`finalize()`对象的存在，虚拟机中的对象一般处于三种可能的状态：

- 可触及的：从根节点开始，可以到达的对象

- 可复活的：对象所有引用都被释放，但是对象有可能在`finalize()`中复活

- 不可触及的：对象`finalize()`被调用，且没有复活，那么久进入不可触及状态

#### 标记过程

1. 对象到GC Roots没有引用链，则进行第一次标记
2. 经过筛选，该对象是否有必要执行finalize()方法
   - 如果没有重写`finalize()`方法，或则该方法已被调用，则判定为不可触及
   - 如果重写了`finalize()`方法，且未被执行。则将该对象添加到`F-Queue`队列中。然后由`Finalizer`线程去触发`finalize()`方法稍后`GC`会对`F-Queue`队列中的对象进行二次标记，如果该对象的`finalize()`方法与引用链上任何对象建立了联系，则在第二次标记时移出“即将回收”集合。之后，如果该对象如果又再次出现没有引用链的情况，则直接回收。



## 清除算法

### 标记-清除（Mark-Sweep）

**过程**

1. 标记：从根节点开始遍历，标记所有被引用的对象，在对象的Header中标记为可达

2. 清除：从头到未线性遍历，如果发现对象没有被标记，则清除

![image-20230101223642566](https://img.fansqz.com/img/image-20230101223642566.png)

**缺点**：

- 效率不算高

- 清除以后会导致许多内存碎片

### 复制算法（Copying）

**过程**：

将内存分为两快，每次只在一块空间上分配内存，另一块则空闲。

1. 一开始在A区分配对象，GC时将存活对象从A区复制到B区

2. 清除A区，然后下次分配对象在B区分配

![image-20230101224330741](https://img.fansqz.com/img/image-20230101224330741.png)

然后直接清除A区

**优点**：

- 没有经过标记清除过程，实现简单，运行高效

- 复制过去保证空间的连续性

**缺点**：

- 需要两倍内存空间

- 对象地址被改变，引用地址也要跟着变

- 如果垃圾很少，就会效率低，所以这就是为什么要在幸存者区使用复制算法

### 标记整理（Mark-Compact）

**过程**：

1. 和标记清除是算法一样，从根节点触发，标记所有被引用对象
2. 将所有存活对象压缩到内存的一端，按顺序排放

![image-20230101230257704](https://img.fansqz.com/img/image-20230101230257704.png)

 **优点**：

- 内存空间连续
- 不需要两倍空间

**缺点**:

- 效率较低
- 对象地址会被改变，引用地址也要改变

### 分代收集算法

当前虚拟机的垃圾收集都采用分代收集算法，这种算法没有什么新的思想，只是根据对象存活周期的不同将内存分为几块。一般将 java 堆分为新生代和老年代，这样我们就可以根据各个年代的特点选择合适的垃圾收集算法。

**比如在新生代中，每次收集都会有大量对象死去，所以可以选择”标记-复制“算法，只需要付出少量对象的复制成本就可以完成每次垃圾收集。而老年代的对象存活几率是比较高的，而且没有额外的空间对它进行分配担保，所以我们必须选择“标记-清除”或“标记-整理”算法进行垃圾收集。**





## 内存分配与回收原则

1. 首先，Eden区最大，对外提供堆内存。当 Eden 区快要满了，则进行 Minor GC，把存活对象放入 Survivor A 区，清空 Eden 区；
2. Eden区被清空后，继续对外提供堆内存；
3. 当 Eden 区再次被填满，此时对 Eden 区和 Survivor A 区同时进行 Minor GC，把存活对象放入 Survivor B 区，同时清空 Eden 区和Survivor A 区；
4. Eden区继续对外提供堆内存，并重复上述过程，即在 Eden 区填满后，把 Eden 区和某个 Survivor 区的存活对象放到另一个 Survivor 区；
5. 当某个 Survivor 区被填满，且仍有对象未被复制完毕时，或者某些对象在反复 Survive 15 次左右时，则把这部分剩余对象放到Old 区；
6. 当 Old 区也被填满时，进行 Major GC，对 Old 区进行垃圾回收。

### 对象优先在 Eden 区分配

大多数情况下，对象在新生代中 Eden 区分配。当 Eden 区没有足够空间进行分配时，虚拟机将发起一次 Minor GC。

当 Eden 区没有足够空间进行分配时，虚拟机将发起一次 Minor GC。GC 期间虚拟机又发现 分配对象无法存入 Survivor 空间，所以只好通过 **分配担保机制** 把新生代的对象提前转移到老年代中去，老年代上的空间足够存放该对象，所以不会出现 Full GC。执行 Minor GC 后，后面分配的对象如果能够存在 Eden 区的话，还是会在 Eden 区分配内存。

### 大对象直接进入老年代

大对象就是需要大量连续内存空间的对象（比如：字符串、数组）。

大对象直接进入老年代主要是为了避免为大对象分配内存时由于分配担保机制带来的复制而降低效率。

### 长期存活对象进入老年代

大部分情况，对象都会首先在 Eden 区域分配。如果对象在 Eden 出生并经过第一次 Minor GC 后仍然能够存活，并且能被 Survivor 容纳的话，将被移动到 Survivor 空间（s0 或者 s1）中，并将对象年龄设为 1(Eden 区->Survivor 区后对象的初始年龄变为 1)。

对象在 Survivor 中每熬过一次 MinorGC,年龄就增加 1 岁，当它的年龄增加到一定程度（默认为 15 岁），就会被晋升到老年代中。

### 主要进行gc的区域

针对 HotSpot VM 的实现，它里面的 GC 其实准确分类只有两大种：

部分收集 (Partial GC)：

- 新生代收集（Minor GC / Young GC）：只对新生代进行垃圾收集；
- 老年代收集（Major GC / Old GC）：只对老年代进行垃圾收集。需要注意的是 Major GC 在有的语境中也用于指代整堆收集；
- 混合收集（Mixed GC）：对整个新生代和部分老年代进行垃圾收集。

整堆收集 (Full GC)：收集整个 Java 堆和方法区。



## 软弱虚引用

### 强引用

最常见的引用，平时程序基本是强引用，平时使用的引用就是强引用

~~~java
Object a = new Object();
~~~

就是强引用，强引用对象都是可触及的。垃圾回收器永远不会回收掉被引用的对象。

### 软引用

软引用在内存不足的时候，发生OOM时会被回收

~~~java
public void softReferenceTest() {
    // 软引用，在系统要发生oom时，即使可达也会被回收
    SoftReference<StringBuffer> sf = new SoftReference<>(new StringBuffer("123456"));
    System.gc();
    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    // gc不会回收
    System.out.println(sf.get());
    try {
        // 需要设置一下堆空间 -Xms10m -Xmx10m
        byte[] b = new byte[1024 * 1024 * 1024 ];
    } catch (Throwable e) {
        e.printStackTrace();
    }
    // 输出null，被回收
    System.out.println(sf.get());
}
~~~

### 弱引用

弱引用被GC时就会被回收

~~~java
    public void weakReferenceTest() {
        // 弱引用，gc以后被回收
        WeakReference<StringBuilder> wf = new WeakReference<>(new StringBuilder("fzw 234234"));
        System.out.println(wf.get());
        System.gc();
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(wf.get());
    }
~~~

### 虚引用

有和没有一样，一般用于跟踪对象被gc

~~~java
public void phantomReferenceTest() {
    //引用队列，设置虚引用需要指定引用队列，如果对象被gc
    ReferenceQueue referenceQueue = new ReferenceQueue();
    Object object = new Object();
    PhantomReference pr = new PhantomReference<Object>(object, referenceQueue);
    // 你是无法通过虚引用获取对象的，这个引用，有和没有一样
    System.out.println(pr.get());
    object = null;
    System.gc();
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    // 如果对象被回收，就会把虚引用添加到引用队列
    Object o = null;
    try {
        o = referenceQueue.remove();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    if ( o != null) {
        System.out.println("追踪对象被gc");
    }
}
~~~

## 垃圾回收器
**GC的性能指标**

1. 吞吐量 = 运行用户代码时间 / (运行用户代码时间 + gc时间)

   在注重吞吐量的情况下，可以忍受长一点的暂停时间

2. 暂停时间：只应用程序线程暂停，让gc执行的时间

   在注重暂停时间的情况下，可以忍受低点的吞吐量

### Serial(串行回收)

- Serial收集器采用**复制算法、串行回收和“Stop-the-World"机制**方式执行内存回收
- Serial Old收集器同样也采用串行回收和"Stop-the-World"，只不过内存回收算法使用**标记-整理**算法

![20210804201210401](https://img.fansqz.com/img/20210804201210401.png)

比较适合单核CPU

### ParNew(并行回收)

- 新生代垃圾回收器

- ParNew收集器采用了**并行回收**的方式进行垃圾回收，其他方面和Serial几乎没任何区别，采用复制算法、“Stop-the-World”机制

![20210804201849588](https://img.fansqz.com/img/20210804201849588.png)

### Parallel(吞吐量优先)

- Parallel和ParNew收集器不同，Parallel目标是达到一个**可控制的吞吐量**，所以它也被称为吞吐量优先的垃圾收集器
- Parallel Old对应的老年代收集器

![20210729181127965](https://img.fansqz.com/img/20210729181127965.png)

高吞吐量可以高效利用CPU时间，尽快完成程序的运算任务。比较适合后台运算而不需要太多交互的任务。

### CMS(低延迟)

CMS(Concurrent-Mark-Sweep)

- 是一款**老年代垃圾回收器**，这款回收器是HotSpot第一款真正意义上的**并发收集器**，第一次实现垃圾收集器与用户线程同时工作

- CMS关注的点是缩短垃圾收集时用户线程停顿时间
- CMS采用**标记-清除**算法，也有“Stop-the-World”

![image-20230102012553083](https://img.fansqz.com/img/image-20230102012553083.png)

**过程**

1. 初始标记：目标标记除GC Root能直接关联到的对象，速度非常快
2. 并发标记：从GC Root直接关联的对象出发，遍历只整个对象图的过程，耗时长，但是不需要暂停用户线程
3. 重新标记：修正并发标记期间，因为用户线程运作产生变动的那一部分对象的标记记录
4. 并发清除：清除掉已死亡对象，并发执行。在这个过程中用户线程产生的**浮动垃圾**留给下一次GC

### G1(区域化分代)

G1(Garbage First)

- 目标是在延迟可控情况下情况下，尽可能提高吞吐量

- 一个并行回收器，将堆内存分割为很多个不相关的区域。使用不同的区域来表示Eden、幸存者0区，幸存者1区、老年代等
- G1 GC有一个优先列表，去记录每一个Region中垃圾堆积的价值大小，每次根据**允许的收集时间，优先回收价值最大的Region**

- 该方式侧重于回收垃圾量大的Region，所以叫：垃圾优先（Garbage First）

![image-20230102150709646](https://img.fansqz.com/img/image-20230102150709646.png)

**步骤**(粗略)：

1. 新生代GC：当eden区用尽时开始新生代回收过程；然后从新生代移动存活对象到幸存者区或老年代区。
2. 老年代并发标记过程：当堆内存使用达到一个值时，开始老年代并发标记过程
3. 混合回收：标记完成以后马上开始混合回收过程。G1 GC从老年代中移动存活对象到空闲区间，这些空闲区间也就成为了老年代的一部分。和新生代不同，**老年代不需要整个回收，只需要回收小部分老年代的Region就行**。
4. （如果需要，单线程、独占式、高强度的Full GC还是继续存在的。它针对GC的评估失败提供一种失败保护机制，即强力回收）

**RSet(Remembered Set)**

一个对象并非是孤立的，它可能被其他对象所引用。判断一个对象是否存活，是否需要遍历整个堆？

解决方法：引入Remembered Set

1. 无论G1还是其他分代收集器，jvm都使用Remembered Set
2. 每个Region对应有一个Remember Set
3. 每次Reference类型数据写操作时，都会产生一个Write Barrier暂时中断操作。然后检测写入的引用对象指向的对象和该Reference类型数据在不同区域，如果不同，通过CardTable把引用信息**记录到引用指向对象的Remembered Set**中



**借鉴与参考**

[(七)JVM成神路之GC分代篇：分代GC器、CMS收集器及YoungGC、FullGC日志剖析 - 掘金 (juejin.cn)](https://juejin.cn/post/7075258758196101156#heading-10)
https://javaguide.cn/java/jvm/jvm-garbage-collection.html#%E4%B8%BB%E8%A6%81%E8%BF%9B%E8%A1%8C-gc-%E7%9A%84%E5%8C%BA%E5%9F%9F
尚硅谷JVM