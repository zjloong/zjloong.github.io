---
title: HashMap
top: true
cover: true
categories: Android
tags:
  - HashMap
date: 2019-04-04 10:45:45
img:
coverImg:
summary: HashMap作为日常开发中用的最多的集合之一, 非常有必要对它的内部原理进行一下了解, 同时源码里的一些写法也是非常值得学习的.
---

HashMap作为日常开发中用的最多的集合之一, 非常有必要对它的内部原理进行一下了解, 同时源码里的一些写法也是非常值得学习的.

### HashMap中的一些常量和变量
```java
// 默认初始化容量 16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
// 默认最大容量 2的30次方
static final int MAXIMUM_CAPACITY = 1 << 30;
// 默认 负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// hash碰撞时, 生成的链表长度超过这个值就转为红黑树结构
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;

// map中的键值对会转换为 Node保存在数组中
transient Node<K,V>[] table;
transient Set<Map.Entry<K,V>> entrySet;
// 键值对的个数
transient int size;
// map内部数据结构发生变化的次数
transient int modCount;
// 阈值 -- 当map中的键值对个数超过这个值, 就需要扩容(resize)
int threshold;
// 负载因子
final float loadFactor;
```

在向HashMap中存储键值对时, 实际上保存的是一个函数key, value属性的Node对象

### 节点Node
```java
static class Node<K,V> implements Map.Entry<K,V> {
    // Node节点的hash
    final int hash;
    // key
    final K key;
    // value
    V value;
    // 指向下一个节点 (单向链表)
    Node<K,V> next;
    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }
    // key的hashCode 异或 value的hashCode
    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }
    // 设置value, 并返回旧的value
    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }
    // 比较是否相等 -- key和value都相等
    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

### 构造方法
```java
public HashMap() {
    // 负载因子取默认常量 0.75
    this.loadFactor = DEFAULT_LOAD_FACTOR; 
}
// 初始化时指定初始容量. 容量建议为 2的指数倍
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
// 初始化时指定初始容量 和 负载因子
public HashMap(int initialCapacity, float loadFactor) {
    // 不允许小于0
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    // 不允许超过 2的30次方
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
    this.loadFactor = loadFactor;
    // 根据初始容量, 计算当前阈值
    this.threshold = tableSizeFor(initialCapacity);
}
...
// 该方法返回值: 大于或等于 cap, 且最接近 cap 的 2的指数倍
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    // 边界判断
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

### 增加

#### put & putIfAbsent
```java
public V put(K key, V value) {
    // 根据key计算出一个hash值
    return putVal(hash(key), key, value, false, true);
}
...
// 保存数据, 但不会覆盖相同key对应的value
public V putIfAbsent(K key, V value) {
    return putVal(hash(key), key, value, true, true);
}
...
static final int hash(Object key) {
    int h;
    // 用原hashCode的低16位异或其高16位, 作为新的低16位, 这样综合了原hashCode的高位和低位, 减少hash碰撞
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
...
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 当数组为null, 或者长度为0时, 直接通过 resize 方法扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 根据hash和当前数组的长度, 确定其将位于数组的位置, 如果该位置没有数据, 则直接创建新节点保存
    // 在 n 的值是2的指数倍时, (n - 1) & hash  等价于  hash % n
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // 该位置已经有数据了, 说明发生了hash碰撞
        Node<K,V> e; K k;
        // 如果hash相等, 且key也相等, 则用变量e记录一下, 后面可以直接对其替换value
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)     // 如果是红黑树结构...(红黑树比较难, 略过)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 从链头开始遍历链表
            for (int binCount = 0; ; ++binCount) {
                // 如果遍历到结尾都没有相等的节点, 那就将新节点添加到链尾
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 如果长度超过了指定的值, 就转换为红黑树结构
                    if (binCount >= TREEIFY_THRESHOLD - 1) 
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果hash相等, 且key也相等, 则用变量e记录一下, 后面可以直接对其替换value
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 如果e不为null, 表示存在hash和key都相等的节点, 可以替换其value
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 只有onlyIfAbsent为false时, 才会替换
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 空方法, 由其子类 LinkedHashMap
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 表示插入了新数据, 用modCount记录一下数据结构发生变化的次数
    ++modCount;
    // 如果键值对个数超出了阈值, 则需要扩容
    if (++size > threshold)
        resize();
    // 空方法, 由其子类 LinkedHashMap
    afterNodeInsertion(evict);
    return null;
}
```

