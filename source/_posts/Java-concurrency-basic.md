---
title: Java 线程基础
date: 2018-01-20 09:15:12
tags: Java Concurrency
categories: Java
---
{% blockquote Wikipedia https://en.wikipedia.org/wiki/Concurrency_(computer_science) Concurrency (computer science) %}
In computer science, concurrency refers to the ability of different parts or units of a program, algorithm, or problem to be executed out-of-order or in partial order, without affecting the final outcome. This allows for parallel execution of the concurrent units, which can significantly improve overall speed of the execution in multi-processor and multi-core systems. In more technical terms, concurrency refers to the decomposability property of a program, algorithm, or problem into order-independent or partially-ordered components or units.
{% endblockquote %}

编写正确的程序很难，而编写正确的并发程序则难上加难。线程是 Java 语言不可或缺的重要功能，它们使得复杂的异步代码变得更简单，从而极大地简化了复杂系统的开发。本文主要介绍了线程和线程池的相关知识。

## 线程和进程

线程和进程的区别是什么？

简单的说，进程是具有一定独立功能的程序关于某个数据集合上的一次运行活动，是操作系统进行资源分配和调度的一个独立单位；线程是进程的一个实体，是 CPU 调度和分派的基本单位，是比进程更小的能独立运行的基本单位。线程的划分尺度小于进程，这使得多线程程序的并发性高；进程在执行时通常拥有独立的内存单元，而线程之间可以共享内存。使用多线程的编程通常能够带来更好的性能和用户体验，但是多线程的程序对于其他程序是不友好的，因为它可能占用了更多的 CPU 资源。当然，也不是线程越多，程序的性能就越好，因为线程之间的调度和切换也会浪费 CPU 时间。

<!-- more -->

## 线程的创建

Java 提供了三种创建线程的方法：

* 继承 Thread 类

MyThread 类继承自 Thread，重载 run 方法。在 main 函数中实例化一个线程对象，运行时必须调用其 **start** 方法。

```java
public class MyThread extends Thread {

    @Override
    public void run() {
        System.out.println("MyThread running");
    }

    public static void main(String[] args) {
        MyThread myThread = new MyThread();
        myThread.start();
    }
}
```

另外可以通过匿名子类的方式创建，JDK 8 还可以采用 `lambda` 表达式。

```java
    Thread thread = new Thread(){
        @Override
        public void run(){
            System.out.println("Thread Running");
        }
    };
    // Thread thread = new Thread(() -> System.out.println("Thread Running"));
    thread.start();
```

* 实现 Runnable 接口

实现 Runnable 接口的 run 方法。Java 中的继承是单继承，一个类只能有一个父类，如果继承了 Thread 类就无法再继承其他类了，显然使用 Runnable 接口更为灵活。

```java
public class MyRunnable implements Runnable {

    @Override
    public void run() {
        System.out.println("MyRunnable running");
    }

    public static void main(String[] args) {
        Thread thread = new Thread(new MyRunnable());
        thread.start();
    }
}
```

同样，可以通过匿名子类和 `lambda` 表达式的方式创建。

```java
    Runnable myRunnable = new Runnable() {
        @Override
        public void run() {
            System.out.println("MyRunnable running");
        }
    };
    // Runnable myRunnable = () -> System.out.println("MyRunnable running");
    Thread thread = new Thread(myRunnable);
    thread.start();
```

值得一提的是 Runnable 接口是这样定义的：

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

大家都知道，接口中的方法默认是 `public` 和 `abstract` 的，可为什么还要显示地声明呢？大概是版本太老的 [原因](https://stackoverflow.com/questions/3289767/why-run-method-defined-with-abstract-keyword-in-runnable-interface) 吧。

* 实现 Callable 接口

Runnable 封装一个异步运行的任务，可以把它想象成一个没有参数和返回值的异步方法。 Callable 与 Runnable 类似，但是有返回值，且能抛出异常。Callable 接口是一个参数化的类型，只有一个方法 call。

```java
@FunctionalInterface
public interface Callable<V> {
    // Computes a result, or throws an exception if unable to do so.
    V call() throws Exception;
}
```

Future 用于保存异步计算的结果。FutureTask 包装器是一个非常便利的机制，可将 Callable 转换成 Future 和 Runnable。下面是一个例子，注意 Thread 的构造函数不能传入 Callable。

```java
import java.util.concurrent.*;

public class CallableExample {
    public static void main(String[] args) throws Exception {
        Callable<String> callable = () -> {
            // Perform some computation
            System.out.println("Entered Callable");
            TimeUnit.SECONDS.sleep(2);
            return "Hello from Callable";
        };

        // 包装成 Runnable
        FutureTask<String> futureTask = new FutureTask<>(callable);
        Thread thread = new Thread(futureTask);
        thread.start();

        System.out.println("Do something else while callable is getting executed");

        System.out.println(futureTask.get());


    }

}
```

PS：线程休眠推荐使用 TimeUnit 类的 sleep 方法，这样 [可读性更强](https://stackoverflow.com/questions/9587673/thread-sleep-vs-timeunit-seconds-sleep)。

线程启动为什么不使用 run 而是 start 方法呢？看下 Thread 类 start 方法的源码：

```java
    public synchronized void start() {
        // 线程必须是 NEW 状态，多次调用 start 会抛出异常
        if (threadStatus != 0)
            throw new IllegalThreadStateException();
        group.add(this);

        boolean started = false;
        try {
            start0(); // 启动线程，运行 run 方法
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }
    // 本地方法
    private native void start0();
```

这里的关键是本地方法 start0，它实现了启动线程、申请栈内存、运行 run 方法、修改线程状态等职责，线程管理和占内存管理都是由 JVM 负责的。启动一个线程是调用 start 方法，使线程处于可运行状态，这意味着它可以由 JVM 调度并执行，但并不是说线程就会立即运行。run 方法是线程启动后要进行回调（callback）的方法。

关于未捕获异常处理器：可以用 setUncaughtExceptionHandler 方法为任何线程安装一个处理器。也可以用 Thread 类的静态方法 setDefaultUncaughtExceptionHandler 为所有线程安装一个默认的处理器。

## 线程的生命周期

![](http://img.blog.csdn.net/20150408002007838)

Java 语言定义了 5 （或者说是 6）种线程状态，在任意一个时间点，一个线程有且只有其中的一种状态。这几种状态的定义在 Thread 类中 State 枚举类中：

```java
public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED;
}
```

* 新建（NEW）：创建后尚未启动的线程处于这种状态。

* 运行（RUNNABLE）：Runnable 包括了操作系统线程状态中的 Running 和 Ready，也就是处于此状态的线程有可能正在执行，也有可能正在等待 CPU 为它分配执行时间。

* 无限期等待（WAITING）：处于这个状态的线程不会被分配 CPU 执行时间，它们要等待被其他线程显示地唤醒。以下方法会让线程陷入无限期的等待状态：
    * 没有设置 timeout 参数的 Object.wait() 方法。
    * 没有设置 timeout 参数的 Thread.join() 方法。
    * LockSupport.park() 方法。

* 限期等待（TIMED_WAITING）：处于这种状态的线程也不会被分配 CPU 执行时间，不过无须等待被其他线程显示地唤醒，在一定时间之后他们会由系统自动唤醒。以下方法会让线程进入限期等待状态：
    * Thread.sleep() 方法。
    * 设置了 timeout 参数的 Object.wait() 方法。
    * 设置了 timeout 参数的 Thread.join() 方法。
    * LockSupport.parkNanos() 方法。
    * LockSupport.parkUntil() 方法。

* 阻塞（BLOCKED）：线程被阻塞了，“阻塞状态” 与 “等待状态” 的区别是：“阻塞状态” 在等待着获取一个排它锁，这个时间将在另外一个线程放弃这个锁的时候发生；而 “等待状态” 则是在等待一段时间。或者唤醒动作的发生。在程序等待进入同步区域的时候，线程将进入这个状态。

* 结束（TERMINATED）：已经终止的线程状态，线程已经结束执行。

### yield sleep join

```java
public static native void yield();
public static native void sleep(long millis) throws InterruptedException;
public final void join() throws InterruptedException
```

问：线程的 sleep 方法和 yield 方法有什么区别？

1. sleep 方法给其他线程运行机会时不考虑线程的优先级，因此会给低优先级的线程以运行的机会；yield 方法只会给相同优先级或更高优先级的线程以运行的机会；  
2. 线程执行 sleep 方法后转入阻塞（BLOCKED）状态，而执行 yield 方法后转入就绪（Ready）状态；  
3. sleep 方法声明抛出 InterruptedException，而 yield 方法没有声明任何异常；  
4. sleep 方法比 yield 方法（跟操作系统 CPU 调度相关）具有更好的可移植性。

问：Thread 类的 sleep 方法和对象的 wait 方法都可以让线程暂停执行，它们有什么区别?

sleep 方法是 Thread 类的静态方法，调用此方法会让当前线程暂停执行指定的时间，将执行机会让给其他线程，但是对象的锁依然保持，因此休眠时间结束后会自动恢复（线程回到就绪状态）。wait 是 Object 类的方法，调用对象的 wait 方法导致当前线程放弃对象的锁（线程暂停执行），进入对象的等待池（wait pool），只有调用对象的 notify/notifyAll 方法时才能唤醒等待池中的线程进入等锁池（lock pool），如果线程重新获得对象的锁就可以进入就绪状态。

而 join 方法会使当前线程等待调用 join 方法的线程结束后才能继续执行。注意该方法是 Thread 类对象实例的方法，也需要捕获 InterruptedException。

比如在主线程中调用了线程 t 的 join 方法，直到线程 t 执行完毕后，才会继续执行主线程。有的同学看了源码会问，`t.join()` 不是让线程 t 等待吗，为什么会是主线程？其实，wait 方法的作用是让 “当前线程” 等待，而这里的 “当前线程” 是指当前在 CPU 上运行的线程。所以，虽然是调用线程 t 的 wait 方法，但是它是通过主线程去调用的，主线程需要等待！

```java
    public final void join() throws InterruptedException {
        join(0);
    }

    public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }
        // timeout 为 0，无限期等待
        if (millis == 0) {
            while (isAlive()) {
                wait(0); // 当前线程等待
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay); // 当前线程等待
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

更多关于这三个方法的区别请参考 [这里](https://www.geeksforgeeks.org/java-concurrency-yield-sleep-and-join-methods/)。

### 线程中断

Java 中断机制是一种协作机制，也就是说通过中断并不能直接终止另一个线程，而需要被中断的线程自己处理中断。

Java 中断模型也是这么简单，每个线程对象里都有一个 boolean 类型的标识（不一定就要是 Thread 类的字段，实际上也的确不是，这几个方法最终都是通过 native 方法来完成的），代表着是否有中断请求（该请求可以来自所有线程，包括被中断的线程本身）。例如，当线程 t1 想中断线程 t2，只需要在线程 t1 中将线程 t2 对象的中断标识置为 true，然后线程 t2 可以选择在合适的时候处理该中断请求，甚至可以不理会该请求，就像这个线程没有被中断一样。

Thread 类提供了几个方法来操作这个中断状态，这些方法包括：

| 方法 | 描述 |
| --- | --- |
| public static boolean interrupted()  | 测试当前线程是否已经中断。线程的中断状态由该方法清除。|
| public boolean isInterrupted()  | 测试线程是否已经中断。线程的中断状态不受该方法的影响。|
| public void interrupt()  | 中断线程 |

更多内容请参考 [详细分析Java中断机制](http://www.infoq.com/cn/articles/java-interrupt-mechanism)。

## 线程池

构建一个新的线程是有一定代价的，因为涉及与操作系统的交互。如果程序中创建了大量的生命期很短的线程，应该使用线程池。使用线程池的好处是减少在创建和销毁线程上所花的时间以及系统资源的开销，解决资源不足的问题。如果不使用线程池，有可能造成系统创建大量同类线程而导致消耗完内存或者 “过度切换” 的问题。

Executors 类有许多静态工厂方法用来构建线程池：

| 方法  | 描述 |
| ------------- | ------------- |
| newCachedThreadPool  | 必要时创建新线程；空闲线程会被保留 60 秒  |
| newFixedThreadPool  | 包含固定数量的线程；空闲线程会一直被保留  |
| newSingleThreadPool  | 只有一个线程的“池”，该线程顺序执行每一个提交的任务  |
| newScheduledThreadPool  | 用于预定执行而构建的固定线程池，替代 java.util.Timer  |

```java
    /**
    * 参数依次是：
    * corePoolSize：最小线程数
    * maximunPoolSize：最大线程数量
    * keepAliveTime：线程最大生命期
    * unit：keepAliveTime 的时间单位
    * workQueue：任务队列
    * 此外还有线程工厂 threadFactory 和拒绝任务处理器 handler
    */
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }

    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
```

如果编写的是小程序，或者是轻载的服务器，使用 newCachedThreadPool 通常是一个不错的选择。在大负载的产品服务器中，最好使用 newFixedThreadPool，它为你提供了一个包含固定线程数目的线程池，或者为了最大限度地控制它，就直接使用 ThreadPoolExecutor 类。

另外，《阿里巴巴Java开发手册（纪念版）》中两个关于线程池的强制性规定如下：  
【强制】线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。  
    说明：Executors 返回的线程池对象的弊端如下：  
    1）FixedThreadPool 和 SingleThreadPool：  
    允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM。  
    2）CachedThreadPool 和 ScheduledThreadPool：  
    允许的创建线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，从而导致 OOM。

更详细的介绍请移步 [JAVA线程池的分析和使用](http://www.infoq.com/cn/articles/java-threadPool/)。

## 推荐阅读

* Brian Goetz 等. Java并发编程实战[M]. 机械工业出版社, 2012.

* 秦小波. 编写高质量代码:改善Java程序的151个建议[M]. 机械工业出版社, 2012.

* Java核心技术 卷I：基础知识（原书第10版）

* 周志明. 深入理解Java虚拟机, 第2版[M]. 机械工业出版社, 2013.

* [Lesson: Concurrency](https://docs.oracle.com/javase/tutorial/essential/concurrency/)

* [Java面试题全集（上）](http://blog.csdn.net/jackfrued/article/details/44921941)

* [Java Callable and Future Tutorial](https://www.callicoder.com/java-callable-and-future-tutorial/)

* [Creating and Starting Java Threads](http://tutorials.jenkov.com/java-concurrency/creating-and-starting-threads.html)

* [阿里巴巴Java开发手册（纪念版）](https://github.com/alibaba/p3c)

* [聊聊并发（三）——JAVA线程池的分析和使用](http://www.infoq.com/cn/articles/java-threadPool/)

* [Java Concurrency – yield(), sleep() and join() methods](https://www.geeksforgeeks.org/java-concurrency-yield-sleep-and-join-methods/)

* [详细分析Java中断机制](http://www.infoq.com/cn/articles/java-interrupt-mechanism)