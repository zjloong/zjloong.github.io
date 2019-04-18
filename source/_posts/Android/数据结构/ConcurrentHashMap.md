---
title: ConcurrentHashMap
top: true
cover: true
categories: Android
tags:
  - ConcurrentHashMap
date: 2019-04-16 19:33:26
img:
coverImg:
summary: 
---

HashMap不支持并发, 在多线程的情况下, 推荐使用ConcurrentHashMap. ConcurrentHashMap在JDK8之后采用了CAS和synchronized来保证多线程下的数据安全性.

### CAS
- 什么是CAS
全称 **Compare And Swap**, 翻译就是比较替换. 其依靠硬件实现, 通过CPU指令比较某个内存地址中的值和预期值, 如果相等, 则将该地址的值替换为某个新值. 由于CAS操作是原子性的, 所以在多线程中通过CAS更新数据时无需加锁, 这样可以提升效率. 但是java中是无法直接操作内存(指针)的, 所以又通过JNI封装了一个Unsafe类, 提供了一些内存操作的方法. 源码中的很多地方都用到了Unsafe, 比如 FutureTask, AtomicInteger, ConCurrentHashMap,... 可以在保证并发安全的同时, 又可以提升效率.

- UnSafe
正如它的名字一样, 直接操作内存是十分危险的, 所以java也是不希望开发者直接使用这个类的. Unsafe使用了单例模式, 其构造方法是私有的, 但是提供了一个获取实例的静态方法:
```java
@CallerSensitive
public static Unsafe getUnsafe() {
    Class var0 = Reflection.getCallerClass();
    // 判断调用者的类加载器是否是系统核心加载器 -- Bootstrap加载器
    if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
        throw new SecurityException("Unsafe");
    } else {
        return theUnsafe;
    }
}
```

因此正常情况下开发者是无法得到UnSafe实例的, 不过我们还是可以通过反射获取:
```java
public static Unsafe getUnsafe(){
    try {
        // 获取 theUnsafe 字段
        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        // 因为 theUnsafe 是静态属性, 所以参数传 null
        return (Unsafe) f.get(null);
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}
```

下面来了解一下 UnSafe 提供了哪些功能:
```java
// 从对象 o 的内存地址偏移量 offset 处读取一个 int值   
public native int getInt(Object o, long offset);
// 将 ... 设置为 x
public native void putInt(Object o, long offset, int x);
...
public native Object getObject(Object o, long offset);
public native void putObject(Object o, long offset, Object x);
...
// 直接在某个地址处读写
public native int getInt(long offset);
public native void putInt(long offset, int x);
...
// 读写(保证 有序性 和 可见性)
public native int getIntVolatile(Object o, long offset);
public native void putIntVolatile(Object o, long offset, int x);
...
// 有序写入
public native void putOrderedInt(Object o, long offset, int x);
// 分配内存
public native long allocateMemory(long offset);
// 重新分配内存
public native long reallocateMemory(long offset, long var3);
// 初始化内存
public native void setMemory(long offset, long var3, byte var5);
// 内存复制
public native void copyMemory(Object var1, long var2, Object var4, long var5, long var7);
// 释放内存
public native void freeMemory(long offset);
...
// 获取静态属性 Field 相对于其所属类的内存地址偏移
public native long staticFieldOffset(Field var1);
// 获取非静态属性Field相对其所属对象的偏移
public native long objectFieldOffset(Field var1);
// 获取Field所属的对象
public native Object staticFieldBase(Field var1);
// 获取数组中第一个元素实际地址相对整个数组内存地址的偏移量
public native int arrayBaseOffset(Class<?> var1);
// 获取数组中元素的地址增量
public native int arrayIndexScale(Class<?> var1);
...
// 引用类型的CAS操作 (比较对象 o 的内存地址偏移量 offset 处的值和 expected, 如果相等, 则将该处的值更新为 x, 并返回 true) ***
public final native boolean compareAndSwapObject(Object o, long offset,  Object expected,  Object x);
// int类型的CAS操作  ***
public final native boolean compareAndSwapInt(Object o, long offset, int expected, int x);
// long类型的CAS操作  ***
public final native boolean compareAndSwapLong(Object o, long offset,long expected,long x);
...
// 使用自旋的方式进行CAS操作（while循环进行CAS操作, 直到操作成功） ***
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        // 获取对象内存地址偏移量上的数值v
        v = getIntVolatile(o, offset);
        // 如果现在还是v,设置为 v + delta,否则返回false,继续循环再次重试.
    } while (!compareAndSwapInt(o, offset, v, v + delta));
    return v;
}
...
// 挂起线程(线程被阻塞, 直到被重写唤起), LockSupport#park 内部就是通过此方法实现
public native void park(boolean var1, long var2);
// 唤起线程, LockSupport#unpark 内部就是通过此方法实现
public native void unpark(Object var1);
...
// 动态一个创建类
public native Class<?> defineClass(String var1, byte[] var2, int var3, int var4, ClassLoader var5, ProtectionDomain var6);
// 动态创建一个匿名内部类
public native Class<?> defineAnonymousClass(Class<?> var1, byte[] var2, Object[] var3);
// 创建一个类的实例
public native Object allocateInstance(Class<?> var1) throws InstantiationException;
// 判断是否需要初始化一个类
public native boolean shouldBeInitialized(Class<?> var1);
// 确保某个类已经被初始化
public native void ensureClassInitialized(Class<?> var1);
...
// 保证在这个屏障之前的所有读操作都已经完成
public native void loadFence();
// 保证在这个屏障之前的所有写操作都已经完成
public native void storeFence();
// 保证在这个屏障之前的所有读写操作都已经完成
public native void fullFence();
```

