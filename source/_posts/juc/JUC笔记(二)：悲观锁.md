---
title: JUC笔记(二):悲观锁
date: 2022-01-15 16:43:00
theme: cyanosis
highlight: atom-one-dark-reasonable
tags:
 - juc
categories:
 - juc
cover: https://img.fansqz.com/img/16720_2403907660.jpg
---
# JUC笔记(二):悲观锁

## 共享变量带来的问题

举一个简单的例子：

- 线程一和线程二都需要去改变一个静态变量`i = 0`
- 线程一将 `i++`
- 线程二将 `i--`

由于`i++`和`i-`-都并非原子操作，在编译成字节码以后会拆分成多个人字节码指令:

- `i++`

~~~java
getstatic i    //获取静态变量i
iconst_1       //准备常量1
iadd           //自增
putstatic i    //修改以后存入静态变量i
~~~

- `i--`

~~~java
getstatic i    //获取静态变量i
iconst_1       //准备常量1
isub           //自减
putstatic i    //修改以后存入静态变量i
~~~

由于线程一和线程二并发执行，CPU会切换不同的线程，这导致了，CPU运行时的真实情况并不一定是先执行完线程一的i++的所有指令再执行线程二的i--，而是可能会出现很多种情况，如下。最终 i 的结果为 -1，与预期的不同，出现了并发问题

![image-20230113161508712](https://img.fansqz.com/img/image-20230113161508712.png)

### 相关概念

**临界区**：一段代码块内如果存在对**共享资源**的多线程读写操作，称这块代码块为临界区。

**竞态条件**：多个线程在**临界区**内执行，由于代码的**执行序列不同**而导致结果无法预测，称之为发生了竞态条件。

## synchronized

synchronized【对象锁】，采用互斥的方式让同一时刻至多只有一个线程能持有【对象锁】。其他线程再想获得这个【对象锁】时就会阻塞住。

### 同步代码块

**语法**

~~~java
synchronized(对象) {
    临界区
}
~~~

**例子**

~~~java
static int counter = 0;
static final Object object = new Object();
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(()->{
        synchronized (object) {
            counter++;
        }
    },"t1");
    Thread t2 = new Thread(()->{
        synchronized (object) {
            counter--;
        }
    },"t2");
    t1.start();
    t2.start();
}
~~~

### 加在方法上

synchronized加在方法上，本质上是锁住this

**语法**

~~~java
class Test {
    public synchronized void test() {
        
    }
}
//等价于
class Test{
    public void test() {
        synchronized(this) {
            
        }
    }
}
~~~

**例子**

~~~java
class Room {
    private int counter = 0;
    public synchronized void increment() {
        counter++;
    }
    public synchronized void decrement() {
        counter--;
    }
    public synchronized int getCounter() {
        return counter;
    }
}
~~~

### Monitor工作原理

Monitor 被翻译为监视器或管理。

每个java对象都可以关联一个Monitor对象，如果使用synchronized给对象上锁后，该对象头的Mark Word就被设置为指向 Monitor对象的指针

Monitor 结构如下

