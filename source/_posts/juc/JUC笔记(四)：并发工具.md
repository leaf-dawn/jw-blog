---
title: JUC笔记(四):并发工具
date: 2022-01-27 16:11:00
theme: cyanosis
highlight: atom-one-dark-reasonable
tags:
 - juc
categories:
 - juc
cover: https://img.fansqz.com/img/wallhaven-zygeko.jpg
---
# 并发工具

## 线程池

![image-20230119151505059](https://img.fansqz.com/img/image-20230119151505059.png)



### ThreadPoolExecutor

#### 线程池状态

ThreadPoolExecutor使用int的高三位来表示线程池状态，低29为表示线程数量

| 状态名     | 高3位 | 接收新任务 | 处理阻塞队列任务 | 说明                                      |
| ---------- | ----- | ---------- | ---------------- | ----------------------------------------- |
| running    | 111   | Y          | Y                |                                           |
| shutdown   | 000   | N          | N                | 不会接收新任务，但会处理阻塞队列剩余任务  |
| stop       | 001   | N          | N                | 会中断正在执行的任务，并抛弃阻塞队列任务  |
| tidying    | 010   | -          | -                | 任务全部执行完毕，活动线程为0即将进入终结 |
| terminated | 011   | -          | -                | 终结状态                                  |

#### 构造方法

~~~java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) 
~~~

- corePoolSize：核心线程数目（最多保留的线程）
- maximumPoolSize 最大线程数目
  - 最大线程数 - 核心线程数 = 救急线程数
- keepAliveTime 生存时间 - 针对救急线程
- unit 时间单位 - 针对救急线程
- workQueue 阻塞队列
- threadFactory 线程工厂 - 可以为线程创建时起个好名字
- handler 拒绝策略

**工作方式**

1. 线程池一开始没有线程，当任务提交给线程池以后，线程池会创建一个新的线程来执行任务
2. 如果有任务过来，则从核心线程中获取一个线程去执行任务
3. 如果没有空闲的核心线程，就将任务放到阻塞队列
4. 如果阻塞队列满了，但是又来了一个任务，就会将这个任务交给救急线程
5. 如果线程达到了maximumPoolSize，还有新任务，这时就会执行拒绝策略
6. 如果救急线程执行完以后，经过生存时间都没有任务，就会被销毁

#### 工厂方法(Executors类)

**newFixedThreadPool**

创建一个固定大小的线程池

~~~java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
~~~

- corePoolSize和maximumPoolSiz相等，救急线程数为0
- 阻塞队列是无界的，可以放任意数量的任务

当任务执行时间比较长，任务数目确定的情况下可以使用

**newCachedThreadPool**

带缓冲功能的线程池

~~~java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
~~~

- corePoolSize设置为0，急救线程数设置为最大
- 全部都是救急线程（60s后可以回收）

- 使用的线程是SynchronousQueue，主要的特点是没有容量，如果没有线程来取是放不进去的。

整个线程池会根据任务量不断增长，没有上限。适合任务数密集，但是执行时间较短的情况。

**newSingleThreadExecutor**

创建单个线程的线程池

~~~java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizablFeDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
~~~

- corePoolSize和maximumPoolSize都为1，线程大小固定为一个

- 希望多个任务排队执行，如果任务对于一时，会放入无界队列中排队等待

和单线程任务区别

- 单线程任务执行失败以后没有任何不久措施，而线程池还会创建一个新的线程，保证池的正常工作

和newSingleThreadExecutor(1)的区别

- newSingleThreadExecutor()线程初始个数为1，不可修改
- 使用了FinalizablFeDelegatedExecutorService装饰器模式，只对外暴露ExecutorService的接口
- newSingleThreadExecutor(1)，创建以后还可以通过setCorePoolSize等方法进行修改

#### 常用方法

**执行**

~~~java
// 执行任务
void execute(Runnable command);
// 提交任务 task，用返回值 Future获得执行结果
<T> Future<T> submit(Callable<T> task);
// 提交tasks中所有任务
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks);
    throws InterruptedException;
// 提交tasks中所有的任务，带超时时间
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                     long timeout, TimeUnit unit);
// 提交tasks中所有任务，哪个任务先成功执行完毕，返回此任务执行结果，其他任务取消
<T> T invokeAny(Collection<? extends Callable<T>> tasks);

<T> T invokeAny(Collection<? extends Callable<T>> tasks,
                           long timeout, TimeUnit unit);
~~~

**关闭线程**

~~~java
/**
线程池状态变为shutdown
- 不会接收新任务
- 但已提交的任务会执行完
- 此方法不会阻塞调用线程的执行
*/
void shutdown();

/**
线程池状态变为 stop
- 不会接收新任务
- 会将队列中的任务返回
- 并用 interrupt 的方式中断正在执行的任务
*/
List<Runnable> shutdownNow();
~~~

#### 任务调度线程池

**Timer的使用**

