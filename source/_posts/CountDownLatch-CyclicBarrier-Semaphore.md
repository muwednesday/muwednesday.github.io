---
title: CountDownLatch、 CyclicBarrier 与 Semaphore 的比较
date: 2018-01-24 19:06:57
tags: Java Concurrency
categories: Java
---

java.util.concurrent 中包含了几个能帮助人们管理相互合作的同步工具类，本文介绍了其中的 CountDownLatch、 CyclicBarrier 以及 Semaphore。

## CountDownLatch

闭锁是一种同步工具类，可以延迟线程的进度直到其到达终止状态。CountDownLatch 是一种灵活的闭锁实现，它可以使一个或者多个线程等待一组事件的发生。闭锁状态包括一个计数器，该计数器被初始化为一个正数，表示需要等待的事件数量。countDown 方法递减计数器，表示有一个事件已经发生，而 await 方法等待计数器达到零，这表示需要等待的事情都已经发生。如果计数器的值非零，那么 await 会一直阻塞直到计数器为零，或者等待中的线程中断，或者等待超时。

下面这个例子给出了闭锁的两种常见用法。startLatch 计数器初始值为 1， endLatch 计数器初始值为工作线程的数量。每个工作线程首先要做的就是在 startLatch 上等待，从而确保所有线程都就绪后才开始执行。而每个线程要做的最后一件事情是调用 endLatch 的 countDown 方法，这能使主线程高效的等待直到所有的线程都执行完成。

<!-- more -->

```java
// 修改自《Java Concurrency in Practice》 5-11 的 TestHarness
import java.util.concurrent.*;

class Task implements Runnable {

    @Override
    public void run() {
        try {
            System.out.println(Thread.currentThread().getName() + " is running");
            TimeUnit.SECONDS.sleep(2); // 模拟任务执行
            System.out.println(Thread.currentThread().getName() + " is finished");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
public class CountDownLatchDemo {

    public void execute(int nThreads, final Runnable task) {
        final CountDownLatch startLatch = new CountDownLatch(1);
        final CountDownLatch endLatch = new CountDownLatch(nThreads);

        for (int i = 0; i < nThreads; i++) {
            new Thread(() -> {
                try {
                    startLatch.await();
                    try {
                        task.run(); // 执行任务
                    } finally {
                        endLatch.countDown();
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }

        try {
            // give the threads chance to start up; we could perform
            // initialisation code here as well.
            TimeUnit.MILLISECONDS.sleep(200);
            System.out.println("Start...");
            startLatch.countDown();
            endLatch.await();
            System.out.println("End...");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        new CountDownLatchDemo().execute(5, new Task());
    }
}

// 输出结果，顺序可能不一致
Start...
Thread-1 is running
Thread-4 is running
Thread-0 is running
Thread-2 is running
Thread-3 is running
Thread-1 is finished
Thread-0 is finished
Thread-4 is finished
Thread-3 is finished
Thread-2 is finished
End...
```

