---
title: volatile 关键字和原子类
date: 2018-01-24 11:31:34
tags: Java Concurrency
categories: Java
---

关键字 volatile 可以说是 Java 虚拟机提供的最轻量的同步机制，但是它并不容易完全被正确、完整的理解。本文探讨了 volatile 关键字和原子变量类的相关问题。

## volatile 关键字

当一个变量定义为 volatile 之后，它具备两种特性，第一是 **保证此变量对所有线程的可见性**，这里的 “可见性” 是指当一条线程修改了这个变量的值，新值对于其它线程来说是可以立即得知的。volatile 变量在各个线程的工作内存中不存在一致性问题，但 Java 里面的运算并非原子操作，导致 volatile 变量的运算在并发下一样是不安全的。

<!-- more -->

下面是一个例子，其输出结果小于 10000。

```java
import java.util.concurrent.*;

public class VolatileDemo {
    private static final int MAX_THREADS = 10;

    public static volatile int counter = 0;

    public static void increase() {
        counter++;
    }

    public static void main(String[] args) {
        ExecutorService pool = Executors.newFixedThreadPool(MAX_THREADS);

        for (int i = 0; i < MAX_THREADS; i++) {
            pool.submit(() -> {
                for (int j = 0; j < 1000; j++) {
                    increase();
                }
            });
        }

        pool.shutdown();
        try {
            pool.awaitTermination(5, TimeUnit.SECONDS); // 等待所有线程结束
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(counter);
    }
}
```

由于 volatile 变量只能保证可见性，在不符合以下两条规则的运算场景中，我们仍然要通过加锁（使用 synchronized 或 java.util.concurrent 中的原子类）来保证原子性。

* 远算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。

* 变量不需要与其他的状态变量共同参与不变约束。

下面是 volatile 变量的一种典型用法：检查某个状态标记以判断是否退出循环。

```java
    volatile boolean shutdownRequested;

    public void shutdown() {
        shutdownRequested = true;
    }

    public void doWork() {
        while (!shutdownRequested) {
            // do stuff
        }
    }
```

使用 volatile 变量的第二个语义是 **禁止指令重新排序优化**，普通的变量仅仅会保证在该方法的执行过程中所有依赖赋值结果的地方都能获取到正确的结果，而不能保证变量赋值操作的顺序与程序代码中的执行顺序一致。

下面是标准的 DCL（双锁检测）单例模式的代码。

```java
public class Singleton {
    private volatile static Singleton instance; // volatile 修饰

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized(Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }

    public static void main(String[] args) {
        Singleton.getInstance();
    }
}
```

有关指令重排的内容请移步 [Java内存访问重排序的研究](https://tech.meituan.com/java-memory-reordering.html)。更多关于 Volatile 变量的介绍，可阅读 Brian Goetz 大神的 [这篇文章](https://www.ibm.com/developerworks/java/library/j-jtp06197/)。

## 原子类

对任意单个 volatile 变量的读/写具有原子性，但类似于 `counter++` 这种复合操作不具有原子性。那如何如何解决 VolatileDemo 类中的问题呢？可以使用同步机制（要么是 synchronized 关键字，要么是显式的 Lock 对象）。另外，还可以使用原子性变量类。

java.util.concurrent.atomic 包中有很多类使用了很高效的机器级指令（而不是使用锁）来保证其他操作的原子性。例如，Atomiclnteger 类提供了方法 getAndIncrement 以原子方式将一个整数自增。

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicDemo {
    private static final int MAX_THREADS = 10;

    public static AtomicInteger integer = new AtomicInteger(0);

    public static void increase() {
        integer.getAndIncrement();
    }

    public static void main(String[] args) {
        ExecutorService pool = Executors.newFixedThreadPool(MAX_THREADS);

        for (int i = 0; i < MAX_THREADS; i++) {
            pool.submit(() -> {
                for (int j = 0; j < 1000; j++) {
                    increase();
                }
            });
        }

        pool.shutdown();
        try {
            pool.awaitTermination(5, TimeUnit.SECONDS); // 等待所有线程结束
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(integer.get());
    }
}
```

原子类使用 CAS（Compare-and-Swap，比较并交换）操作。CAS 指令需要三个操作数，分别是内存位置（用 V 表示）、旧的预期值（用 A 表示）和新值（用 B 表示）。CAS 指令执行时，当且仅当 V 符合旧的预期值 A 时，处理器才用新值 B 更新 V 的值，否则它就不指定更新，但是无论是否更新了 V 的值，都会返回 V 的旧值，上述的处理过程是一个原子操作。

```java
    // AtomicInteger 类中的方法
    // Atomically increments by one the current value.
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

    // sun.misc.Unsafe 类中的方法
    public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!compareAndSwapInt(o, offset, v, v + delta));
        return v;
    }

    // Atomically update Java variable to x if it is currently holding expected.
    public final native boolean compareAndSwapInt(Object o, long offset, int expected, int x);
```

如果有大量线程要访问相同的原子值，性能会大幅下降，因为乐观更新需要太多次重试。JDK 8 提供了 LongAdder 和 LongAccumulator 类来解决这个问题。LongAdder 包括多个变量（加数)，其总和为当前值。可以有多个线程更新不同的加数，线程个数增加时会自动提供新的加数。通常情况下，只有当所有工作都完成之后才需要总和的值， 对于这种情况，这种方法会很高效。性能会有显著的提升。

有关 LongAdder 的源码解析请参考 [这里](http://ifeve.com/atomiclong-and-longadder/)。

## 推荐阅读

* Brian Goetz 等. Java并发编程实战[M]. 机械工业出版社, 2012.

* 周志明. 深入理解Java虚拟机, 第2版[M]. 机械工业出版社, 2013.

* [Java Volatile Keyword](http://tutorials.jenkov.com/java-concurrency/volatile.html)

* [The "Double-Checked Locking is Broken" Declaration](https://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)

* [Double Checked Locking in Singleton](https://stackoverflow.com/questions/18093735/double-checked-locking-in-singleton)

* [Managing volatility](https://www.ibm.com/developerworks/java/library/j-jtp06197/)

* [深入理解Java内存模型（四）——volatile](http://www.infoq.com/cn/articles/java-memory-model-4)

* [Java内存访问重排序的研究](https://tech.meituan.com/java-memory-reordering.html)

* [What Volatile Means in Java](http://jeremymanson.blogspot.com/2008/11/what-volatile-means-in-java.html)

* [Atomicity, Visibility and Ordering](http://jeremymanson.blogspot.com/2007/08/atomicity-visibility-and-ordering.html)

* [How CAS (Compare And Swap) in Java works](https://dzone.com/articles/how-cas-compare-and-swap-java)

* [比 AtomicLong 还高效的 LongAdder 源码解析](http://ifeve.com/atomiclong-and-longadder/)