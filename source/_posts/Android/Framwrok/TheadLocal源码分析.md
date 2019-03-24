---
title: TheadLocal源码分析
top: false
cover: false
categories: Android
tags:
  - ThreadLocal
date: 2019-03-24 10:40:36
img:
coverImg:
summary: 
---

默认情况下, 多线程间数据是共享的, Java中也提供了相关的同步机制用于多线程数据共享造成的数据安全性问题. 同时考虑到某些需要线程间数据隔离的场景, 也提供了ThreadLocal让我们可以管理属于线程私有的数据.

### ThreadLocal的用法
```java
new Thread(() -> {
    ThreadLocal<String> threadLocal = new ThreadLocal<>();
    // 保存
    threadLocal.set("siyueyihao");
    // 读取  ---  siyueyihao
    Log.e(">>>", threadLocal.get());
    new Thread(() -> {
        // null
        Log.e(">>>", threadLocal.get());
    }).start();
    // 删除
    threadLocal.remove();
}).start();
```

可以发现, 通过 ThreadLocal 的 set() 方法保存的数据, 只有在当前线程才能访问. 下面就进入到 ThreadLocal 来了解一下它的工作原理.

### ThreadLocal的原理
- ThreadLocal的set方法
```java
public void set(T value) {
    // 得到当前线程
    Thread t = Thread.currentThread();
    // 通过当前线程实例, 获得 ThreadLocalMap对象
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 如果 ThreadLocalMap 不为空, 则以当前ThreadLocal实例为Key, 要保存的数据为value, 存储到 Map 中
        map.set(this, value);
    else
        // 如果 ThreadLocalMap 为空, 就新建ThreadLocalMap
        createMap(t, value);
}
```

看看 getMap 和 createMap 里面做了什么:
```java
ThreadLocalMap getMap(Thread t) {
    // 发现 ThreadLocalMap 其实是保存在 Thread中的
    return t.threadLocals;
}
...
void createMap(Thread t, T firstValue) {
    // 创建ThreadLocalMap实例, 并以当前ThreadLocal实例为Key, 要保存的数据为value, 存储到 Map 中
    // 然后将该实例赋值给 Thread 的 threadLocals变量.
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

通过源码可以知道, 我们要保存的数据会和当前ThreadLocal实例组成键值对, 然后保存在 Thread 的成员变量 ThreadLocalMap中的. 

- ThreadLocal的get方法
```java
public T get() {
    // 获取当前线程的实例
    Thread t = Thread.currentThread();
    // 获取线程实例中的 ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 从 map 中获取Key为当前ThreadLocal实例, 的 ThreadLocalMap.Entry 对象
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            // 返回 ThreadLocalMap.Entry 对象 的 value
            T result = (T)e.value;
            return result;
        }
    }
    // map为空 或 Entry为空, 则返回此方法的返回值
    return setInitialValue();
}
...
private T setInitialValue() {
    T value = initialValue();
    //////////////////这部分的代码, 和 set方法一样/////////////////////////
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    ///////////////////////////////////////////////////////////////////
    return value;
}
...
protected T initialValue() {
    // 默认为 null, 不过是 protected 方法, 可以由子类实现
    return null;
}
```

发现整个过程就是尝试从 ThreadLocalMap 取值. 如果没有取到值, 则通过 initialValue() 获取一个默认值, 并将其保存到ThreadLocalMap中, 最后返回该默认值.

- ThreadLocal的remove方法
```java
public void remove() {
   ThreadLocalMap m = getMap(Thread.currentThread());
   if (m != null)
       // 调用了 ThreadLocalMap 的 remove 方法
       m.remove(this);
}
```

通过上面的分析, 可以发现整个逻辑的核心在于 ThreadLocalMap. 因此 ThreadLocalMap 才是我们要了解的重点.

### ThreadLocalMap
ThreadLocalMap 是 ThreadLocal的静态内部类. 其内部维护了一个 Entry数组, 用来存储数据.
```java
// 长度必须是2的幂次方
private Entry[] table;
...
// Entry 继承自 WeakReference
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        // key为 ThreadLocal
        // 注意: 其对key的引用关系为 弱引用. 当该ThreadLocal实例的强引用计数为0时, 可能被GC回收掉, 导致Entry的key为null.
        // 当key为null时, 此时Entry的value也就无法被访问到, 如果当前线程又没有销毁(仍然引用ThreadLocalMap), 此时会导致 内存泄漏
        // 所以, 当我们使用 ThreadLocal保存数据后, 一定要记得及时 remove
        super(k);
        // value的引用
        value = v;
    }
}
```

了解了ThreadLocalMap中是通过 Entry来保存数据后, 下面再看看 ThreadLocalMap是如何管理数据的.

#### ThreadLocalMap的set方法
```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    // 根据 ThreadLocal的 threadLocalHashCode 以及 Entry数组的长度, 计算新数据将要保存在Entry数组中的位置
    // 在 len 的长度是2的幂次方时, 这一步计算等同于:  key.threadLocalHashCode % len
    int i = key.threadLocalHashCode & (len-1);
    // "碰撞检测" 的过程. 从索引i开始循环数组, 判断当前位置是否适合用来保存数据, 否则就通过 nextIndex() 计算一个新的索引
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // 表示该索引处已经有了一个Entry实例, 且它的key和当前key一样, 则直接修改 Entry的value 
        if (k == key) {
            e.value = value;
            return;
        }
        // 表示该索引处已经有了一个Entry实例, 但它的key为null, 则将其 key, value 覆盖掉
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 表示没有冲突的 Entry, 则新建 Entry实例, 直接插入到数组的 i 处
    tab[i] = new Entry(key, value);
    int sz = ++size;
    // 清除 key 为null 的 Entry, 并且如果发现长度大于阈值, 则对数组进行扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        // 数组扩容
        rehash();
}
```
 
先看下 ThreadLocal 的 threadLocalHashCode是什么
```java
private final int threadLocalHashCode = nextHashCode();
private static AtomicInteger nextHashCode = new AtomicInteger();
private static final int HASH_INCREMENT = 0x61c88647;
// threadLocalHashCode 通过 nextHashCode 计算, 每调用一次, 增加 HASH_INCREMENT
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