### ConcurrentHashMap的成员变量
```java
// 实际保存数据的数组, volatile保证数据的可见性
transient volatile Node<K,V>[] table;
// 在扩容过程中作为过渡使用的数组
private transient volatile Node<K,V>[] nextTable;
// ConCurrentHashMap为了保证多线程下计数的准确性和效率, 使用计数数组counterCells和baseCount共同记录元素的个数
// 在没有线程争抢的情况下, 优先使用baseCount计数; 在CAS更新baseCount失败的情况下, 会根据当前线程指定counterCells中的某一个位置进行计数
// 最后在获取元素的总个数时, 将这些数据进行累加
private transient volatile long baseCount;
/**
 * 非常重要
 * ① 当 sizeCtl > 0 时, 其值可能有两种种情况
 *      - 数组容量未初始化, sizeCtl表示初始容量
 *      - 数组容量已初始化, sizeCtl表示扩容的阈值 -> 数组长度 * 0.75
 * ② 当 sizeCtl == -1 时: 表示当前正在初始化容量, 或扩容
 * ③ 当 sizeCtl < -1 时: sizeCtl的高16位存储扩容标识, 低16位则等于参数扩容的线程数 +1    
 */
private transient volatile int sizeCtl;
// 平时为0, 在扩容时用来给当前线程分配迁移范围
private transient volatile int transferIndex;
private transient volatile CounterCell[] counterCells;
```

