title: ArrayList源码解析
date: 2018-04-26 00:00:00
categories:  
- 源码解析
tags:
- Java
- 源码解析
---
## 前言
每个`ArrayList`都有一个容量(capacity)的含义, 他接近于本身队列长度大小, 基本每个元素在新增的时候,都可以做到自动扩容.本篇主要是了解他的扩容机制.本篇源码以openjdk8为准
<!-- more -->
## 构造
`ArrayList`实现了`Serializable`接口, 说明它是支持序列化的, 在它的内部有个`elementData`数组对象元素用来实现内存中的元素缓存, 它的长度相当于就是ArrayList的长度.这里有个关于`transient`关键字的知识点, 它保证了`elementData`不会被序列化, 使得它的生命周期保在调用者的内存中而不会被保存在磁盘中.
``` java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
          transient Object[] elementData;
        }
```
首先我们看下, 日常开发中我们最常用到的无参构造函数, 它主要做的就是将`elementData`引用指向默认静态的一个空数组.
``` java
public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```
还有其他的两个构造函数, 一个是可以初始定义队列的容量, 当传入的`initialCapacity`为负数的时候, 会抛出异常.要注意的是, 当定义的初始容量为0的时候, `elementData`指向的是另外一个空数组`EMPTY_ELEMENTDATA`, 具体为什么要区分两个静态空数组实例, 留在后面的扩容机制上说明.
``` java
public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```
最后一个构造函数式可以直接传集合进去, `elementData`引用指向传入的集合数组, 当集合长度为`0`的时候, 仍然会使它指向 `EMPTY_ELEMENTDATA`空数组.而当传入的集合有元素的情况下, 从注释上看是为了处理6260652的bug, 所以需要判断不是`Object[]`的情况下的时候, 使用`Arrays`内部实现的拷贝的方法`copyOf`进行元素的拷贝.
``` java
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```
具体我们可以稍微看下`Arrays.copyOf`的源码, 后面会发现他是内部核心调用方法, 可以看出每次调用的时候, 实际是实例化了一个新的数组, 将原来的数组元素填充进去实现了copy的目的.
``` java
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
```
## add
我们首先看下几个add的方法, 其实内部实现的原理都不会错过扩容的操作, 所以我们具体看下扩容的原理.
``` java
public boolean add(E e) {
        // size为arrayList的长度大小
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

public void add(int index, E element) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    // 容量确保
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    // 空出index位, 进行拷贝
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    // 根据索引获取数组index位进行赋值
    elementData[index] = element;
    // 长度 + 1
    size++;
}
```
首先, 每次都需要调用到`ensureCapacityInternal`, 进行容量的确定
``` java
/**
     * 确保内部容量大小
     * @param minCapacity
     */
    private void ensureCapacityInternal(int minCapacity) {
        // 当调用ArrayList()构造函数, 内部维护的数组是DEFAULTCAPACITY_EMPTY_ELEMENTDATA
        // 则minCapacity = 10
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        // minCapacity为10 或者为 size + 1
        ensureExplicitCapacity(minCapacity);
    }
```
这里可以看到, 当内部管理数组`elementData`指向内存地址与`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`默认空数组实例相等的时候, 最小的容量会以传入的最小容量和默认容量(10)的最大值为准, 同时, 这里可以了解到, 区分两个空数组的实例, 就是为了扩容的时候确定容量的时候, 可以区分到调用无参构造函数的arrayList, 在第一次添加元素的时候, 可以保证他的容量首先是10(`DEFAULT_CAPACITY`)).然后再是调用到`ensureExplicitCapacity`方法.
``` java
private void ensureExplicitCapacity(int minCapacity) {
        // 操作数记录
        modCount++;

        // overflow-conscious code
        // 如果 当前数组的长度比添加元素后的长度要小则进行扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```
当内部当前管理的数组`elementData`的长度小于添加元素后的长度, 则需要进行真正的扩容方法`grow`

可以看到, 每次容量是根据原来容量的1.5倍来扩充的, 当扩充后的容量仍然没有加入新元素后的长度大的时候, 那么直接扩容到加入后的长度.

而实现扩容的真正机制, 其实还是调用了`Arrays.copyOf`方法, 声明了目标容量的数组, 进行元素拷贝. 这样的话, 其实每次`ArrayList`的内部元素变化的时候, 都会存在相对的内存开销.
``` java
/**
     * 将原来的数组, 拷贝到一个扩容后新长度的数组内
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        // oldCapacity >> 1 相当于 oldCapacity / 2
        // 新容量为老容量的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 如果扩容后容量比添加元素后的长度小
        if (newCapacity - minCapacity < 0)
            // 直接扩容到添加元素后的长度大小
            newCapacity = minCapacity;
        // 新容量大小比 MAX_ARRAY_SIZE 大
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        // 构建newCapacity长度的新数组, elementData指向它
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    // 如果是添加元素后的长度大于 MAX_ARRAY_SIZE, 则容量设为Integer的最大. 否则 -8
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```
## remove
搞懂扩容机制后, 我们可以对应看下其他我们常用的API, 首先看下`remove`相关, 可以看出在移除元素的时候, 其实实际上我们还是做了个拷贝的动作, 将除去移除目标元素的数组其他元素, 拷贝到新的数组中, 同时, 这个时候容量其实是没有变的.
``` java
public E remove(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        modCount++;
        E oldValue = (E) elementData[index];

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }

public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
private void fastRemove(int index) {
    // 操作数的新增
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```
## 其他
我们在看下`get`和`contains`是怎么实现的
``` java
public E get(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        return (E) elementData[index];
    }
```
可以看到`get`的方法, 实际就是对于内部数组的索引查找
``` java
public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```
而`contains(Object o)`方法其实做的就是对内部数组进行遍历查找.
## 总结
考量到使用无参构造函数的时候, 当添加元素的时候, 初始容量为10, 以10为基准进行1.5倍的扩容, 通过源码的解读, 我们可以就可以进行一定的内存优化, 譬如在使用ArrayList的时候, 就应该避免使用无参构造函数, 尽量多的给它定义明确的初始容量, 一个是可以导致不会有过多的内存空间被浪费, 另外一个是可以减少调用到`System.arraycopy`native方法, 保证了一定的内存开销的节省.
