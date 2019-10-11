---
title: ArrayList 源码分析
date: 2018-01-02 09:41:20
tags: Java Collections
categories: Java
---

{% blockquote Documentation https://docs.oracle.com/javase/8/docs/api/java/util/ArrayList.html Java Platform SE 8 %}
Resizable-array implementation of the List interface. Implements all optional list operations, and permits all elements, including null. In addition to implementing the List interface, this class provides methods to manipulate the size of the array that is used internally to store the list. (This class is roughly equivalent to Vector, except that it is unsynchronized.)
{% endblockquote %}

这篇 note 介绍一下 Java 集合框架中的 ArrayList，并从源码的角度分析其扩容机制、迭代方法和序列化等内容。

## ArrayList 简介

想要深入学习某个 class，还是要看其源码。我这里使用 [IntelliJ IDEA](https://www.jetbrains.com/idea/) 这一款出色的 IDE 来研究 ArrayList。

ArrayList 是一个动态数组，它继承自 AbstractList 这个抽象类，并实现了 List、RandomAccess、Cloneable 以及 Serializable 接口。

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

为了避免对链表的随机访问操作，Java SE 1.4 引入了一个标记接口 RandomAccess。这个接口不包含任何方法，不过可以用它来测试一个特定的集合是否支持高效的随机访问。显然，ArrayList 支持。

```java
    if (list instanceof RandomAccess) {
        // use random access algorithm
    } else {
        // use sequential access algorithm
    }
```

<!-- more -->

ArrayList 的继承关系如下图所示。

{% qnimg ArrayList/ArrayList.png  %}

其中绿色虚线为实现接口，绿色实现代表继承接口，蓝色实线表示类的继承。

## API

### 扩容机制

我们知道，ArrayList 是一个动态数组，它在底层使用 Object 类型的定长数组存储数据，默认初始容量为10。

```java
// 数组
transient Object[] elementData;
// 默认初始容量
private static final int DEFAULT_CAPACITY = 10;
```

要实现动态长度必须进行容量的扩展。这里以 add 方法为例：

```java
public boolean add(E e) {
    // 动态扩容，size 为元素个数，而不是数组长度
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

add 方法调用 ensureCapacityInternal 进行动态扩容，add 每次加一，因此传递的参数是 size + 1。

```java
    private void ensureCapacityInternal(int minCapacity) {
        // ArrayList 为空，取 minCapacity 和默认初始长度的较大值
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
```

ensureCapacityInternal 方法首先判断数组是否为空（即通过无参构造函数创建的，而不是 `size == 0`），若为空，参数 minCapacity 取默认初始容量和它本身的较大值，然后调用 ensureExplicitCapacity：

```java
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++; // fail-fast 计数器

        // overflow-conscious code
        if (minCapacity - elementData.length > 0) // 大于数组长度，扩容
            grow(minCapacity);
    }
```

最终，grow 方法实现了扩容。正常情况下，new 一个 1.5 倍长度的新数组，使用 System.arraycopy 方法将原数组中的每一个元素拷贝到新数组。

有些虚拟机在数组中保存 header words，因此 MAX_ARRAY_SIZE 比 MAX_VALUE 略小，可参考[这里](https://stackoverflow.com/questions/35582809/java-8-arraylist-hugecapacityint-implementation)。

每次扩容都是一次数组的拷贝，如果数据量很大，这样的拷贝会非常耗费资源。推荐在集合初始化时指定初始容量。

```java
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        // 右移一位，相当于除以2向下取整，newCapacity 变为原来的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 下面四句 overflow 也会起作用
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);

        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

另外，可以显示调用ensureCapacity方法扩容，前提是 list 声明为 ArrayList。

```java
 public void ensureCapacity(int minCapacity) {
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            // any size if not default element table
            ? 0
            // larger than default for default empty table. It's already
            // supposed to be at default size.
            : DEFAULT_CAPACITY;

        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
    }
```

#### overflow-conscious code

源码中出现了令人不解的注释 `overflow-conscious code`，这到底是什么意思呢？我举两个例子：

* oldCapacity 大于 MAX_ARRAY_SIZE 的三分之二，newCapacity 溢出，但仍然会调用 hugeCapacity 方法使新的容量增大到最大值。

```java
    // add
    int oldCapacity = 1500000000; // oldCapacity = 1500000000
    int minCapacity = oldCapacity + 1; // minCapacity = 1500000001
    int newCapacity = oldCapacity + (oldCapacity >> 1); // newCapacity = -2044967296
    if (newCapacity - minCapacity < 0) // 749999999 > 0
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0) { // 102516361 > 0
        System.out.println("call hugeCapacity");
        // newCapacity = MAX_ARRAY_SIZE
    }
```

* 两个容量很大的 list，通过 addAll 方法合并，会抛出 OutOfMemoryError。

```java
    // addAll, the same size of 1500000000
    int size = 1500000000;
    int minCapacity = size + size; // minCapacity = -1294967296
    if (minCapacity - size > 0) { // 1500000000 > 0
        System.out.println("call grow");
    }
    // grow
    int oldCapacity = size; // oldCapacity = 1500000000
    int newCapacity = oldCapacity + (oldCapacity >> 1); // newCapacity = -2044967296
    if (newCapacity - minCapacity < 0) // -750000000 < 0
        newCapacity = minCapacity; // newCapacity = -1294967296
    if (newCapacity - MAX_ARRAY_SIZE > 0) { // 852516361 > 0
        System.out.println("call hugeCapacity");
        // minCapacity = -1294967296 < 0, throw new OutOfMemoryError()
    }
```

比较大小时使用 `a - b > 0` 这种方式，而不是传统的 `a > b`，这或许就是 `overflow-conscious code` 的巧妙之处吧。

### add 和 remove 方法

ArrayList 支持随机访问，get方法很快，但在 add(index) 和 remove 的时候要挪动 index 后面的元素，效率很低。因此，频繁插入和删除时应使用 LinkedList。

```java
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }

    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```

### 与数组的转换

使用集合转数组的方法，必须使用集合的 toArray(T[] a)，传入的是类型完全一致的数组，大小就是 list.size()，向下面这样：

```java
List<String> list = new ArrayList<>(3);
list.add("A");
list.add("B");
list.add("C");

String[] strings = new String[list.size()];
strings = list.toArray(strings);
```

使用 toArray(T[] a)，入参分配的数组空间不够大时，方法内部将重新分配内存空间，并返回新数组地址；如果数组元素大于实际所需，下标为 [list.size()] 的数组元素将被置为 null，其它数组元素保持原值，因此最好将方法入参数组大小定义与集合元素个数一致。源码如下：

```java
    public <T> T[] toArray(T[] a) {
        if (a.length < size)
            // Make a new array of a's runtime type, but my contents:
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        System.arraycopy(elementData, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }
```

当使用 Arrays 类中的 asList() 把数组转成集合时，不能使用修改集合相关的方法，它的 add/remove/clear 方法会抛出 UnsupportedOperationException 异常。究其原因，asList 的返回对象是一个Arrays 的内部类，继承自 AbstractList，并没有实现集合的修改方法。

```java
// 不能用修改集合的方法
List<String> stooges = Arrays.asList("Larry", "Moe", "Curly");
// 正确的打开方式
List<String> names = new ArrayList<>(Arrays.asList("Ana", "Bob", "Tom"));
```

### 遍历

查看 ArrayList 的源码，其中很多 modCount，这个变量到底有什么用呢？其实，它是在 AbstractList 中定义的变量，用于记录 list 发生结构性修改的次数。

```java
    // The number of times this list has been structurally modified
    protected transient int modCount = 0;
```

ArrayList 中无论 add、remove、clear 方法只要是涉及了改变 ArrayList 元素的个数的方法都会导致 modCount 的改变，set 方法不被视为结构性修改。ArrayList 的迭代器是 fail-fast 的，参考 [fail-fast机制](http://blog.csdn.net/chenssy/article/details/38151189)。

不要在 foreach 循环里进行元素的 remove/add 操作。remove 请使用 Iterator 方式，如果并发操作，需要对 Iterator 对象加锁。

```java
    Iterator<String> iterator = list.iterator();
    while (iterator.hasNext()) {
        String item = iterator.next();
        if ( condition ) {
            iterator.remove();
            // 不能是 list.remove(item);
        }
    }
```

但有趣的是，使用 foreach 循环删除倒数第二个元素时，不会报错，其他情况下会产生 ConcurrentModificationException。这是因为每次循环会调用迭代器的 hasNext 方法，删除倒数第二个元素再次判断时，hasNext 返回 false，直接退出循环。

```java
    List<String> list = new ArrayList<>();
    list.add("1");
    list.add("2");
    list.add("3");
    list.add("4");
    // hasNext ?
    for (String item : list) {
        if ("3".equals(item)) {
            list.remove(item); // 这里不会报错
        }
    }
```

另外，ArrayList 中还有 ListIterator 的实现，提供了反向遍历的功能，具体参考 [Interface ListIterator](https://docs.oracle.com/javase/8/docs/api/)。

### 序列化

ArrayList 实现了 Serializable 接口，表明其可以被序列化，但是保存所有元素的 elementData 数组却是 transient 类型的，并且源码中还有两个诡异的 private 方法与序列化有关，一个 writeObject, 一个 readObject。什么鬼？

原来，这两个方法是通过 Java 的反射原理来定制序列化和反序列化策略的。有时数组中的元素很少但 `elementData.length` 很大，这样在序列化过程中会包含很多的 null 值，因此 writeObject 和 readObject 中都有一个 `0~size` 的 for 循环，以去掉多余的 null。更详细的介绍请参考[深入分析Java的序列化与反序列化](http://www.hollischuang.com/archives/1140)。

```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }
```

### SubList

ArrayList 的 subList 结果不可强转成 ArrayList，否则会抛出 ClassCastException 异常。subList 返回的是 ArrayList 的内部类 SubList，并不是 ArrayList ，而是 ArrayList 的一个视图，对于 SubList 子列表的所有操作最终会反映到原列表上。

在 subList 场景中，高度注意对原集合元素个数的修改，会导致子列表的遍历、增加、删除均会产生 ConcurrentModificationException 异常，因此在生成子列表后不要再操作原列表。

## 推荐阅读

* [阿里巴巴Java开发手册（纪念版）](https://github.com/alibaba/p3c)

* [为什么Java里的Arrays.asList不能用add和remove方法？](http://blog.csdn.net/loveaborn/article/details/39754031)

* [Why does ArrayList use transient storage?](https://stackoverflow.com/questions/9848129/why-does-arraylist-use-transient-storage)

* [深入分析Java的序列化与反序列化](http://www.hollischuang.com/archives/1140)

* [Fail Fast Vs Fail Safe Iterator In Java : Java Developer Interview Questions](http://javahungry.blogspot.com/2014/04/fail-fast-iterator-vs-fail-safe-iterator-difference-with-example-in-java.html)

* [Java提高篇（三四）—— fail-fast机制](http://blog.csdn.net/chenssy/article/details/38151189)

* [Top 10 Mistakes Java Developers Make](https://www.programcreek.com/2014/05/top-10-mistakes-java-developers-make/)

* [How does Enhanced for loop works in Java?](http://javarevisited.blogspot.com/2016/02/how-does-enhanced-for-loop-works-in-java.html)

* 秦小波. 编写高质量代码:改善Java程序的151个建议[M]. 机械工业出版社, 2012.

* Java核心技术 卷I：基础知识（原书第10版）