Timer可以实现任务调度。但是由于所有任务都是由一个线程来调度，因此**所有任务都是串行执行的，同一个时间只能有一个任务执行，前一个任务的延迟或异常都将会影响之后的任务**。

~~~java
public static void main(String[] args) {
    Timer timer = new Timer();
    TimerTask task1 = new TimerTask() {
        @Override
        public void run() {
            System.out.println("task 1");
            try {
                Thread.sleep(2);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    };
    TimerTask task2 = new TimerTask() {
        @Override
        public void run() {
            System.out.println("task 2");
        }
    };
    timer.schedule(task1, 1000);
    timer.schedule(task2, 1000);
}
~~~

**ScheduledThreadPoolExecutor**

前面的任务并不会影响后面的任务的运行。因为可以设置多个核心线程来调度任务。

~~~java
public static void main(String[] args) {
    ScheduledExecutorService pool = Executors.newScheduledThreadPool(2);
    pool.schedule(() -> {
        System.out.println("task1");
        try {
            Thread.sleep(2);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }, 1, TimeUnit.SECONDS);
    pool.schedule(() -> {
        System.out.println("task2");
    }, 1, TimeUnit.SECONDS);
}
~~~

~~~java
/**
* command:需要定时执行的任务
* initialDelay:开始的延时时间
* period:间隔该时间循环执行任务，该时间间隔是从上一个任务开始算起的，不过还得等待上一个任务执行结束。间隔时间 = max(period，上一个任务执行时间)
*/
ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);
/**
* command:需要定时执行的任务
* initialDelay:开始的延时时间
* delay:间隔时间，该间隔时间是从上一个任务执行结束以后开始算起的。
*/
ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);
~~~



### Fork/Join

JDK1.7加入的线程池的实现，体现一种分治思想，适用于能够进行任务拆分的cpu密集型运算

任务拆分是将一个大任务拆分为算法上相同的小任务，直到无法拆分。比如说归并排序等。Fork/Join在分治的基础上加入多线程，**可以将每个任务分解和合并交给不同的线程。进一步提高效率**。

Fork/Join默认创建和cpu核心数大小相同的线程池

#### 基本使用

~~~java
public class Code {

    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool(4);
        System.out.println(pool.invoke(new MyTask(5)));
    }
}

/**
 * 无返回结果继承RecursiveAction
 * 有返回结果继承RecursiveTask
 */
// 求 1-N 之间的和
class MyTask extends RecursiveTask<Integer> {

    private int n;
    public MyTask(int n) {
        this.n = n;
    }
    @Override
    protected Integer compute() {
        if (n == 1) {
            return 1;
        }
        MyTask t1 = new MyTask(n - 1);
        t1.fork(); // 让一个线程去执行
        int a = t1.join(); // 获取任务结果
        return n + a;
    }
}
~~~



## 并发工具

### AQS

AbstractQueuedSynchronizer，是阻塞锁和相关同步器工具的框架

- 用state属性表示资源状态（分为独占模式和共享模式），子类需要定义如何维护这个状态，控制如何获取和释放锁
  - 独占模式为只有一个线程能访问资源，共享模式允许多个线程访问资源

- 提供了基于FIFO的等待队列
- 条件变量来实现等待、唤醒机制，支持多个条件变量

子类需要实现一些方法

- tryAcquire
- tryRelease
- tryAcqireShared
- tryReleaseShared
- isHeldExclusively

#### 自定义锁

~~~java
// 自定义锁（不可重入锁）
class MyLock implements Lock {

