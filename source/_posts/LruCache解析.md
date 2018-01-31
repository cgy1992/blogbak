title: LruCache解析
categories:
- 源码解析
tags:
- android
- 源码解析
---
## 前言
在学习`Glide`的时候, 我们会看到Glide的`二级缓存`, 分别分为`内存缓存`和`磁盘缓存`, 而不论哪种缓存都使用到了`Lru`算法, 本篇主要看一下Android里的`LruCache`的实现
<!-- more -->
## Lrucache实现原理
以v4包的LruCahce类源码为准, 我们先看下他的构造函数
``` java
public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
```
主要关注的是,LruCache内部通过`LinkedHashMap`用来管理缓存列表, `LinkedHashMap`是一个由数组+双向链表的数据结构实现的(我们以api26代码为准)
``` java
/**
     * Constructs a new {@code LinkedHashMap} instance with the specified
     * capacity, load factor and a flag specifying the ordering behavior.
     *
     * @param initialCapacity
     *            the initial capacity of this hash map.
     * @param loadFactor
     *            the initial load factor.
     * @param accessOrder
     *            {@code true} if the ordering should be done based on the last
     *            access (from least-recently accessed to most-recently
     *            accessed), and {@code false} if the ordering should be the
     *            order in which the entries were inserted.
     * @throws IllegalArgumentException
     *             when the capacity is less than zero or the load factor is
     *             less or equal to zero.
     */
    public LinkedHashMap(
            int initialCapacity, float loadFactor, boolean accessOrder) {
        super(initialCapacity, loadFactor);
        init();
        this.accessOrder = accessOrder;
    }
```
它的构造函数中的accessOrder表示的是如果为`true`,则为访问顺序; 否则, 为插入顺序排序, 我们可以看下`accessOrder`的相关的处理逻辑, 当我们调用`map.get(key)`和`map.put()`的使用, 都会调用到`afterNodeAccess()`方法, 该方法的作用就是将命中获取的引用对象, 放到链表的尾部, 就是说明, `LinkedHashMap`本身每次访问读取的时候, 都会把读取到的值放在尾部, 那么越不常用的对象越会在链表的头部
``` java
public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
```
这一段代码的处理就是判断目标节点的前后是否有对象, 摘除出目标节点, 将其放在last
``` java
void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMapEntry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMapEntry<K,V> p =
                (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```
我们可以由此了解到`LruCache`类是通过`LinkedHashMap`来做缓存的`Lru`(Least Recently Used)管理, 我们在来看下`LruCache`的几个主要的方法
### get()
``` java
public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            // LinkedHashMap 的get(key)方法会重新链接排序
            mapValue = map.get(key);
            if (mapValue != null) {
                // 命中次数
                hitCount++;
                return mapValue;
            }
            // 非命中次数
            missCount++;
        }

        // create是个空方法, 可以自己实现
        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

        synchronized (this) {
            createCount++;
            mapValue = map.put(key, createdValue);

            // 如果对应的key之前是有值, 说明是有冲突的
            if (mapValue != null) {
                // 有冲突的情况, 则替换为旧值
                // There was a conflict so undo that last put
                map.put(key, mapValue);
            } else {
                size += safeSizeOf(key, createdValue);
            }
        }

        // 冲突的情况下
        if (mapValue != null) {
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
            trimToSize(maxSize);
            return createdValue;
        }
    }
```
### put()
``` java
public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            // 缓存次数添加
            putCount++;
            size += safeSizeOf(key, value);
            previous = map.put(key, value);
            // size是缓存大小数, 如果之前对应key有缓存的情况下, 缓存大小其实是不变的, 所以要减去原来的数
            if (previous != null) {
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            // 缓存被替换, 调用到的方法
            entryRemoved(false, key, previous, value);
        }

        trimToSize(maxSize);
        return previous;
    }
```
### trimToSize()
不论是`get`还是`put`还是设置最大缓存大小`resize`,我们都会调用到`trimToSize`方法, 这个方法就是用来处理当超出缓存大小要求的时候, 删除最老的缓存, 直到缓存大小低于要求
``` java
public void trimToSize(int maxSize) {
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize || map.isEmpty()) {
                    break;
                }

                Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
                key = toEvict.getKey();
                value = toEvict.getValue();
                // LinkedHashMap构造函数中的accessOrder字段为true, 表示有读取排序
                // 从最少使用顺序排序到最多排序
                // 所以移除第一个value, 等于是移除最少使用的缓存
                // map存储缓存, 直接移除第一个
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }
```
## 总结
现在, 我们可以了解到, 真正辅助`LruCache`实现它的算法的`LinkedHashMap`, 它会以读取的顺序来做顺序排序, 最近读取的在队尾, 当我们调用`LruCache.put`的时候, 将插入元素放在`map`队尾, 然后通过调用`trimToSize`判断是否超出缓存大小, 如果超出, 则移除`map`的队尾对象.当我们调用`LruCache.get`的时候, 直接读取map对应`key`的`value`, 并由于`LinkedHashMap`的内部机制, 对读取顺序重排序, 将对应的元素更新到队尾
