---
layout: post
category: java
title:  "Source code of Buffer in java."
comments: true
description: "Analyze source code of Buffer ."
tags: [java,source]
---

## 说明

缓存是一个JAVA基础数据类型的数据容器，是一个线性的，有限的序列。除了它的内容，一个缓存的必要属性有`capacity` `limit` `position`。

* `capacity`：是容量中的数据数量。缓存的这个属性不会是负值也不会改变
* `limit`： 是缓存中可以写入或读取的第一个元素索引。这个属性不会是负值也不会大于容量
* `position`：下一个将要读取或写入的元素的索引。不会是负值也不会大于`limit`

本类是一个抽象类，所有子类都定义了两类方法`put` `get`:

相应的方法在当前益读或写一个或多个元素并且根据元素数量增加位置。当要求传输的数量超过限制，则相对应 的`get`方法抛出`BufferUnderflowException`，`put`抛出`BufferOverflowException`，两种情况，都不会有数据传输。绝对的操作会直接使用索引而不会修改位置。绝对的`put` `get`方法使用的索引超过限制时会抛出`IndexOutOfBoundsException`

数据也会因为通道的IO操作传入传出缓存，该动作跟当前位置有关。

一个新创建的缓存，`position=0` `mark`未定义。初始的限制可能也是0，也会根据如果构造或缓存的类型而是别的值。新分配的缓存的元素值都被初始化为0。

除了以上这些方法，还定义了其他的方法：

* `clear`：使缓存可以接受通道读取序列方法或相对应的`put`操作：设置`limit`到`capacity`，`position=0`。 

* `flip`：使缓存可以接受通道写入功能序列方法或相对应的`get`操作：设置`limit`到当前位置并且设置`position=0`

* `rewind`：使缓存可以重新读取包含的数据：不修改`limit`并设置`position=0` 

  每个缓存都是可读的，但不是每个都是可写的。缓存的使用不是线程安全的。可以链式调用，比如：

  ```
   b.flip();
   b.position(23);
   b.limit(42);</pre></blockquote>
  ```

可以写成`b.flip().position(23).limit(42);`

## fields

### 缓存中保存的元素分隔符

```
static final int SPLITERATOR_CHARACTERISTICS =
    Spliterator.SIZED | Spliterator.SUBSIZED | Spliterator.ORDERED;
```

### 初始变量

```
private int mark = -1;
private int position = 0;
private int limit;
private int capacity;
```

### 为了给`JNI GetDirectBufferAddress`提速，只给直接缓存使用。

```
long address;
```

## methods

### 构造器，用指定的参数构造

```
Buffer(int mark, int pos, int lim, int cap) {       // package-private
    if (cap < 0)
        throw new IllegalArgumentException("Negative capacity: " + cap);
    this.capacity = cap;
    limit(lim);
    position(pos);
    if (mark >= 0) {
        if (mark > pos)
            throw new IllegalArgumentException("mark > position: ("
                                               + mark + " > " + pos + ")");
        this.mark = mark;
    }
}
```

### 参数操作

```
//获取容量
public final int capacity() {
    return capacity;
}

//获取位置
public final int position() {
    return position;
}

//设置位置
public final Buffer position(int newPosition) {
    if ((newPosition > limit) || (newPosition < 0))
        throw new IllegalArgumentException();
    position = newPosition;
    if (mark > position) mark = -1;
    return this;
}

//获取限制
public final int limit() {
    return limit;
}

//获取限制
public final Buffer limit(int newLimit) {
    if ((newLimit > capacity) || (newLimit < 0))
        throw new IllegalArgumentException();
    limit = newLimit;
    if (position > limit) position = limit;
    if (mark > limit) mark = -1;
    return this;
}

//设置位置
public final Buffer mark() {
    mark = position;
    return this;
}

//重置位置到记录的位置 
public final Buffer reset() {
    int m = mark;
    if (m < 0)
        throw new InvalidMarkException();
    position = m;
    return this;
}
```

### 清空缓存，在通道读取序列或`put`操作填充缓存之前调用该方法。 

```
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
```

### 翻转缓存，如果有`mark`存在，则被丢弃。在读取或`put`操作之后 ，调用这个方法使缓存可以通道写入或`get`操作。

```
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```

### 绕回缓存 ，在通道写入或`get`方法之前调用该方法，如果`limit`适合需求。

```
public final Buffer rewind() {
    position = 0;
    mark = -1;
    return this;
}
```

### 缓存剩余

```
public final int remaining() {
    return limit - position;
}

public final boolean hasRemaining() {
    return position < limit;
}
```

### 是否可读

```
public abstract boolean isReadOnly();
```

### 是否是由数据构建的，如果返回`true`，则`array()` ` arrayOffset()`可以安全的调用

```
public abstract boolean hasArray();
//获取数组（用来将数组传给本地方法以提高效率），修改数组会使缓存的数据修改，反之亦然
public abstract Object array();
//获取当前缓存的数组在第一个元素的位移
public abstract int arrayOffset();
```

### 是否是直接缓存

```
public abstract boolean isDirect();
```

### 用来确定边界的私有方法

```
//获取GET下一个索引 
final int nextGetIndex() {                          
    if (position >= limit)
        throw new BufferUnderflowException();
    return position++;
}

final int nextGetIndex(int nb) {                   
    if (limit - position < nb)
        throw new BufferUnderflowException();
    int p = position;
    position += nb;
    return p;
}

//获取PUT下一个索引 
final int nextPutIndex() {                         
    if (position >= limit)
        throw new BufferOverflowException();
    return position++;
}

final int nextPutIndex(int nb) {                    
    if (limit - position < nb)
        throw new BufferOverflowException();
    int p = position;
    position += nb;
    return p;
}


//测试指定的索引 
final int checkIndex(int i) {                       
    if ((i < 0) || (i >= limit))
        throw new IndexOutOfBoundsException();
    return i;
}

final int checkIndex(int i, int nb) {              
    if ((i < 0) || (nb > limit - i))
        throw new IndexOutOfBoundsException();
    return i;
}

//mark值
final int markValue() {                           
    return mark;
}

//截断
final void truncate() {                            
    mark = -1;
    position = 0;
    limit = 0;
    capacity = 0;
}

//丢弃mark
final void discardMark() {                          
    mark = -1;
}

static void checkBounds(int off, int len, int size) { 
    if ((off | len | (off + len) | (size - (off + len))) < 0)
        throw new IndexOutOfBoundsException();
}
```