#### putAll
```java
public void putAll(Map<? extends K, ? extends V> m) {
    putMapEntries(m, true);
}
...
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        if (table == null) { // pre-size
            // 计算阈值
            float ft = ((float)s / loadFactor) + 1.0F;
            // 阈值不允许超过 2的30次方
            int t = ((ft < (float)MAXIMUM_CAPACITY) ? (int)ft : MAXIMUM_CAPACITY);
            // 如果新的阈值大于当前的阈值, 则根据新阈值重新计算
            if (t > threshold)
                threshold = tableSizeFor(t);
        } else if (s > threshold)
            // 扩容
            resize();
        // 遍历, 将数据一个一个通过 putVal添加进来
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

#### resize
```java
final Node<K,V>[] resize() {
    // 使用变量 oldTab 保存当前数组
    Node<K,V>[] oldTab = table;
    // 当前数组的长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 保存当前的阈值
    int oldThr = threshold;
    // 这两个变量用来保存新的容量和阈值
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 如果数组长度已经超过了最大范围, 则不再进行扩容
            threshold = Integer.MAX_VALUE;
            return oldTab;
        } else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 数组长度扩大一倍. 如果扩容后还小于最大范围, 且之前的容量大于或等于16, 则阈值也扩大一倍
            newThr = oldThr << 1; // double threshold
    } else if (oldThr > 0) // initial capacity was placed in threshold
        // 如果数组是空的, 但阈值大于0. 表示初始化时指定了初始化容量, 那么新的容量就等于阈值
        newCap = oldThr;
    else {               
        // 数组为空, 且阈值也未0. 那么就将容量设置为16, 阈值设置为 16 * 0.75
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // newThr == 0 表示当前数组为空, 但阈值大于0
    if (newThr == 0) {
        // 根据新的容量和负载因子计算新的阈值
        float ft = (float)newCap * loadFactor;
        // 阈值不允许超出范围
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }
    // 更新阈值
    threshold = newThr;
    // 根据新的容量, 构建新的数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    // 更新数组
    table = newTab;
    if (oldTab != null) {
        // 将旧数组的数据转移到新数组中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                // 就数组中的数据置空, 便于回收
                oldTab[j] = null;
                if (e.next == null)
                    // 如果该位置只有一个数据, 则计算出新的下标, 并直接保存即可
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 红黑树...
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    /*链表结构. 
                     * 因为Node在新数组中的下标只会有两种情况: ①不变; ② 旧下标 + 扩容的长度 => 旧下标 + 旧数组长度(因为数组是翻倍扩容)
                     * 所以这里用两个链表分别保存可能存在的两种情况
                     */
                    // 下标不变的链表的 头 和 尾
                    Node<K,V> loHead = null, loTail = null;
                    // 下标改变的链表的 头 和 尾
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {            // 表示下标不变
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        } else {                                 // 表示下标 = 旧下标 + 旧数组长度
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;                      // 将下标不变化的链表的头放到原位置处
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;             // 将下标要改变的链表的头放到新位置处
                    }
                }
            }
        }
    }
    return newTab;
}
```

#### replace
```java
public V replace(K key, V value) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) != null) {
        V oldValue = e.value;
        e.value = value;
        // 空方法, 由其子类 LinkedHashMap
        afterNodeAccess(e);
        return oldValue;
    }
    return null;
}
```

### 删除

#### remove
```java
// 如果key对应的value存在，则删除这个键值对, 并返回value; 否则返回null
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ? null : e.value;
}
...
public boolean remove(Object key, Object value) {
    return removeNode(hash(key), key, value, true, true) != null;
}
...
// 当 matchValue 为true时, 必须key、value都相等时才删除节点
final Node<K,V> removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 当数组不为空, 且根据hash计算出的index处有值时, 才进行删除
    if ((tab = table) != null && (n = tab.length) > 0 && (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // hash相等, 且key也相等, 则记录该节点, 等下再进行删除
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                // 红黑树...
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                // 从链头开始遍历链表, 找出要删除的节点
                do {
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        // node不为空, 表示存在要删除的节点. 当 matchValue 为true, 还需要判断value也相等才进行删除
        if (node != null && (!matchValue || (v = node.value) == value || (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                // 红黑树...
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                // 链表的头就是要删除的节点. 直接将其下一个值上移
                tab[index] = node.next;
            else
                // 从链表中移除 node
                p.next = node.next;
            ++modCount;
            --size;
            // 空方法, 由其子类 LinkedHashMap
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

#### clear
```java
public void clear() {
    Node<K,V>[] tab;
    modCount++;
    if ((tab = table) != null && size > 0) {
        size = 0;
        for (int i = 0; i < tab.length; ++i)
            tab[i] = null;
    }
}
```

### 查询

#### get
```java
public V get(Object key) {
    Node<K,V> e;
    // 找到Node节点就返回其value, 否则返回 null
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
...
// 找不到就返回一个默认值
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
}
...
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 当数组不为空, 且根据hash计算出的index处有值
    if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
        // hash相等, 且key也相等, 该节点就是要查找的目标
        if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // next不为空
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                // 红黑树...
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 从链表的头开始遍历查找
            do {
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

#### containsKey
```java
public boolean containsKey(Object key) {
    // getNode
    return getNode(hash(key), key) != null;
}
```

#### containsValue
```java
public boolean containsValue(Object value) {
    Node<K,V>[] tab; V v;
    if ((tab = table) != null && size > 0) {
        // 遍历数组
        for (int i = 0; i < tab.length; ++i) {
            // 遍历(链表|树)
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                // 判断 value相等
                if ((v = e.value) == value || (value != null && value.equals(v)))
                    return true;
            }
        }
    }
    return false;
}
```