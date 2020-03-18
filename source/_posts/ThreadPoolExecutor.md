---
title: ThreadPoolExecutor
date: 2019-07-15 09:48:18
tags:
---


线程池

<!-- more -->

![Executor](Executor.png)

## API

### Constants
int类型，一共32位，其中前三位代表线程池的状态`runState`,后29位表示工作线程的数量`workerCount`。

```java
    // 控制参数
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    // 29
    private static final int COUNT_BITS = Integer.SIZE - 3;
    // 00011111 11111111 1111111 11111111 29个1
    private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    // 前三位表示状态，分别为111、000、001、010、011，后面再补29个0
    // 线程池能接受新任务
    private static final int RUNNING    = -1 << COUNT_BITS;
    // 不再接受新任务，但可以继续执行队列中的任务
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    // 全面拒绝，并中断正在处理的任务
    private static final int STOP       =  1 << COUNT_BITS;
    // 表示所有任务已经被终止
    private static final int TIDYING    =  2 << COUNT_BITS;
    // 已清理完现场
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    // 根据ctl得到runState和workerCount
    private static int runStateOf(int c)     { return c & ~COUNT_MASK; }
    private static int workerCountOf(int c)  { return c & COUNT_MASK; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```


### Executors

```java
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
    
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }
```


### Methods

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        // 工作线程的数量小于核心线程数
        if (workerCountOf(c) < corePoolSize) {
            // 创建新的线程，成功直接返回
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 线程池还在running并且工作队列没有满
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // double-check，不是running状态，回滚，拒绝
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 防止了SHUTDOWN状态下没有活动线程了，但是队列里还有任务没执行这种特殊情况
            // 添加一个null任务是因为SHUTDOWN状态下，线程池不再接受新任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 队列已满并且创建线程失败，拒绝
        else if (!addWorker(command, false))
            reject(command);
    }
```

根据当前线程池状态，检查是否可以添加新的任务线程。如果可以则创建并启动任务，一切正常返回true，返回false可能是以下原因：
（1）线程池没有处于RUNNING状态
（2）线程工厂创建新的任务线程失败，即大于poolSize
```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (int c = ctl.get();;) {
            // Check if queue empty only if necessary.
            // rs不小于SHUTDOWN，并且满足三条任意一条：rs不小于STOP、firstTask不为null、工作队列为空
            // 说明线程池没有处于RUNNING状态，返回false
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP)
                    || firstTask != null
                    || workQueue.isEmpty()))
                return false;

            for (;;) {
                // 判断wc与poolSize的大小，core为true判断核心线程，false判断最大线程
                // 达到了最大线程，返回false
                if (workerCountOf(c)
                    >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                    return false;
                // CAS设置线程数+1，成功跳出循环，负数(RUNNING)+1也一样，秀！
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateAtLeast(c, SHUTDOWN))
                    continue retry;
                // 只有RUNNING状态才会到这里
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        // 工作线程数+1成功，开始创建Worker并启动线程，启动失败时-1
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int c = ctl.get();
                    
                    // RUNNING 或者 SHUTDOWN且firstTask为null
                    if (isRunning(c) ||
                        (runStateLessThan(c, STOP) && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    // Worker的thread是Worker自身的依附对象，相当于new Thread(worker)
                    // start方法执行Worker的run()，即runWorker()
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    try {
                        task.run();
                        afterExecute(task, null);
                    } catch (Throwable ex) {
                        afterExecute(task, ex);
                        throw ex;
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

```java
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();

            // Check if queue empty only if necessary.
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```


```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
```