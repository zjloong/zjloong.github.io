---
title: LinkedHashMap
top: false
cover: false
categories: Android
tags:
  - LinkedHashMap
date: 2019-04-04 15:25:33
img:
coverImg:
summary: LinkedHashMap继承自HashMap, 它的大部分功能都维持了HashMap的原样, 同时在此基础上又维护了一个双向链表结构. 通过对链表的操作, 可以实现LRU、FIFO等功能.
---

LinkedHashMap继承自HashMap, 它的大部分功能都维持了HashMap的原样, 同时在此基础上又维护了一个双向链表结构. 通过对链表的操作, 可以实现LRU、FIFO等功能.

### LinkedHashMap中的节点
```java
// 继承自 HashMap.Node, 并将其拓展为双向链表结构
static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
    // 增加了这两个属性, 实现双向链表结构
    LinkedHashMapEntry<K,V> before, after;
    LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

### LinkedHashMap中新增的属性
```java
// 双向链表的头
transient LinkedHashMapEntry<K,V> head;
// 双向链表的尾
transient LinkedHashMapEntry<K,V> tail;
// 当accessOrder为false时, 双向链表的顺序就是节点插入的顺序;
// 当accessOrder为true时, 每当节点被访问或修改时, 会将该节点移到链表的尾部. 利用这一原理, 可以实现LRU功能
final boolean accessOrder;
```

### LinkedHashMap构造方法
```java
public LinkedHashMap() {
    super();
    accessOrder = false;
}
...
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}
...
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}
...
public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```

可以发现, 构造方法基本和HashMap一致, 唯一多了一个 accessOrder 的设置.

### LinkedHashMap中重写的方法

#### 增加元素
LinkedHashMap没有重写HashMap的put方法, 回顾一下HashMap的putVal.
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        ...
        if (e != null) { 
            ...
            // 空方法, 由其子类LinkedHashMap实现
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    // 空方法, 由其子类LinkedHashMap实现
    afterNodeInsertion(evict);
    return null;
}
```

在添加新元素时, 如果在hash表中没有找到hash值和key都相匹配的节点, 会调用 newNode 方法添加新节点, 并最终调用一个叫 afterNodeInsertion 的空方法; 如果找到了匹配的节点, 则直接替换其value, 并最终调用一个叫 afterNodeAccess 的空方法.

对于需要新建节点的情况, LinkedHashMap重写了 newNode 和 afterNodeInsertion 方法
```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    // 创建的节点是 LinkedHashMapEntry
    LinkedHashMapEntry<K,V> p = new LinkedHashMapEntry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}
...
private void linkNodeLast(LinkedHashMapEntry<K,V> p) {
    // 保存当前链表的尾部
    LinkedHashMapEntry<K,V> last = tail;
    // 将新节点设置为链表的尾部
    tail = p;
    if (last == null)
        // 链尾为空, 表示是第一个元素, 则直接设置为链头
        head = p;
    else {
        // 将新元素放在链尾
        p.before = last;
        last.after = p;
    }
}
...
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMapEntry<K,V> first;
    // 如果 evict为true, 且链头不为空, 且 removeEldestEntry 方法也返回true
    if (evict && (first = head) != null &&  (first)) {
        // 删除链头指向的节点
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
...
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

对于无需新建节点, 而是直接替换其value的情况, LinkedHashMap重写了 afterNodeAccess 方法
```java
// 在改变元素节点的value后, 会调用该方法, 将节点移到链表的尾部
// 在访问元素节点后, 也可能会调用该方法, 将节点移到链表的尾部
void afterNodeAccess(Node<K,V> e) { // move node to last
    // 原来的链尾
    LinkedHashMapEntry<K,V> last;
    // accessOrder为true, 且当前访问的节点不是链尾
    if (accessOrder && (last = tail) != e) {
        // 将该节点强转为LinkedHashMapEntry类型, 同时保存它在链表中的上一个节点和下一个节点
        LinkedHashMapEntry<K,V> p = (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
        // 因为要将该节点移到尾部, 所以将其after置为空
        p.after = null;
        if (b == null)
            // b为null, 表示p就是原来的链头, 所以将直接将它的下一个节点设置为新的链头
            head = a;
        else
            // 将p从原来在链表上的位置移除 (即将p的前置节点的后置节点, 设置为p的后置节点 ...)
            b.after = a;
        if (a != null)
            // 如果p的后置节点不为空, 则将其前置节点设置为p的前置节点
            a.before = b;
        else
            // 如果p的后置节点为null, 表示p就是链尾. 此时设置last为p的前置节点
            last = b;
        if (last == null)
            // 如果last为null, 表示此事链表上没有数据, 直接将p设置为链头
            head = p;
        else {
            // 将p放在链表的尾部
            p.before = last;
            last.after = p;
        }
        // 将链尾设置为p
        tail = p;
        ++modCount;
    }
}
```

#### 删除元素
LinkedHashMap同样也没有重写 remove 方法, 和上面的添加元素相似, HashMap在成功删除元素后, 会调用一个叫 afterNodeRemoval的空方法, LinkedHashMap也重写了该方法
```java
// 在删除了节点e后, 会调用此方法将其从链表上移除
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMapEntry<K,V> p = (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
    // 将其前置节点和后置节点都设置为null, 解除引用关系
    p.before = p.after = null;
    if (b == null)
        // b为null, 表示p就是原来的链头, 因此直接将其后置节点设置为新的链头
        head = a;
    else
        // 将b的后置节点设置为a
        b.after = a;
    if (a == null)
        // a为null, 表示p就是原来的链尾, 因此将其前置节点设置为新的链尾
        tail = b;
    else
        // 将a的前置节点设置为b
        a.before = b;
}
```

#### 查找元素
LinkedHashMap重写了 get 和 getOrDefault 方法
```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    // 如果 accessOrder 为true, 则调用 afterNodeAccess将查到的节点移到链尾
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
...
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return defaultValue;
    // 如果 accessOrder 为true, 则调用 afterNodeAccess将查到的节点移到链尾
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```

#### containsKey & containsValue
LinkedHashMap只重写了containsValue 方法.
```java
public boolean containsValue(Object value) {
    // 直接遍历链表 (HashMap中是同时遍历数组和链表)
    for (LinkedHashMapEntry<K,V> e = head; e != null; e = e.after) {
        V v = e.value;
        if (v == value || (value != null && value.equals(v)))
            return true;
    }
    return false;
}
```