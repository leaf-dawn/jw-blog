---
title: JUC笔记(一):线程基础
date: 2022-01-13 15:23:00
theme: cyanosis
highlight: atom-one-dark-reasonable
tags:
 - juc
 - 线程
categories:
 - juc
cover: https://img.fansqz.com/img/2205_51708225.jpg
---
# JUC笔记(一):线程基础

## 创建线程

**方法一：直接使用Thread**

~~~java
Thead t = new Tread() {
    @Override
    public void run() {
        //执行的任务
    }
};
t.start();
~~~

**方法二：使用runnable**

~~~java
Runnable runnable = new Runnable() {
    public void run() {
        //要执行的任务
    }
};
// Thread t = new Thread(runnable, "线程名称");
Thread t = new Thread(runnable);
//启动线程
t.start();
//简化
new Thread(() -> {
    //要执行的任务
}).start();
~~~

简单的源码解析

~~~java
public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize) {
    init(g, target, name, stackSize, null, true);
}
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
    //..............
    this.target = target;
    //...............
}
@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
~~~

其实就是将runnable传入Thread中的时候赋值个target，当Thread调用run方法的时候就会调用target的run方法。

**方法三：通过FutureTask**

~~~java
Callable<Integer> callable = new Callable<Integer>() {
    @Override
    public Integer call() throws Exception {
        //需要执行的任务
        return 100; 
    }
};
FutureTask<Integer> futureTask = new FutureTask<>(callable);
Thread thread = new Thread(futureTask);
thread.start();
//可以获取结果，阻塞等待
Integer result = futureTask.get();
~~~

- FutrueTask可以通过get获取线程方法执行的结果，阻塞。

## 常见方法

| 方法                 | 作用                                        |
| -------------------- | ------------------------------------------- |
| join()               | 等待线程运行结束                            |
| join(long n)         | 等待线程运行结束，最多等待n毫秒             |
| getId()              | 获取线程长整型id                            |
| getName(String name) | 获取线程名称                                |
| getPriority()        | 获取线程优先级                              |
| setPriority(int)     | 修改线程优先级                              |
| getState()           | 获取线程状态                                |
| isInterrupted()      | 判断是否被打断（不会清除打断标记）          |
| isAlive()            | 线程是否存活（还没有运行完毕）              |
| interrupt()          | 打断线程                                    |
| interrupted()        | 判断当前线程是否被打断（会清除打断标记）    |
| currentThread()      | 获取当前正在执行的线程                      |
| sleep(long n)        | 让当前执行的线程休眠n毫秒，休眠时让出时间片 |
| yield()              | 提示线程调度器让出当前线程对CPU的使用       |

### sleep和yield

**sleep**

- 让当前线程从Running进入Timed Waiting状态

- 其他线程可以使用interrupt方法打断正在睡眠的线程，这时sleep方法会抛出InterruptedException

  ~~~java
  Thread t1 = new Thread("t1") {
      @Override
      public void run() {
          System.out.println("enter sleep...");
          try{
              Thread.sleep(2000);
          } catch (InterruptedException e) {
              System.out.println("wake up...");
              throw new RuntimeException(e);
          }
      }
  };
  t1.start();
  Thread.sleep(1000);
  //唤醒t1
  System.out.println("interrupt...");
  t1.interrupt();
  ~~~

- 休眠结束以后的线程未必会立即得到执行。

**yield**

- 调用yield会让当前线程从Running进入Runnable状态，然后调度执行其他线程。

### 线程优先级

- 最小优先级`MIN_PRIORITY = 1`，默认优先级`NORM_PRIORITY = 5`，最大优先级`MAX_PRIORITY = 10`

- 线程优先即会提示（hint)调度器优先调度该线程，但是他仅仅是一个提示，调度器可以忽略它
- 如果CPU比较忙，那么优先级高的线程会获得更多的时间片，如果CPU空闲，那么优先级基本没作用

~~~java
Thread t1 = new Thread(()->{
    int count = 0;
    for (;;) {
        System.out.println("---->1: " + count++);
    }
}, "t1");
Thread t2 = new Thread(()->{
    int count = 0;
    for (;;) {
        System.out.println("---------->2: " + count++);
    }
}, "t2");
t1.setPriority(Thread.MIN_PRIORITY);
t2.setPriority(Thread.MAX_PRIORITY);
t1.start();
t2.start();
~~~

