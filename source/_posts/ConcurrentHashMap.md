---
title: ConcurrentHashMap
date: 2019-07-08 10:08:56
tags: Java Collections, Java Concurrency
categories: Java
---

## 

hello concurrenthashmap
<!-- more -->

## API

### Constant



```java
static final int MOVED     = -1; // hash for forwarding nodes
static final int TREEBIN   = -2; // hash for roots of trees
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
```

相当于 HashMap 的 `threshold`，构造函数中与 loadFactor 无关，只有真正初始化的时候才会变成 capacity * loadFactor.

```java
// table 初始化和扩容的标志位，非常重要
// 负数代表正在进行初始化或扩容操作，-1 代表正在初始化，低16位的 N（正数）表示有 N-1 个线程正在进行扩容操作
// 正数或 0 代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小，与 loadFactor 有关
private transient volatile int sizeCtl;

// 扩容时nextTable的索引 + 1，当 transferIndex<=0 时表示扩容已经完成
private transient volatile int transferIndex;
```

说白了，sizeCtl 在扩容时有两部分组成，int 类型一共32位，高16位为rs，通过`resizeStamp()`保证最高位一定是1，因此扩容时 sizeCtl 是一个负数；而低16位是扩容线程数 + 1.扩容时，首个线程将 sizeCtl 设置成 `(resizeStamp(table.length) << RESIZE_STAMP_SHIFT) + 2`（见`addCount()`函数），即高16位为rs，低16位为线程数 1+1=2.
举个例子：

```
// table size = 1024，前面21个0
00000000 00000000 00000100 00000000
// rs = 21 | (1 << 15)
00000000 00000000 10000000 00010101
// sc = rs << 16 + 2
//   高16位rs      低16位扩容线程数+1
10000000 00010101 00000000 00000010
```

```java
// 扩容时每个线程的最小步长，即每个线程至少操作16个bin
private static final int MIN_TRANSFER_STRIDE = 16;
// sizeCtl的标识位数，一共32位，高16位代表扩容的标识，resizeStamp，rs
private static int RESIZE_STAMP_BITS = 16;
// 最大的扩容线程数，为2^16-1=65535
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
// sizeCtl中记录rs的移动位数
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

```

```java
    // 前面有几个0 | 10000000 00000000，保证最高位必是1
    static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
```

### NODE
Node 用于存储单个KV数据节点，内部有key、value、hash、next等属性，它有4个子类，继承关系如下：

![NODE](Node.png)

TreeBin：并不存储实际数据，维护对桶内红黑树的读写锁，存储对红黑树节点的引用。
TreeNode：在红黑树结构中，实际存储数据的节点
ForwardingNode：扩容转发节点，放置此节点后，外部对原有table的操作会转发到nextTable上
ReservationNode：占位加锁节点，执行某些方法时，对其加锁，如computeIfAbsent

### GET
```java
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        // 计算hash
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            // 可能是只有一个节点的链表，如果首节点是的话直接返回
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            // 实现在个子类的find()方法中，具体的：
            // ForwardingNode(-1)查找nextTable
            // TreeBin(-2)的话遍历红黑树节点
            // ReservationNode(-3)直接返回null
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            // 循环链表
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

### PUT
```java
    /**
     * Implementation for put and putIfAbsent
     * JDK12.0.1
     */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        // 这里也说明了ConcurrnetHashMap的KV不能为null
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh; K fk; V fv;
            // 初始化
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            // 空的bin，直接CAS设置值
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                    break;                   // no lock when adding to empty bin
            }
            // 是ForwardingNode，协助扩容
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            // putIfAbsent的优化，直接返回原来的值，JDK8没有这一步
            else if (onlyIfAbsent // check first node without acquiring lock
                     && fh == hash
                     && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                     && (fv = f.val) != null)
                return fv;
            else {
                V oldVal = null;
                // 非空bin，需要加锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        // 链表
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 找到已有值
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                // 在链表后添加节点
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key, value);
                                    break;
                                }
                            }
                        }
                        // TreeBin
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) {
                    // 判断是否需要转为红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 计数
        addCount(1L, binCount);
        return null;
    }
```

与 HashMap 的 `hash()` 方法基本一致，唯一不同是强制最高位为0.

```java
    // key hash 算法，异或，HASH_BITS为31个1
    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
```

ConcurrentHashMap 只是在构造函数中使用 threshold，并且只影响初始容量，后面的扩容是按照 `n - (n >>> 2)` 计算的。

```java
    /**
     * Initializes table, using the size recorded in sizeCtl.
     */
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            // 初始化，sizeCtl小于0，当前线程yield，即只有一个线程进行初始化
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
                // CAS设置sizeCtl为-1，表示在初始化
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        // 相当于0.75倍的table size
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

协助扩容。

```java
    /** 
     * Helps transfer if a resize is in progress.
     * JDK12.0.1
     */
    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        // 遇到ForwadingNode，协助扩容
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length) << RESIZE_STAMP_SHIFT;
            // table和nextTable没有被并发修改，且sc<0，还在扩容
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                // 扩容结束的条件：扩容线程达到最大值，或者没有线程进行扩容，或者nextTable的索引transferIndex<=0
                if (sc == rs + MAX_RESIZERS || sc == rs + 1 ||
                    transferIndex <= 0)
                    break;
                // CAS将sc加一，则当前线程开始协助扩容，跳出循环
                if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```

