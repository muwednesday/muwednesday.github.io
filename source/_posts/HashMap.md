---
title: HashMap 源码分析
date: 2018-01-03 20:10:58
tags: Java Collections
categories: Java
---

{% blockquote Documentation https://docs.oracle.com/javase/8/docs/api/java/util/HashMap.html Java Platform SE 8 %}
Hash table based implementation of the Map interface. This implementation provides all of the optional map operations, and permits null values and the null key. (The HashMap class is roughly equivalent to Hashtable, except that it is unsynchronized and permits nulls.) This class makes no guarantees as to the order of the map; in particular, it does not guarantee that the order will remain constant over time.
{% endblockquote %}

HashMap 是基于散列表的 Map 接口的非同步实现，用于处理映射（键值对）。本文以 JDK 8 的源码为基础，深入分析了 HashMap 的 get&put 方法、扩容机制以及遍历等内容。

## HashMap 简介

链表和数组可以按照人们的意愿排列元素的次序，但当查找某个元素时，使用这两种数据结构效率较低。如果不在意元素的顺序，可以用散列表来快速查找所需要的对象。散列表（Hash table，也叫哈希表），是根据键而直接访问在内存存储位置的数据结构。也就是说，它通过计算一个关于键值的函数，将所需查询的数据映射到表中一个位置来访问记录，这加快了查找速度。这个映射函数称为散列函数，存放记录的数字称为散列表。

有时我们知道某些键的信息，并想要查找与之对应的元素，这就需要用到映射（map）。映射用来存放键值对，如果提供了键， 就能够查找到值。HashMap 是散列映射的一种实现，对 key 进行散列，且只能作用于 key，与之关联的 value 不能进行散列。

<!-- more -->

HashMap 使用[链地址法](https://www.geeksforgeeks.org/hashing-set-2-separate-chaining/)（数组 + 链表）处理哈希碰撞，JDK 8 又加入了红黑树的部分。其继承关系如下图所示：

![HashMap](HashMap.png)

可见 Map 是一个不同于 Collection 的单独接口。

## API

### 重要参数

HashMap 中定义了几个重要的参数：

```java
// 默认初始容量，必须为2的幂次方，这里为16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
// 最大容量，Integer.MAX_VALUE = 2^31-1，这里只能取2^30
static final int MAXIMUM_CAPACITY = 1 << 30;
// 默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 链表转红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;
// 红黑树转链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;
// 链表转红黑树需满足的最小容量
static final int MIN_TREEIFY_CAPACITY = 64;

// 扩容的阈值
int threshold;
// 负载因子
final float loadFactor;
// 键值对的数目
transient int size;
```

其中 Capacity 表示散列表中 buckets 的数目，也就是数组的长度； load factor 是负载因子，即数组的填充度；threshold 为扩容的阈值，显然 `threshold = Capacity * load factor`。当 size 大于 threshold 时，就要进行扩容。

HashMap 中数组的容量必须为 2 的幂次方，主要是为了在取模和扩容时做优化；同时为了减少冲突，HashMap 定位数组索引位置时，也加入了高位参与运算的过程。tableSizeFor 方法保证了这一点。

```java
    // 计算最小的2的幂次方，使得其不小于cap，比如 tableSizeFor(100) = 128
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

### 索引位置的确定

HashMap 的键值对存储在名为 table 的 Node 型数组中。 Node 是 HashMap 中的静态内部类，实现了 Map.Entry<K,V> 接口，包含 key、value 以及 hash 值，并且保存下一个 Node 的引用，属于单向链表节点。

```java
// Node 数组
transient Node<K,V>[] table;
// Node
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash; // hash 算法的结果
    final K key;
    V value;
    Node<K,V> next; // 下一个 Node 的引用
    // 省略其他
 }
```

hash 并不是 key 的 hashCode 值，而是 hashCode 高 16 位异或低 16 位的结果。这主要是从速度、功效、质量来考虑的，当 table 数组的 length 比较小的时候，也能保证考虑到高低 bit 都参与到 hash 的计算中，同时不会有太大的开销。

```java
    // JDK 8的 hash 算法，通过 hashCode() 的高16位异或低16位实现
    // putVal 中 (n - 1) & hash 相当于取模运算
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

对于任意给定的 key，相同的 hashCode 必定对应相同的 hash 值。

在确定数组索引位置时，一般采用对数组长度取模的方法，使元素的分布较为均匀。HashMap 中对此过程做了优化，相较于消耗较大的取模运算，采用 `(length - 1) & hash` 来计算。这种做法非常巧妙，由于 HashMap 底层数组的长度总是 2 的幂次方，它等价于对 length 取模，，但是 `&` 比 `%` 有更高的效率。

下面是一个例子：

```java
Map<LocalDateTime, String> map = new HashMap<>(16);
// hashCode = -833367753, hash = -833413276, index = 4
map.put(LocalDateTime.of(2018, 1, 12, 20, 0, 0), "clock");
```

![How does the bucket index calculation work?](hash.png)

### put 方法

put 方法比较复杂，大致可以分为七个步骤：

1. 若 table 为 null 或 length 等于 0，通过 resize 方法初始化。

2. 计算索引位置，若数组中该位置正好为 null，直接添加新节点，转到 7；否则需要处理 hash 碰撞，转到 3。

3. 若与 first node (存放在数组中，链表的首个节点或红黑树的根节点) 的 key 相同，value 会被覆盖，转到 6；否则转向 4。这里的相同是指 hash 相等且 `==` 或 equals 二者有一个为 true，下同。

4. 若节点为 TreeNode，调用 putTreeVal 方法（在 key 相同时会覆盖 value 并返回节点，插入新节点会返回 null），转向 6；否则属于链表，转向 5。

5. 循环遍历链表，若找到 key 相同的节点，value 会被覆盖；否则在最后追加新节点，如果此时链表长度大于 `8`，则要调用 treeifyBin 方法将链表转为红黑树，转向 6。注意，数组长度小于 `64` 时不会转换而是继续扩容。

6. 统一处理 key 相同的情况，相同的话覆盖 oldValue 并返回，否则转到 7。

7. `++size` 若大于 threshold，要进行扩容。最后返回 null。

源码如下：

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // table 为空，通过 resize 创建
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 计算 index，若该位置为空，直接存放 Node
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else { // 处理 Hash 碰撞
            Node<K,V> e; K k;
            // 与数组中 first node 的 key 相同，会被覆盖
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 属于红黑树，插入新值返回 null，覆盖返回对应的 node
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {  
                // 属于链表，binCount 计算链表的长度（不包括 first node，因此会 -1）
                for (int binCount = 0; ; ++binCount) {
                    // 在链表最后添加
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            // 转红黑树，当 tab.length < MIN_TREEIFY_CAPACITY 不会转换，而是 resize
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 与链表中其他 node 的 key 相同，会被覆盖
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // 统一处理 key 相同的情况，覆盖旧值并返回
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        // 真正的 put 了一个 node
        ++modCount;
        if (++size > threshold) // 需要时扩容
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

JDK 8 利用红黑树快速增删改查的特点提高 HashMap 的性能。下面的例子演示了一次由链表转红黑树的过程：

```java
    // 64 的倍数（包括 0）都保存在数组中 index 为 0 的位置
    Map<Integer, String> map = new HashMap<>(64);
    for (int i = 0; i < 38; i++) { // 为了防止扩容，总的 size 不要超过 48
        map.put(i, Integer.toString(i));
    }
    map.put(64, "64");
    map.put(128, "128");
    map.put(192, "192");
    map.put(256, "256");
    map.put(320, "320");
    map.put(384, "384");
    map.put(448, "448");
    // 链表长度已是8，再加一个就会转为红黑树
    map.put(512, "512");
```

正常情况下，红黑树的 first node 为其根节点。转换后的红黑树如下图所示：

![RBTree](RBTree.png)

上图使用 [Red/Blcak Tree](https://www.cs.usfca.edu/~galles/visualization/RedBlack.html) 绘制。想了解更多关于红黑树的知识，请参考[教你透彻了解红黑树](https://github.com/julycoding/The-Art-Of-Programming-By-July/blob/master/ebook/zh/03.01.md)。

### get 方法

分析完 put 方法，get 方法就很简单了，直接上源码：

```java
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    // 先比较 hash，再比较 key
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // first node 满足
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                // 红黑树
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do { // 链表
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

### 扩容机制

扩容（resize）就是重新计算容量，向 HashMap 对象里不停的添加元素，而 HashMap 对象内部的数组无法装载更多的元素时，对象就需要扩大数组的长度，以便能装入更多的元素。

JDK 8 对存在 hash 碰撞的情况做了优化，不需重新计算 index，只需要看看原 hash 值对应 bit（`hash & oldCap`，oldCap 是原数组的长度， 2 的幂次方）是 1 还是 0 就好了，是 0 的话索引没变，是 1 的话索引变成 `index + oldCap`。由于原 hash 值对应位置是 0 还是 1 可认为是随机的，因此 resize 的过程，均匀的把之前的冲突的节点分散到新的 bucket 了。

下面是一个容量从 16 扩充为 32 的例子：

```java
Map<Integer, String> map = new HashMap<>(16);

for (int i = 0; i < 10; i++) {
    map.put(i, Integer.toString(i));
}
map.put(16, "16");
map.put(32, "32");
// threshold = 12，需要 resize
map.put(48, "48");
```

![resize](resize.png)

resize 的源码如下：

```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 无法扩容，只能增大阈值
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE; // 修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了
                return oldTab;
            }
            // 左移一位，扩容为原来的 2 倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults（初始化的过程）
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // newThr 没有重新赋值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        // threshold 重新赋值，初始化也是在这赋值的
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; // 新建 Node 数组
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    // 释放旧 table 的对象引用，clear to let GC do its work
                    oldTab[j] = null;
                    // 只有 first node，就存在数组中
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e; // 重新计算 index
                    else if (e instanceof TreeNode) // 红黑树
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order（顺序不变）
                        // 链表，分为 low 和 high 两部分
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // hash 中对应位是0，在数组中的 index 不变
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else { // hash 中对应位是1，index 加上 oldCap
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead; // 位置不变
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead; // 位置要加上 oldCap
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

## 遍历

Map 接口包含三种映射视图，可以很方便的进行遍历：

```java
// key
Set<K> keySet();
// value
Collection<V> values();
// key-value
Set<Map.Entry<K, V>> entrySet();
```

如果只需要 key，可以遍历 keySet：

```java
    Map<Integer, String> map = new HashMap<>();
    // 省略赋值

    for (Integer key : map.keySet()) {
        System.out.println(key);
    }
```

如果只关注 vaule，可以遍历 values：

```java
    for (String value : map.values()) {
        System.out.println(value);
    }
```

需要 key-value 的话，遍历 entrySet 是最好的选择：

```java
    for (Map.Entry<Integer, String> entry : map.entrySet()) {
        System.out.println(entry.getKey());
        System.out.println(entry.getValue());
    }
```

当然，使用迭代器可以在循环中使用 remove 等方法，避免 ConcurrentModificationException：

```java
    Iterator<Map.Entry<Integer, String>> iterator = map.entrySet().iterator();
    while (iterator.hasNext()) {
        Map.Entry<Integer, String> entry = iterator.next();
        System.out.println(entry.getKey());
        System.out.println(entry.getValue());
        iterator.remove(); // avoids a ConcurrentModificationException
    }
```

在 Java 8 中可以使用 `forEach + lambda` 的方式遍历：

```java
    map.forEach((k, v) -> {
        System.out.println(k);
        System.out.println(v);
    });
```

另外，还可以使用 Stream API：

```java
    map.entrySet().stream().forEach(e -> {
        System.out.println(e.getKey());
        System.out.println(e.getValue());
    });
```

除非你还需要用到 Stream API 的一些方法，否则最好使用 `forEach + lambda` 的方式遍历。

## 推荐阅读

* [Java 8系列之重新认识HashMap](https://tech.meituan.com/java-hashmap.html)

* [HashMap的扩容机制---resize()](http://blog.csdn.net/aichuanwendang/article/details/53317351)

* [Java HashMap工作原理及实现](http://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/index.html)

* [Iterate over a Map in Java](http://www.baeldung.com/java-iterate-map)