好像没什么区别。不知为哈，设置了优先级，t2并没有运行得更多:disappointed_relieved:

### join

用于等待一个线程执行结束。想象一个场景，主线程需要获取子线程的执行结果。如果不做特殊操作，主线程不会等待子线程执行，而是直接执行结束。使用sleep方法也不行，因为主线程并不知道子线程何时执行结束。这个时候就可以尝试使用join

~~~java
static int r = 0;
//
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(()->{
        try {
            Thread.sleep(1000);
            r = 100;
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }, "t1");
    t1.start();
    //同步等待t1执行结束
    t1.join();
    System.out.println(r);
}
~~~



### interrupt

打断sleep，wait，join的线程，打断这些正在阻塞的线程，当然也可以打断正在运行的线程。

#### 打断sleep，wait，join的线程

~~~java
//try一try
Thread t1 = new Thread(()->{
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
}, "t1");
t1.start();
Thread.sleep(500);
//等待t1执行结束
System.out.println("interrupt...");
t1.interrupt();
System.out.println("打断标记：" + t1.inInterrupted());
~~~

会有一个打断标记，如果一个线程被打断的话，就会有被设置一个打断标记

- 对sleep，wait，join的线程（以异常的方式表示被打断）被打断以后，打断标记会被清除

#### 打断正常运行的线程

会设置打断标记，至于是否要打断，由被打断的线程决定

~~~java
Thread t1 = new Thread(()->{
    while(true) {
        Thread currentThread = Thread.currentThread();
        if (currentThread.isInterrupted()) {
            System.out.println("线程被打断");
            break;
        }
    }
},"t1");
t1.start();
Thread.sleep(1000);
t1.interrupt();
~~~

#### 设计模式-两阶段终止

使用stop()方法停止线程的危害

- stop方法会真正杀死线程，如果这时线程锁住了共享资源，那么它被杀死以后就没机会释放锁了。

两阶段终止（下面是一个监控线程的例子）

![image-20230112235106722](https://img.fansqz.com/img/image-20230112235106722.png)

~~~java
private Thread monitor;
public void start() {
    monitor = new Thread(()->{
        while(true) {
            Thread current =Thread.currentThread();
            if (current.isInterrupted()) {
                System.out.println("处理后事");
                break;
            }
            try {
                Thread.sleep(1000);
                System.out.println("执行监控记录");
            } catch (InterruptedException e) {
                e.printStackTrace();
                current.interrupt();
            }
        }
    });
}

public void stop() {
    monitor.interrupt();
}
~~~



#### 打断park

线程执行`LockSupport.park();`以后会停止运行。调用线程的`interrupt()`可以打断处于park的线程，让线程继续执行。

~~~java
Thread t1 = new Thread(()->{
    System.out.println("park...");
    LockSupport.park();
    System.out.println("unpark....");
});
t1.start();
Thread.sleep(1000);
t1.interrupt();
~~~



## 线程状态

### 从操作系统层面（五种）

![image-20230113145948831](https://img.fansqz.com/img/image-20230113145948831.png)

- 初始状态：仅仅在语言层面创建了线程对象，还未与操作系统关联
- 可运行状态（就绪状态）：指线程已经被创建，并在就绪队列里，等待CPU调度
- 运行状态：获取了CPU时间片，正在运行
- 阻塞状态：调用了阻塞的API，使得线程处于阻塞状态
- 终止状态：线程已经执行完毕，生命周期已经结束

### 从java层面（六种）

![image-20230113150524852](https://img.fansqz.com/img/image-20230113150524852.png)

- new：线程刚被创建，没有start
- runnable:
  - 请求IO等操作使得java线程处于阻塞状态时，在java层面也叫RUNNABLE
  - 运行状态
  - 可运行状态在java层面也时RUNNABLE

- terminated：结束状态
- blocked：等待锁导致了阻塞
- time_waiting：通过待用join或sleep等有时限的等待
- waiting：调用join等无时限制的等待