CountDownLatch 是通过 AbstractQueuedSynchronizer（AQS）实现的，具体细节可参考 [这里](http://www.infoq.com/cn/articles/java8-abstractqueuedsynchronizer)。

## CyclicBarrier

我们已经看到通过闭锁来启动一组相关的操作，或者等待一组相关的操作结束。闭锁是一次性对象，一旦进入终止状态，就不能被重置。

栅栏（Barrier）类似于闭锁，它能阻塞一组线程直到某个事件发生。栅栏和闭锁的关键区别在于，所有线程必须同时达到栅栏位置，才能继续执行。CyclicBarrier 可以使一定数量的参与方反复地在栅栏位置汇集，它在并行迭代算法中非常有用，有兴趣的可以看下 [CyclicBarrier example: a parallel sort algorithm](https://www.javamex.com/tutorials/threads/CyclicBarrier_parallel_sort.shtml)。

当现成达到栅栏位置时将调用 await 方法这个方法将阻塞直到所有线程都达到栅栏位置。如果所有线程都达到了栅栏位置，那么栅栏将打开，此时所有线程都被释放，而栅栏将被重置一遍下次使用。如果对 await 的调用超时，或者 await 阻塞的线程被中断，那么栅栏就被认为是被打破了，所有阻塞的 await 调用都将终止并抛出 BrokenBarrierException。如果成功通过栅栏，那么 await 将为每个线程返回一个唯一的到达索引号。CyclicBarrier 还可以使你将一个栅栏操作传递给构造函数，这是一个 Runnable，当成功通过栅栏时会（在一个子任务线程中）执行它，但在阻塞线程被释放之前是不能执行的。下面是一个 Two Step 的例子。

```java
import java.util.concurrent.*;

public class CyclicBarrierDemo {
    private static final int MAX_THREADS = 5;

    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(MAX_THREADS,
                new Runnable() {
                    private int count = 1;

                    @Override
                    public void run() {
                        System.out.println("Cyclic Barrier Finished " + count++);
                    }
                });

        ExecutorService pool = Executors.newFixedThreadPool(MAX_THREADS);
        for (int i = 0; i < MAX_THREADS; i++) {
            pool.submit(new Worker("Thread-" + i, barrier));
        }

        pool.shutdown();
    }

    private static class Worker implements Runnable {
        private CyclicBarrier barrier;
        private String name;

        public Worker(String name, CyclicBarrier barrier) {
            this.name = name;
            this.barrier = barrier;
        }

        public void run() {
            try {
                System.out.println("Doing Step 1 Work on " + name);
                TimeUnit.SECONDS.sleep(2);
                System.out.println("Finished Step 1 work on " + name);

                // zero indicates the last to arrive
                if (barrier.await() == 0) {
                    barrier.reset();  // 重置
                }

                System.out.println("Doing Step 2 Work on " + name);
                TimeUnit.SECONDS.sleep(2);
                System.out.println("Finished Step 2 work on " + name);
                barrier.await();

            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        }

    }
}

// 输出结果，顺序可能不一致
Doing Step 1 Work on Thread-1
Doing Step 1 Work on Thread-2
Doing Step 1 Work on Thread-4
Doing Step 1 Work on Thread-0
Doing Step 1 Work on Thread-3
Finished Step 1 work on Thread-2
Finished Step 1 work on Thread-1
Finished Step 1 work on Thread-0
Finished Step 1 work on Thread-4
Finished Step 1 work on Thread-3
Cyclic Barrier Finished 1
Doing Step 2 Work on Thread-1
Doing Step 2 Work on Thread-2
Doing Step 2 Work on Thread-0
Doing Step 2 Work on Thread-4
Doing Step 2 Work on Thread-3
Finished Step 2 work on Thread-2
Finished Step 2 work on Thread-0
Finished Step 2 work on Thread-1
Finished Step 2 work on Thread-4
Finished Step 2 work on Thread-3
Cyclic Barrier Finished 2
```

有关 CountDownLatch 和 CyclicBarrier 区别的讨论，请移步 [Stack Overflow](https://stackoverflow.com/questions/4168772/java-concurrency-countdown-latch-vs-cyclic-barrier)。

## Semaphore

计数信号量（Counting Semaphore）用来控制同时访问某个特定资源的访问数量，或者同时执行某个指定操作的数量。计数信号量还可以用来实现某种资源池，或者对容器施加边界。

Semaphore 中管理着一组虚拟的许可（permit），许可的初始容量可通过构造函数来指定。在执行操作时可以首先获得许可（只要还有剩余的许可），并在使用以后释放许可。如果没有许可，那么 acquire 将阻塞直到有许可（或者直到被中断或者操作超时）。release 方法将返回一个许可给信号量。二值信号量（即初始值为 1 的 Semaphore）可以用做互斥体（mutex），并具备不可重入的加锁语义：谁拥有这个唯一的许可，谁就拥有了互斥锁。因此，它可以用来 [实现生产者-消费者模式](http://www2.hawaii.edu/~walbritt/ics240/synchronization/Solution.java)。

下面是一个 Semaphore 的实例。

```java
import java.util.concurrent.*;

public class SemaphoreDemo {
    private static final int MAX_PERMITS = 5;
    private static final int MAX_WORKERS = 8;

    public static void main(String[] args) {

        Semaphore semaphore = new Semaphore(MAX_PERMITS);

        ExecutorService pool = Executors.newFixedThreadPool(MAX_PERMITS);
        for (int i = 0; i < MAX_WORKERS; i++) {
            pool.submit(new Worker("Thread-" + i, semaphore));
        }

        pool.shutdown();

    }

    private static class Worker implements Runnable {

        private String name;
        private Semaphore semaphore;

        public Worker(String name, Semaphore semaphore) {
            this.name = name;
            this.semaphore = semaphore;
        }

        @Override
        public void run() {
            try {
                semaphore.acquire();  // 获得许可
                System.out.println(name + " gets a permit");

                TimeUnit.SECONDS.sleep(2);

                System.out.println(name + " releases the permit");
                semaphore.release(); // 释放许可
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

// 输出结果，顺序可能不一致
Thread-1 gets a permit
Thread-0 gets a permit
Thread-2 gets a permit
Thread-3 gets a permit
Thread-4 gets a permit
Thread-0 releases the permit
Thread-2 releases the permit
Thread-1 releases the permit
Thread-5 gets a permit
Thread-3 releases the permit
Thread-4 releases the permit
Thread-7 gets a permit
Thread-6 gets a permit
Thread-5 releases the permit
Thread-7 releases the permit
Thread-6 releases the permit
```

## 推荐阅读

* Brian Goetz 等. Java并发编程实战[M]. 机械工业出版社, 2012.

* [CountDownLatch和CyclicBarrier的区别](http://scau-fly.iteye.com/blog/1955165)

* [Coordinating threads with CountDownLatch](https://www.javamex.com/tutorials/threads/CountDownLatch.shtml)

* [CyclicBarrier: concordinating the stages of a multithreaded operation](https://www.javamex.com/tutorials/threads/CyclicBarrier.shtml)

* [Java并发编程：CountDownLatch、CyclicBarrier和Semaphore](https://www.cnblogs.com/dolphin0520/p/3920397.html)

* [Java concurrency: Countdown latch vs Cyclic barrier](https://stackoverflow.com/questions/4168772/java-concurrency-countdown-latch-vs-cyclic-barrier)

* [A Comparison of CountDownLatch, CyclicBarrier and Semaphore](http://shazsterblog.blogspot.com/2011/12/comparison-of-countdownlatch.html)

* [深度解析Java 8：AbstractQueuedSynchronizer的实现分析（下）](http://www.infoq.com/cn/articles/java8-abstractqueuedsynchronizer)

* [生产者消费者问题的五种实现](http://huachao1001.github.io/article.html?QhSkxKKX)

* [Solution.java](http://www2.hawaii.edu/~walbritt/ics240/synchronization/Solution.java)