    //独占锁
    class MySync extends AbstractQueuedSynchronizer {
        // 尝试获取锁
        protected  boolean tryAcquire(int arg) {
            // 将状态从0改为1即加锁
            if (compareAndSetState(0, 1)) {
                // 加锁并设置owner为当前线程
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        //尝试释放锁
        protected boolean tryRelease(int arg) {
            // 设置owner线程为null，设置锁状态为0
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        // 是否持有独占锁
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        public Condition newCondition() {
            return new ConditionObject();
        }
    }

    private MySync sync = new MySync();
    @Override
    public void lock() {
        // 里面会调用tryAcquire,如果不成功会放到阻塞队列等待
        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    // 加锁
    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    // 带超时加锁
    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    // 解锁
    @Override
    public void unlock() {
        sync.release(1);
    }

    // 创建条件变量
    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}
~~~



### 读写锁

#### ReentrantReadWriteLock

支持重入的读写锁，如果读操作远远高于写操作时，这时候使用**读写锁**可以让**读-读**并发，提高性能。

~~~JAVA
class Test {
    private Object data = new Object();
    private ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
    private ReentrantReadWriteLock.ReadLock r  = rw.readLock();
    private ReentrantReadWriteLock.WriteLock w = rw.writeLock();
    
    public Object read() {
        r.lock();
        try {
            return data;
        } finally {
            r.unlock();
        }
    }
    
    public void write(Object obj) {
        w.lock();
        try {
            data = obj;
        } finally {
            w.unlock();
        }
    }
}
~~~

- 读锁不支持条件变量
- 重入时升级不支持：持有读锁情况下去获取写锁，会导致获取写锁永久等待
- 重入时降级支持：持有写锁下去获取读锁



#### StampedLock

jdk8加入的，为了进一步优化读新能，特点时在使用读锁、写锁时都必须配合【戳】使用。

~~~java
// 读锁
long stamp = lock.readLock();
lock.unlockRead(stamp);
// 写锁
long stamp = lock.writeLock();
lock.unlockWrite(stamp);
~~~

乐观读，StampedLock支持tryOptimisticRead()方法（乐观读），读取完毕后需要做一次【戳】校验，校验通过则说明这期间没有写操作，数据安全。如果校验没有通过，需要重新获取锁，保证数据安全

~~~java
long stamp = lock.tryOptimisticRead();
// 读取数据
// 验戳
if (lock.validate(stamp)) {
    // 搓有效，返回数据
}
// 无效，获取读锁
~~~



### Semaphore

信号量，用来限制访问共享资源线程上限

~~~java
Semaphore s = new Semaphore(3); //限制资源上限为3
for (int i = 0; i < 10; i++) {
    new Thread(() -> {
        try {
            s.acquire(); // 获取信号量
            System.out.println("线程:" + Thread.currentThread().getName());
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            s.release(); // 释放信号量
        }
    }).start();
}
~~~

### CountdownLatch

倒计时锁，用来进行线程同步协作，等待所有线程完成倒计时。

其中构造参数用来初始化等待计数值，await() 用来等待计数归零，countDown() 用来让计数一

~~~java
CountDownLatch latch = new CountDownLatch(3);
new Thread(() -> {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
    System.out.println("线程一执行结束");
}).start();
new Thread(() -> {
    try {
        Thread.sleep(2000);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
    System.out.println("线程二执行结束");
}).start();
new Thread(() -> {
    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
    System.out.println("线程三执行结束");
}).start();
System.out.println("等待");
latch.await();
System.out.println("等待结束");
~~~



### CyclicBarrier

循环栅栏，用于线程协作，等待线程满足某个计数。当等待线程满足【计数个数】时，继续执行。CyclicBarrier不像CountdownLatch，它是**可以恢复计数**的。

~~~java
ExecutorService service = Executors.newFixedThreadPool(2);
CyclicBarrier barrier = new CyclicBarrier(2, () -> {
    // 等到计数为 0 开始执行
    System.out.println("线程三开始执行");
});
service.submit(() -> {
    System.out.println("线程一启动");
    try {
        Thread.sleep(1000);
        barrier.await(); // 计数 - 1
        System.out.println("线程二继续执行");
    } catch (InterruptedException | BrokenBarrierException e) {
        throw new RuntimeException(e);
    }
});
service.submit(() -> {
    System.out.println("线程二启动");
    try {
        Thread.sleep(2000);
        barrier.await(); // 计数继续 - 1
        System.out.println("线程二继续执行");
    } catch (InterruptedException | BrokenBarrierException e) {
        throw new RuntimeException(e);
    }
});
service.shutdown();
~~~

- CyclicBarrier为0以后，会恢复计数，可以重用



### 线程安全的集合类

线程安全集合类可以分为三大类

- 遗留的线程安全集合：Hashtable（线程安全的map），Vector（线程安全的list），由于锁的粒度大，现在有别的集合类来取代

- 使用Collections修饰的线程安全集合

  - Collections.synchronizedCollection
  - Collections.synchronizedList 等，使用装饰器设计模式，给集合每个方法加上synchronized

  ~~~java
  List<Integer> list = Collections.synchronizedList(new ArrayList<>());
  ~~~

- JUC线程安全集合

  - Blocking类，大部分实现基于锁，并提供用来阻塞的方法

  - CopyOnWrite类，通过修改时拷贝的方式来避免并发安全，适用于读多写少的情况

  - Concurrent类

    - 内部很多操作使用了cas进行优化
    - 弱一致性
      - 遍历时的弱一致性，比如当迭代器遍历时，如果容器发生修改，迭代器仍然可以继续遍历，这时内容是旧的
      - 求大小时，size不是百分之百准确
      - 读取弱一致性

    > 遍历时发生了修改，对于非安全的集合来讲，使用**fail-fast**机制让遍历立刻失败，抛出ConcurrentModificationException，不再继续遍历

#### CopyOnWriteArrayList

底层实现采用了**写入拷贝**思想，增删改操作会将底层数组拷贝一份，更改操作在新数组上执行，不会影响其他线程并发读。

~~~java
// add方法源码
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+len);
        Object[] newElements;
        int numMoved = len - index;
        if (numMoved == 0)
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            newElements = new Object[len + 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index, newElements, index + 1,
                             numMoved);
        }
        newElements[index] = element;
        setArray(newElements);
    } finally {
        lock.unlock();
    }
}
~~~

