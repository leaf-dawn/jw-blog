---
title: JUC笔记(三):无锁
date: 2022-01-19 11:12:00
theme: cyanosis
highlight: atom-one-dark-reasonable
tags:
 - juc
categories:
 - juc
cover: https://img.fansqz.com/img/wallhaven-j3m8y5.png
---
# 无锁
## CAS原子操作

系统提供一个原子操作CAS(compare and swap 或 Compare And Set)比较并交换，需要传递三个值，数据所在地址，数据过去的值，数据需要更新的值。

cpu会先比较传入数据原来的值是否和内存中当前的值是否相等，如果相等才更新内存中的值，否则操作失败。
使用`cas(&i, i, i + 1)`替代获取锁，那么如果操作失败了，那就循环重新再来一次，直到成功`while(! cas(&i,i, i+1))`;

### ABA


CAS会出现的一个经典问题就是ABA了，比如说：

- 一个线程1获取到内存中a的值为A
- 然后线程2把a改为了B
- 然后线程2又把a改为了A
- 最后线程A再调用CAS，它以为a变量没有被改过。执行成功


解决方法：版本号

- 对不仅需要传递值，还需要传递版本号，只有值相等且版本号相等才认为该值没有被修改过

##  原子类

### 原子整数

- AtomicBoolean
- AtomicInteger
- AtomicLong

~~~java
// AtomicInteger
AtomicInteger integer = new AtomicInteger();
// 获取，然后自增1
integer.getAndIncrement();
// 自增1，然后获取
integer.incrementAndGet();
// 获取，然后自减1
integer.getAndDecrement();
// 自减1，然后获取
integer.decrementAndGet();
// 获取，然后增加一个数值
integer.getAndAdd(10);
// 先增加一个数，然后再获取
integer.addAndGet(10);
// 更新，然后获取
integer.updateAndGet(x -> x*10);
// 获取，然后更新
integer.getAndUpdate(x -> x*10);
// 获取
integer.get();
~~~

- 使用的原理就是CAS

~~~java
// updateAndGet源码
public final int updateAndGet(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        // 获取原始值
        prev = get();
        // 计算更新以后的值
        next = updateFunction.applyAsInt(prev); 
        // CAS
    } while (!compareAndSet(prev, next));
    return next;
}
~~~

### 原子引用

- AtomicReference
- AtomicMarkableReference：有版本号，可以解决ABA问题
- AtomicStampedReference

**AtomicReference**

无版本号

~~~是他java
AtomicReference<Integer> integer = new AtomicReference<>();
Integer prev = integer.get();
Integer next = prev + 1;
integer.compareAndSet(prev, next);
~~~

**AtomicStampedReference**

有版本号，可解决ABA问题

~~~java
// 参数：初始值，初始版本号
AtomicStampedReference<Integer> integer = new AtomicStampedReference<>(0,0);
Integer prev = integer.getReference();
Integer stamp = integer.getStamp();
integer.compareAndSet(prev, prev + 1, stamp, stamp + 1);
~~~

**AtomicMarkableReference**

AtomicStampedReference的简化，版本号使用一个布尔值，只有两种状态

~~~java
// 参数：初始值，初始版本号
AtomicMarkableReference<Integer> integer = new AtomicMarkableReference<>(0,true);
Integer prev = integer.getReference();
integer.compareAndSet(prev, prev + 2, true, false);
~~~

### 原子数组

- AtomicIntegerArray
- AtomicLongArray
- AtomicReferenceArray

~~~java
//简单使用，基本差不多
AtomicIntegerArray array = new AtomicIntegerArray(10);
int prev = array.get(0);
array.compareAndSet(0, prev, prev + 1);
array.getAndAdd(0, 100);
~~~

### 字段更新器

- AtomicReferenceFieldUpdater
- AtomicIntegerFieldUpdater
- AtomicLongFieldUpdater

~~~java
public class Code {
    public static void main(String[] args) {
        Student student = new Student();
        AtomicReferenceFieldUpdater fieldUpdater = AtomicReferenceFieldUpdater.newUpdater(
                Student.class,String.class, "name"
        );
        fieldUpdater.compareAndSet(student, null, "fzw");
    }
}
class Student {
    // 一定要使用volatile修饰
    volatile String name;
}
~~~

### 原子累加器

- LongAccumulator
- LongAdder

~~~java
LongAdder longAdder = new LongAdder();
///
longAdder.add(100);
longAdder.increment();
System.out.println(longAdder.longValue());
~~~

基本原理：当存在竞争的时候，设置多个累加单元，线程一累加Cell[0]，线程二累加Cell[1]，最后将结果汇总。性能更佳。