再看下 ThreadLocalMap是如何确定将 Entry 插入到数组中的位置
```java
// 默认为 key.threadLocalHashCode % len
int i = key.threadLocalHashCode & (len-1);
// 遇到冲突时, 通过 nextIndex 方法计算新的索引
private static int nextIndex(int i, int len) {
    // 从 i 开始向后查找, 到达数组末尾则又从 0 开始
    return ((i + 1 < len) ? i + 1 : 0);
}
```

Entry数组的长度:
```java
// 数组默认长度 16
private static final int INITIAL_CAPACITY = 16;
private int threshold;
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    // 根据默认长度创建数组
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    // 设置阈值, 当数组中的数据量达到这个值, 则进行扩容
    setThreshold(INITIAL_CAPACITY);
}
```

ThreadLocalMap 如何对Entry数组进行扩容:
```java
// 当数组中的数据量达到阈值, 则调用此方法进行扩容
private void rehash() {
    // 清理 key 为 null 的 Entry
    expungeStaleEntries();
    if (size >= threshold - threshold / 4)
        resize();
}
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    // 长度扩大一倍
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;
    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                // 循环变量, key为null时, 将 value也设置为null
                e.value = null; // Help the GC
            } else {
                // 将原数组的数据, 迁移到新数组
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }
    // 更新阈值
    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

#### ThreadLocalMap的getEntry方法
```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        // Entry不为null, 且其 key 也不为null, 则直接返回
        return e;
    else
        // 否则就从索引 i 开始, 依次向后进行查找
        return getEntryAfterMiss(key, i, e);
}
...
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            // 找到了, 直接返回
            return e;
        if (k == null)
            // Entry 的key为null, 清空该数据
            expungeStaleEntry(i);
        else
            // 没找到, 计算新的索引
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

理解了set方法后, getEntry方法的逻辑还是挺简单的. 最后再看下 remove方法

#### ThreadLocalMap的remove方法
```java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    // 计算索引
    int i = key.threadLocalHashCode & (len-1);
    // 和set方法中一样的循环方式
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            // 找到了, 将数据清除掉
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

完!