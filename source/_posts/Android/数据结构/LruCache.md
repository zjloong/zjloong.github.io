---
title: LruCache
top: false
cover: false
categories: Android
tags:
  - LruCache
date: 2019-04-11 16:29:33
img:
coverImg:
summary: 
---

使用缓存有助于提高应用的体验, 但是内存的大小是有限的, 所以有必要考虑如何更好更合理的对缓存进行管理.  LruCache是一种常用的缓存策略, 它利用 LinkedHashMap 的特点, 在缓存大小超出阈值时, 对最老的缓存数据进行清理. 也就是最近最少使用算法(Least Recently Used).

### LruCache的用法
```java
// 获取内存大小
int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);
// 一般使用内存的 1/8 作为缓存空间
LruCache<String, Bitmap> cache = new LruCache<String, Bitmap>(maxMemory / 8) {
            // 重写该方法, 计算一对 key-value 占用缓存的大小
            @Override
            protected int sizeOf(@NonNull String key, @NonNull Bitmap value) {
                return bitmap.getRowBytes() * bitmap.getHeight() / 1024;
            }
        };
// 缓存数据
cache.put("key", bitmap);
// 从LruCache中读取缓存数据
Bitmap bitmap = cache.get("key");
```

LruCache的用法比较简单, 下面通过LruCache源码了解一下它的工作原理 (基于 v4 包内的 LruCache)    

### LruCache 的成员变量和构造方法
```java
public class LruCache<K, V> {
    // LruCache中使用 LinkedHashMap 缓存数据, 并利用LinkedHashMap的特点实现 最近最少算法
    private final LinkedHashMap<K, V> map;
    // 当前已缓存数据占用的大小
    private int size;
    // 总的缓存空间大小
    private int maxSize;
    private int putCount;
    private int createCount;
    private int evictionCount;
    private int hitCount;
    private int missCount;
    // 构造方法. 参数指定 缓存空间总的大小
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        // 初始化 LinkedHashMap
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
    ...
    // 更新 maxSize
    public void resize(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        synchronized (this) {
            this.maxSize = maxSize;
        }
        // 判断如果当前缓存的数据超过了 maxSize, 则从最老的缓存数据开始删除
        trimToSize(maxSize);
    }
}
```

### LruCache的put方法
```java
public final V put(K key, V value) {
    // key, value 都不能为null
    if (key == null || value == null) {
        throw new NullPointerException("key == null || value == null");
    }
    // previous用来保存被替换的数据
    V previous;
    synchronized (this) {
        // putCount计数加1
        putCount++;
        // 计算缓存数据的大小
        size += safeSizeOf(key, value);
        // 将数据保存到 LinkedHashMap 中
        previous = map.put(key, value);
        if (previous != null) {
            // 如果是数据覆盖, 则减去对应的缓存大小
            size -= safeSizeOf(key, previous);
        }
    }
    // 如果是数据覆盖, 则回调entryRemoved方法. 该方法是空方法, 用户可以选择自己实现
    if (previous != null) {
        entryRemoved(false, key, previous, value);
    }
    // 判断如果当前缓存的数据超过了 maxSize, 则从最老的缓存数据开始删除
    trimToSize(maxSize);
    return previous;
}
...
private int safeSizeOf(K key, V value) {
    // sizeOf默认返回1, 实际中用户应该自己重写该方法计算被缓存数据的大小
    int result = sizeOf(key, value);
    if (result < 0) {
        throw new IllegalStateException("Negative size: " + key + "=" + value);
    }
    return result;
}
...
public void trimToSize(int maxSize) {
    while (true) {
        K key;
        V value;
        synchronized (this) {
            // size异常, 推断是用户没有正确重写 sizeOf 方法 
            if (size < 0 || (map.isEmpty() && size != 0)) {
                throw new IllegalStateException(getClass().getName() + ".sizeOf() is reporting inconsistent results!");
            }
            // 如果还没有达到maxSize, 则跳出循环
            if (size <= maxSize || map.isEmpty()) {
                break;
            }
            // 取出 LinkedHashMap 中最老的数据
            Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
            key = toEvict.getKey();
            value = toEvict.getValue();
            // 移除缓存
            map.remove(key);
            // 减去对应的大小
            size -= safeSizeOf(key, value);
            // evictionCount计数加1
            evictionCount++;
        }
        // 如果有删除缓存, 则回调entryRemoved方法. 该方法是空方法, 用户可以选择自己实现
        entryRemoved(true, key, value, null);
    }
}
```

### LruCache的get方法
```java
public final V get(K key) {
    // key不能为null
    if (key == null) {
        throw new NullPointerException("key == null");
    }
    V mapValue;
    synchronized (this) {
        // 根据key读取数据
        mapValue = map.get(key);
        if (mapValue != null) {
            // 如果成功取到数据. hitCount计数加1, 并直接返回该数据
            hitCount++;
            return mapValue;
        }
        // 没有取到数据. missCount计数加1
        missCount++;
    }
    // 当没有取到数据时, 调用 create 尝试创建一条数据
    // create默认返回null, 用户可以选择自己重写该方法
    V createdValue = create(key);
    if (createdValue == null) {
        return null;
    }
    // 走到这里表示用户重写了create()方法, 并返回了不为null的数据
    synchronized (this) {
        // createCount计数加1
        createCount++;
        // 将数据添加到集合中
        mapValue = map.put(key, createdValue);
        if (mapValue != null) {
            // 如果对应的key已经有了数据, 则将其重新加回去
            map.put(key, mapValue);
        } else {
            // 计算缓存大小
            size += safeSizeOf(key, createdValue);
        }
    }
    if (mapValue != null) {
        entryRemoved(false, key, createdValue, mapValue);
        return mapValue;
    } else {
        // 因为添加了一条 create 返回的数据.  判断如果当前缓存的数据超过了 maxSize, 则从最老的缓存数据开始删除
        trimToSize(maxSize);
        return createdValue;
    }
}
```

### LruCache的remove方法
```java
public final V remove(K key) {
    // key不能为null
    if (key == null) {
        throw new NullPointerException("key == null");
    }

    V previous;
    synchronized (this) {
        // 从集合中删除数据
        previous = map.remove(key);
        if (previous != null) {
            // 减去对应的缓存大小
            size -= safeSizeOf(key, previous);
        }
    }

    if (previous != null) {
        entryRemoved(false, key, previous, null);
    }

    return previous;
}
```

### 其它方法
```java
// 清除所有缓存
public final void evictAll() {
    trimToSize(-1); 
}
// 获取当前使用缓存的大小
public synchronized final int size() {
    return size;
}
// 获取最大缓存大小
public synchronized final int maxSize() {
    return maxSize;
}
// 调用get方法成功取出缓存的次数
public synchronized final int hitCount() {
    return hitCount;
}
// 调用get方法没有取到缓存的次数
public synchronized final int missCount() {
    return missCount;
}
// 调用get方法没有取到缓存后, 通过 create创建数据的次数
public synchronized final int createCount() {
    return createCount;
}
// 调用put方法保存数据的次数
public synchronized final int putCount() {
    return putCount;
}
// 因为缓存空间不够, 导致清除缓存的个数
public synchronized final int evictionCount() {
    return evictionCount;
}
// ...
public synchronized final Map<K, V> snapshot() {
    return new LinkedHashMap<K, V>(map);
}
```