### put方法
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
...
// onlyIfAbsent == true时, 不会覆盖相同key对应的value
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // key, value 都不允许为null
    if (key == null || value == null) throw new NullPointerException();
    // 获取扰动后的hash值
    int hash = spread(key.hashCode());
    int binCount = 0;
    // 自旋(无限循环)
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果数组为空, 则通过 initTable 初始化数组
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 根据hash得到索引i, 并取出索引i处的元素 (tabAt 中使用了CAS -> UnSafe.getObjectVolatile)
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {                 
            // 索引i处为空, 则通过 casTabAt 直接添加新的 Node  (casTabAt -> UnSafe.compareAndSwapObject)
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))    
                break;                   
        }
        // hash == MOVED. 说明该元素是 ForwardingNode, 表示正在扩容, 但该位置已经迁移完毕
        // ConcurrentHashMap在扩容时, 如果某个位置的元素迁移完毕, 则会在旧数组的该位置插入 ForwardingNode节点, 其hash值就是常量MOVED
        else if ((fh = f.hash) == MOVED)  
            // 帮助其它位置扩容
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 锁住数组上位置 i 处的元素
            synchronized (f) {
                if (tabAt(tab, i) == f) {                       // 再次比较一下, 确保数据还没有被修改
                    if (fh >= 0) {                              // hash >= 0  表示当前桶结构为链表结构
                        binCount = 1;                           // 记录链表上节点的个数
                        for (Node<K,V> e = f;; ++binCount) {    // 循环整个链表
                            K ek;
                            // hash值相等, 且key也相等
                            if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                // onlyIfAbsent为false, 直接替换value
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 遍历到了链表的末尾还没有找到匹配的节点. 则创建新节点加到链表的尾部
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    } else if (f instanceof TreeBin) {    // 红黑树
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    } else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            // binCount != 0 表示 当前桶发生了变化
            if (binCount != 0) {
                // 超过了限定值, 将链表转换为红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 元素个数 +1, 检查是否需要扩容
    addCount(1L, binCount);
    return null;
}
```

### 数组初始化
```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // 自旋
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)   // sizeCtl小于0, 表示有其它线程正在初始化或扩容
            Thread.yield();       // 当前线程让出CPU
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {   // CAS. 将 sizeCtl 赋值为 -1
            try {
                if ((tab = table) == null || tab.length == 0) {  // 再次检查数组是否初始化
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;    // 使用 sizeCtl 为初始化容量. sc就是被 CAS 之前的 sizeCtl
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);                          // 初始化完成后, 设置 sc 为 0.75n, 作为新的扩容阈值
                }
            } finally {
                sizeCtl = sc;                                    // 赋值 sizeCtl
            }
            break;
        }
    }
    return tab;
}
```

### 扩容
- addCount -- 元素计数 + 检查扩容
```java
// x 增加元素的个数;  check >= 0 时检查是否需要扩容
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // counterCells是一个计数数组, 保存了各个线程的计数, 不同的线程可以在不同的计数单元上进行计数, 减少冲突, 提高效率
    // baseCount则是一个优先使用的公共计数, 在更新baseCount失败的情况下, 会使用counterCells计数
    // counterCells 不为null, 或 CAS 修改 baseCount 失败
    if ((as = counterCells) != null || !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||                                      // counterCells为null, 或长度为 0
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||                         // 从counterCells数组随机取一个位置为null
            !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {    // CAS修改CounterCell的value失败
            // baseCount和counterCells都更新失败了, 则强制进入死循环增加 counterCells 计数
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        // 计算元素个数  s = baseCount + counterCells计数总和
        s = sumCount();
    }
    // 检查是否需要扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 正常情况下, sizeCtl 存储的是扩容的阈值; 如果正在进行扩容, 则 sizeCtl 的高16位存储扩容标志, 低16位存储扩容的线程数+1
        // (元素总数大于sizeCtl | 且数组不为null | 且数组长度没到最大值) 表示需要扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null && (n = tab.length) < MAXIMUM_CAPACITY) {
            // rs作为扩容的一个标志
            int rs = resizeStamp(n);
            if (sc < 0) {   // sc < 0 表示正在扩容
                // 扩容已完成(sc的高16位不等于rs | 扩容线程数为0 | 扩容线程达到了最大 | nextTable为null | 数组扩容区域已分配完), 退出循环
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null || transferIndex <= 0)
                    break;
                // 扩容未完成, 当前线程也开始扩容, 并将 sizeCtl 中线程数 +1
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            } else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))  // 高16位存储扩容标志, 低16位表示扩容线程数+1
                // 表示没有其它线程正在扩容, 这里直接开始扩容
                transfer(tab, null);
            // 计算元素个数
            s = sumCount();
        }
    }
}
```

- transfer -- 扩容, 迁移元素
```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // NCPU指CPU的核心数. NCPU = Runtime.getRuntime().availableProcessors()   
    // stride指本次需要迁移的桶的数量. (多核: 1/8 * n / NCPU;  单核: n)
    // stride 最小为 16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; 
    // 初始化nextTab, 作为扩容临时数组使用
    if (nextTab == null) {            
        try {
            // 数组长度为原来的两倍
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        // 更新成员变量 nextTable
        nextTable = nextTab;
        // transferIndex为旧数组长度
        transferIndex = n;
    }
    // 新数组的长度
    int nextn = nextTab.length;
    // 新建一个ForwardingNode类型的节点，将新数组存储在里面
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // 标记是否继续递减
    boolean advance = true;  
    // 标记整个扩容是否完毕        
    boolean finishing = false; 
    // 死循环. i表示要迁移的桶的下标; bound表示当前线程需要迁移的桶区间范围对应的最小下标      
    for (int i = 0, bound = 0;;) {   
        Node<K,V> f; int fh;
        /**
         * while循环有两个作用: 
         *    1. 如果当前线程未分配迁移范围, 或其负责的范围已迁移完毕, 则为其分配新的迁移范围;
         *    2. 如果已分配迁移范围, 且还未迁移完毕, while循环的作用就是将索引从该范围的最大值递减推进到最小值
         */
        while (advance) {
            int nextIndex, nextBound;
            /**
             * while每循环一次都会执行一次 --i, 表示将下标递减
             *    --i >= bound 表示当前线程所负责的范围还没有迁移完毕 (默认第一次进来时该条件也不成立, 因为还没有分配范围)
             *    finishing == true 表示整个扩容都结束了
             */
            if (--i >= bound || finishing)
                // 如果是扩容结束了, 需要将 advance 设置为 false, 结束 while循环
                // 如果是当前线程所负责的范围还没有迁移完毕, 也要将  advance 设置为 false, 暂时跳出while循环, 进入后面的迁移步骤
                advance = false;
            // 走到这里, 表示当前线程没有分配迁移范围, 或者其负责的范围都已迁移完成了
            // transferIndex <= 0 表示整个要迁移的数组都分配完毕, 没有新的范围可分配了
            else if ((nextIndex = transferIndex) <= 0) {
                // 将 i 设置为 -1, ,即 i < 0, 会满足下面 if 判断的第一个分支, 标识扩容区域已分配完
                i = -1;
                // 跳出while循环
                advance = false;
            }
            // CAS将transferIndex更新为 nextIndex - stride (在前面已经设置 nextIndex 等于 transferIndex). CAS更新成功, 则开始分配范围
            else if (U.compareAndSwapInt (this, TRANSFERINDEX, nextIndex, nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
                // 分配给当前线程的迁移范围对应的 最小下标
                bound = nextBound;
                // 从最大下标开始递减迁移
                i = nextIndex - 1;
                // 设置 advance 为 false, 暂时跳出循环, 进入后面的迁移步骤
                advance = false;
            }
        }
        // 这个判断表示以下几种情况: 
        // i < 0 表示扩容区域已经分配完, 当前线程不需要再参与扩容
        // i >= n 和 i + n >> nextn 都表示i超出了旧数组范围
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {                       // 如果整个扩容都完成了
                nextTable = null;                  // 将nextTable设置为null
                table = nextTab;                   // 更新 table
                sizeCtl = (n << 1) - (n >>> 1);    // 将sizeCtl设置为 1.5n, 也就是 新数组长度 * 0.75
                return;                            // 退出
            }
            // 表示本线程结束扩容, 线程数 -1
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 表示有其它还有其它线程正在扩容, 直接退出
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                // 没有其它线程在扩容了, 标记整个扩容结束了
                finishing = advance = true;
                // 将 i 重新赋值为n, 这样会重新遍历一遍, 检查是否迁移完成
                i = n; 
            }
        } 
        // 如果要迁移的位置为null, 则插入 ForwardingNode, 表示该位置迁移完毕
        else if ((f = tabAt(tab, i)) == null)      
            advance = casTabAt(tab, i, null, fwd);
        // 如果该位置已经是 ForwardingNode 了, 表示不需要处理, 继续下一次循环
        else if ((fh = f.hash) == MOVED)
            // advance = true.  继续上面的循环 --i
            advance = true; 
        else {
            // 加锁
            synchronized (f) {
                // 再次判断一下, 保证数据没有被修改
                if (tabAt(tab, i) == f) {
                    /**
                     * 因为数组的长度被限制为2的指数倍, 而通过hash计算index又相当于一个取模的过程, 所以在数组扩大到2倍时, 每个节点的下标只会有两种可能;
                     *      ① 如果 hash & n == 0,  下标不变 (n是旧数组的长度)
                     *      ② 如果 hash & n == 1,  新下标 = 旧下标 + n
                     * 所以思路就是将原来的桶按照这个规则拆分为两个;
                     * 关于上面的这个结论, 可以拿出草稿纸比划一下基本就能清楚了;
                     */
                    Node<K,V> ln, hn;
                    if (fh >= 0) {                                         // hash > 0 , 表示为链表结构
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        // 这个循环的作用是取出链表上最后一段位置都要变(或都不变)的子链
                        // 不知道为什么要多做这一步操作, 觉得并没有提升效率...
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {                               // 子链lastRun位置不变, 保存到 ln 中
                            ln = lastRun;
                            hn = null;
                        } else {                                         // 子链lastRun位置 +n, 保存到 hn 中
                            hn = lastRun;
                            ln = null;
                        }
                        // 再次循环, 将链表拆分为两个. 注意值循环到 lastRun, 因为子链lastRun上都是一样的了
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                // 位置不变的元素, 采用 头插 的方式加到 ln 链上
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                // 位置 +n的元素, 采用 头插 的方式加到 hn 链上
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 链表 ln 中都是位置不变的节点, 插入到新数组相同的下标处
                        setTabAt(nextTab, i, ln);
                        // 链表 hn 中都是位置 +n的节点, 插入到新数组对应的下标处
                        setTabAt(nextTab, i + n, hn);
                        // 在旧数组上插入一个 ForwardingNode 节点, 表示该位置迁移完毕了
                        setTabAt(tab, i, fwd);
                        // 设置 advance 为 true.  继续上面的循环 --i
                        advance = true;
                    } else if (f instanceof TreeBin) {                       // 红黑树
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
                }
            }
        }
    }
}
```

- helpTransfer -- 帮助扩容
```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 如果 tab 不为null, 且当前位置元素为ForwardingNode(表示当前位置已迁移完毕), 且ForwardingNode的nextTab不为空. 则帮助其它位置迁移
    if (tab != null && (f instanceof ForwardingNode) && (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        // 扩容标志
        int rs = resizeStamp(tab.length);
        // 判断正在扩容
        while (nextTab == nextTable && table == tab && (sc = sizeCtl) < 0) {
            // 扩容已完成(sc的高16位不等于rs | 扩容线程数为0 | 扩容线程达到了最大 | nextTable为null | 数组扩容区域已分配完), 退出循环
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 扩容未完成, 当前线程也开始扩容, 并将 sizeCtl 中线程数 +1
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                // 开始扩容
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

### get方法
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 获取扰动后的 hash
    int h = spread(key.hashCode());
    // 数组不为空, 且对应的下标处也不为null
    if ((tab = table) != null && (n = tab.length) > 0 && (e = tabAt(tab, (n - 1) & h)) != null) {
        // hash相等
        if ((eh = e.hash) == h) {
            // key 也相等
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                // 返回 value
                return e.val;
        }
        // 红黑树
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 链表. 从链头开始循环
        while ((e = e.next) != null) {
            // hash相等 & key也相等, 返回 value
            if (e.hash == h && ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    // 没找到
    return null;
}
```

### size方法
```java
public int size() {
    long n = sumCount();
    // 边界判断
    return ((n < 0L) ? 0 : (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE : (int)n);
}
...
// 返回计数数组的总和再加上baseCount
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