```java
     /**
     * Moves and/or copies the nodes in each bin to new table. See
     * above for explanation.
     * JDK12.0.1
     */
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        // 根据CPU核数设置线程处理的bin的数量
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        // nextTable没有初始化，进行初始化
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        // 是否需要继续向前查找bin，每个线程需要完成stride个bin的检查
        boolean advance = true;
        // 是否完成扩容
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                // 本线程stride个bin还没有处理完，或者完成扩容
                if (--i >= bound || finishing)
                    advance = false;
                // 给nextIndex赋值，没有完成扩容，可以继续处理剩下的bin
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                // CAS设置值，占坑，本线程处理stride个bin，其他人不要掺和这些bin了~
                // 比方说一共64个bin，我这里处理48~63，此时i为63，transferIndex变为48，第一个if的--i>=bound成立，循环16次，即处理16个bin
                // 如果还有bin未处理，本线程可以继续处理~
                // 例如两个线程，第一个处理48~63，第二个处理32~47；第一个线程处理完16个，可以继续处理16~31；同理第二个线程可以处理0~15
                else if (U.compareAndSetInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            // 应该是只有i<0有用，代表所有的bin都处理过了，可以退出
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 扩容完成，应该是最后一个扩容的线程才会去执行这一步
                // nextTable设置null，help gc；然后设置table和sizeCtl，退出
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                // 线程数-1
                if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 不是最后一个扩容线程，直接溜了~
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    // 是的话，设置完成的标识，重新检查下~
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            // 是空，直接CAS设置bin为ForwardingNode
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            // 该bin被处理过了，继续向前
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                // 该bin有数据，需要加锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        // 定义一个low链表、一个high链表，和HashMap差不多
                        Node<K,V> ln, hn;
                        // 链表，因为TreeBin的hash是-2
                        if (fh >= 0) {
                            // 根据 hash&n 判断是在low还是high上
                            // fh&2^x 即判断fh的第x位是0还是1，相当于随机，用于判断Node是在low还是high上
                            int runBit = fh & n;
                            // lastRun是原链表中hash值第x位最后一个变化的节点，因为如果都不变化的话，直接把剩余的节点赋值next就行
                            // 例如1->0->1->0->【1】->1->1，可以把【1】看做一个整体，便于后面构造链表
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            // 判断最后一个节点变化节点lastRun的hash的第x位，0的在low上，1的在high上
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            // 循环构造两个链表，这里反序了
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            // low的设置在nextTable的i位置，high的设置在i+n位置，并将原table的bin设置为ForwardingNode
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            // 继续向前
                            advance = true;
                        }
                        // 红黑树，差不多，最后检查是否需要转为链表
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        // ReservationNode的话，循环更新，抛异常
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
            }
        }
    }
```

```java
    private final void addCount(long x, int check) {
        CounterCell[] cs; long b, s;
        // counterCells不为空，或者CAS更新baseCount失败
        if ((cs = counterCells) != null ||
            !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell c; long v; int m;
            boolean uncontended = true;
            if (cs == null || (m = cs.length - 1) < 0 ||
                (c = cs[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n) << RESIZE_STAMP_SHIFT;
                if (sc < 0) {
                    if (sc == rs + MAX_RESIZERS || sc == rs + 1 ||
                        (nt = nextTable) == null || transferIndex <= 0)
                        break;
                    if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                // 首个线程扩容，sc = rs + 2
                else if (U.compareAndSetInt(this, SIZECTL, sc, rs + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```


```java
/**
 * Base counter value, used mainly when there is no contention,
 * but also as a fallback during table initialization
 * races. Updated via CAS.
 * 没有冲突是直接使用它作为size
 */
private transient volatile long baseCount;
/**
 * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
 */
private transient volatile int cellsBusy;
/**
 * Table of counter cells. When non-null, size is a power of 2.
 * 并发时counterCells不为空
 */
private transient volatile CounterCell[] counterCells;
```


```java
    public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
    }
    
    public long mappingCount() {
        long n = sumCount();
        return (n < 0L) ? 0L : n; // ignore transient negative values
    }
```

```java
    /** 
     * A padded cell for distributing counts.  Adapted from LongAdder
     * and Striped64.  See their internal docs for explanation.
     */
    @jdk.internal.vm.annotation.Contended static final class CounterCell {
        volatile long value;
        CounterCell(long x) { value = x; }
    }
```

```java
    final long sumCount() {
        CounterCell[] cs = counterCells;
        // 先使用baseCount，如果counterCells不为空，累加
        long sum = baseCount;
        if (cs != null) {
            for (CounterCell c : cs)
                if (c != null)
                    sum += c.value;
        }
        return sum;
    }
```
    
```java
    // See LongAdder version for explanation
    private final void fullAddCount(long x, boolean wasUncontended) {
        int h;
        if ((h = ThreadLocalRandom.getProbe()) == 0) {
            ThreadLocalRandom.localInit();      // force initialization
            h = ThreadLocalRandom.getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            CounterCell[] cs; CounterCell c; int n; long v;
            if ((cs = counterCells) != null && (n = cs.length) > 0) {
                if ((c = cs[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {            // Try to attach new Cell
                        CounterCell r = new CounterCell(x); // Optimistic create
                        if (cellsBusy == 0 &&
                            U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                            boolean created = false;
                            try {               // Recheck under lock
                                CounterCell[] rs; int m, j;
                                if ((rs = counterCells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                else if (U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))
                    break;
                else if (counterCells != cs || n >= NCPU)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 &&
                         U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                    try {
                        if (counterCells == cs) // Expand table unless stale
                            counterCells = Arrays.copyOf(cs, n << 1);
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = ThreadLocalRandom.advanceProbe(h);
            }
            else if (cellsBusy == 0 && counterCells == cs &&
                     U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                boolean init = false;
                try {                           // Initialize table
                    if (counterCells == cs) {
                        CounterCell[] rs = new CounterCell[2];
                        rs[h & 1] = new CounterCell(x);
                        counterCells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (U.compareAndSetLong(this, BASECOUNT, v = baseCount, v + x))
                break;                          // Fall back on using base
        }
    }
```