![image-20230113195512552](https://img.fansqz.com/img/image-20230113195512552.png)

- 刚开始Monitor中的Owner为null
- 当Thread-2执行synchronized(obj) 以后就会将Monitor的所有者设置为Thread-2，Monitor中只有一个Owner
- 在Thread-2上锁的过程中，如果Thread-3，Thread-4，Thread5也来执行synchronized(obj)，就会进入EntryList中

- Thread-2执行完同步代码块内容，然后唤醒EntryList中的等待的线程来竞争锁，竞争锁时是非公平的



其中Mark Word的结构为：

![image-20230113224422906](https://img.fansqz.com/img/image-20230113224422906.png)



## wait notify

### 使用

**API**

- obj.wait() 让进入 object 监视器的线程到 waitSet 的等待
- obj.notify() 在 object 上正在 waitSet等待的线程中挑一个唤醒
- obj.notifyAll() 让 object 上正在 waitSet 等待的线程全部唤醒

他们都是线程之间协作的手段，都属于Object对象的方法。**必须获得此对象的锁，才能调用这些方法**



**例子**

~~~java
static final Object object = new Object();
public static void main(String[] args) throws InterruptedException {
    new Thread(()->{
        synchronized (object) {
            System.out.println("执行...");
            try {
                object.wait();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println("执行其他代码...");
        }
    }, "t1").start();
    new Thread(()->{
        synchronized (object) {
            System.out.println("执行...");
            try {
                object.wait();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println("执行其他代码...");
        }
    }, "t2").start();

    //等待一秒以后，挑选一个睡眠中的线程唤醒。
    Thread.sleep(1000);
    synchronized (object) {
        object.notify();
    }
}
~~~

### 原理

![image-20230113225342364](https://img.fansqz.com/img/image-20230113225342364.png)

- Owner线程调用wait方法，即可进入WaitSet变为WAITING状态。
- BLOCKED和WAITING的线程都处于阻塞状态，不占用CPU时间片
- BLOCKED 线程会在 Owner 线程释放时唤醒
- WAITING 线程会在 Owner 线程调用notify 或 notify All时唤醒，但唤醒后并不意味着立即获得锁，仍需进入 EntryList重新竞争



### sleep(long n)和wait(long n)的区别

1. sleep时Thread方法，而wait是Object方法
2. sleep不需要强制和synchronized配合使用，但wait需要和synchronized配合使用
3. sleep在睡眠的同时，不会释放对象锁，但wait在等待的时候会释放对象锁

4. 相同点：他们的状态都是Time_Waiting，有时限的等待





## park unpark

- park停止线程，unpark开启

### 使用

先通过park停止线程，然后通过unpark恢复线程使用

~~~java
//暂停当前线程
LockSupport.park();
//恢复某个线程对象
LockSupport.park(暂停线程对象);
~~~

例：

~~~java
Thread t1 = new Thread(() -> {
    System.out.println("暂停线程");
    LockSupport.park();
    System.out.println("恢复线程");
}, "t1");
t1.start();
Thread.sleep(1000);
LockSupport.unpark(t1);
~~~

注意：unpark()可以在park()之前执行来解除park()



## ReentrantLock

- 可中断
- 可以设置超时时间，如果一段时间无法获取到锁，就放弃竞争锁
- 可以设置为公平锁
- 支持多个条件变量
- 与synchronized一样都是可重入的

### 使用

#### 基本语法

~~~java
// 获取锁
reentrantLock.lock();
// 释放锁
reentrantLock.unlock();
~~~

#### 可打断

`reentrantLock.lock()`是无法被打断的，`reentrantLock.lockInterruptibly()`可以被打断z

~~~java
private static ReentrantLock lock = new ReentrantLock();

public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(()->{
        System.out.println("尝试获得锁");
        try {
            lock.lockInterruptibly();
        } catch (InterruptedException e) {
            e.printStackTrace();
            System.out.println("被打断，没有获得锁");
            return;
        }
        try {
            System.out.println("获得锁");
        } finally {
            lock.unlock();
        }
    }, "t1");
    lock.lock();
    t1.start();
    Thread.sleep(1000);
    //打断t1
    t1.interrupt();
}
~~~

#### 锁超时

通过`reentrantLock.tryLock()`来尝试获取锁。也可以传递一个时间，通过`reentrantLock.tryLock(1, TimeUnit.SEOCNDS)`来等待一定时间，如果在这段时间内都没有获取到锁，则返回false，获得锁返回true。可避免死锁

~~~java
private static ReentrantLock lock = new ReentrantLock();

public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(()->{
        System.out.println("尝试获取锁");
        if (!lock.tryLock()) {
            System.out.println("获取不到锁");
            return;
        }
        try {
            System.out.println("获取到锁");
        } finally {
            lock.unlock();
        }
    }, "t1");
    lock.lock();
    t1.start();
}
~~~

#### 公平锁

**非公平锁**：对于`synchronized`，如果一个线程持有锁，其他线程就会进入阻塞队列。等到该线程释放锁，在阻塞队列的线程就会一拥而上去争夺锁，谁先抢到谁就拥有锁。并不会按照阻塞队列的顺序获取锁。

**公平锁**：按照阻塞队列，谁先进去的，谁就先获得锁。

`ReentrantLock`默认是非公平锁，通过`RenntrantLock lock = new RentrantLock(true)`设置为公平锁。实际上，公平锁一般没有必要，会降低并发度。

#### 条件变量

和`synchronized`中也有条件变量，就是`waitSet`休息室，当条件不满足的时候，进入`waitSet`等待。

通过`Condition condition = lock.newCondition()`来创建条件变量，如果条件不满足则调用`condition.await()`来停止线程，如果条件满足了调用`condition.signal()`来唤醒一个线程，通过`condition.signalAll()`唤醒所有线程

~~~java
private static ReentrantLock lock = new ReentrantLock();
private static Condition condition = lock.newCondition();
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(()->{
        try{
            lock.lock();
            //如果条件不满足，则等待
            try {
                System.out.println("条件不满足，等待");
                condition.await();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println("条件满足,继续干活");
        } finally {
            lock.unlock();
        }
    }, "t1");
    t1.start();
    Thread.sleep(1000);
    //条件满足，唤醒一个等待的线程
    lock.lock();
    condition.signal();
    lock.unlock();
}
~~~