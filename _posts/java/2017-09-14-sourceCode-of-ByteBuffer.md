---
layout: post
category: java
title:  "Source code of ByteBuffer in java."
comments: true
description: "Analyze source code of ByteBuffer  ."
tags: [java,source]
---

## 说明

本类定义了6个种类的操作：

* 绝对或相对的`get()` `put()`方法用来读取或写入单个字节
* 相对的`get(byte[])`方法，从缓存中传输一段连续的字节序列到数组中
* 相对的`put(byte[])`方法，从数组或另是缓存中传输一段连续的字节序列到缓存中
* 绝对或相对的`getChar()`  `putChar(char)`方法，读取或写入其他基础类型的数据，转换成/自字节
* 创建缓存视图，可以被视为是包含特定基础数据类型的缓存
* 压缩`compact()` 复制`duplicate()`切片 `slice()`

`ByteBuffer`可以用`allocate`方法（指定缓存空间）创建，或用`wrap(byte[]`方法（包裹一个已存在的字节数组）创建

一个字节缓存或直接的或是非直接的。直接字节缓存，JVM会努力用本地IO方法来直接运行。也就是说，它会试着避免在操作系统本地IO操作之前拷贝缓存数据到中间缓存区，或之后从中间缓存区拷贝到缓存中。

直接缓存可以用工厂方法`allocateDirect(int)`来创建，直接缓存会比非直接缓存有更高的存储分配空间和存储单元。直接缓存的内容不在垃圾回收堆内，所以它对程序的内存占用没有明显的影响。所以推荐直接内存分配给大型，持久的缓存。通常来说，只有在程序性能有效提高的前提下分配直接缓存。

直接缓存也可以由`FileChannel`映射文件生成，如果这类缓存引用了一个不可以用的内存区域，则获取该区域时，不会改动缓存内容并抛出异常。

本类提供了读取写入除了`boolean`以外的基础类型。基础类型与字节序列根据缓存的当期字节顺序来转换，这个顺序可以用`order`方法来获取或修改。初始值为`ByteOrder.BIG_ENDIAN`

## implements

```
extends Buffer
implements Comparable<ByteBuffer>
```

## fields

### 在堆缓存时非空

```
final byte[] hb;                  
```

### 位移

```
final int offset;
```

### 在堆缓存时有效

```
boolean isReadOnly;               
```

### 字节顺序

```
boolean bigEndian                                    
    = true;
boolean nativeByteOrder                            
    = (Bits.byteOrder() == ByteOrder.BIG_ENDIAN);
```

## methods

### 构造器

```
ByteBuffer(int mark, int pos, int lim, int cap,   // package-private
             byte[] hb, int offset)
{
    super(mark, pos, lim, cap);
    this.hb = hb;
    this.offset = offset;
}


ByteBuffer(int mark, int pos, int lim, int cap) { // package-private
    this(mark, pos, lim, cap, null, 0);
}
```

### 分配直接内存

```
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
```

### 分配一个字节缓存，有一个底层数组

```
public static ByteBuffer allocate(int capacity) {
    if (capacity < 0)
        throw new IllegalArgumentException();
    return new HeapByteBuffer(capacity, capacity);
}
```

### 包裹指定字节数组，该数组成为底层数组，修改缓存会使数组改动，反之亦然，缓存容量为数组长度，位置为`offset`，显示为`offset+length`

```
public static ByteBuffer wrap(byte[] array,
                                int offset, int length)
{
    try {
        return new HeapByteBuffer(array, offset, length);
    } catch (IllegalArgumentException x) {
        throw new IndexOutOfBoundsException();
    }
}

public static ByteBuffer wrap(byte[] array) {
  return wrap(array, 0, array.length);
}

```

###  创建一个新的字节缓存，内容是本缓存共享的一部分。新缓存的内容从本缓存的当前位置开始，新缓存与本缓存的修改会互相影响，位置，记录值，限制相互独立。

```
public abstract ByteBuffer slice();
```

### 拷贝

```
public abstract ByteBuffer duplicate();//修改相互影响
public abstract ByteBuffer asReadOnlyBuffer();//创造一个制度的拷贝

```

### 读写一个字节

```
public abstract byte get();

public abstract ByteBuffer put(byte b);

public abstract byte get(int index);

public abstract ByteBuffer put(int index, byte b);

```

### 读写多个字节

```

public ByteBuffer get(byte[] dst, int offset, int length) {
    checkBounds(offset, length, dst.length);
    if (length > remaining())
        throw new BufferUnderflowException();
    int end = offset + length;
    for (int i = offset; i < end; i++)
        dst[i] = get();
    return this;
}

public ByteBuffer get(byte[] dst) {
    return get(dst, 0, dst.length);
}


public ByteBuffer put(ByteBuffer src) {
    if (src == this)
        throw new IllegalArgumentException();
    if (isReadOnly())
        throw new ReadOnlyBufferException();
    int n = src.remaining();
    if (n > remaining())
        throw new BufferOverflowException();
    for (int i = 0; i < n; i++)
        put(src.get());
    return this;
}

public ByteBuffer put(byte[] src, int offset, int length) {
    checkBounds(offset, length, src.length);
    if (length > remaining())
        throw new BufferOverflowException();
    int end = offset + length;
    for (int i = offset; i < end; i++)
        this.put(src[i]);
    return this;
}

public final ByteBuffer put(byte[] src) {
    return put(src, 0, src.length);
}
```

### 其他方法

```
//是否有底层数组 
public final boolean hasArray() {
    return (hb != null) && !isReadOnly;
}

//返回数组 
public final byte[] array() {
    if (hb == null)
        throw new UnsupportedOperationException();
    if (isReadOnly)
        throw new ReadOnlyBufferException();
    return hb;
}

//返回位移 
public final int arrayOffset() {
    if (hb == null)
        throw new UnsupportedOperationException();
    if (isReadOnly)
        throw new ReadOnlyBufferException();
    return offset;
}

//压缩缓存
//将缓存当前位置到限制位置的字节复制到通道开头，然后把容量移动到复制字节总数量的长度处
//再把限制位移动到容量处。
 /**
 *   buf.clear();         
 *   while (in.read(buf) >= 0 || buf.position != 0) {
 *       buf.flip();
 *       out.write(buf);
 *       buf.compact();    // 只剩一部分需要写入
 *   }
 */
public abstract ByteBuffer compact();

//是否是直接缓存
public abstract boolean isDirect();

//缓存的描述 
public String toString() {
    StringBuffer sb = new StringBuffer();
    sb.append(getClass().getName());
    sb.append("[pos=");
    sb.append(position());
    sb.append(" lim=");
    sb.append(limit());
    sb.append(" cap=");
    sb.append(capacity());
    sb.append("]");
    return sb.toString();
}

//hashcode
public int hashCode() {
    int h = 1;
    int p = position();
    for (int i = limit() - 1; i >= p; i--)



        h = 31 * h + (int)get(i);

    return h;
}

//判断相等
public boolean equals(Object ob) {
    if (this == ob)
        return true;
    if (!(ob instanceof ByteBuffer))
        return false;
    ByteBuffer that = (ByteBuffer)ob;
    if (this.remaining() != that.remaining())
        return false;
    int p = this.position();
    for (int i = this.limit() - 1, j = that.limit() - 1; i >= p; i--, j--)
        if (!equals(this.get(i), that.get(j)))
            return false;
    return true;
}

private static boolean equals(byte x, byte y) {



    return x == y;

}

//比较
public int compareTo(ByteBuffer that) {
    int n = this.position() + Math.min(this.remaining(), that.remaining());
    for (int i = this.position(), j = that.position(); i < n; i++, j++) {
        int cmp = compare(this.get(i), that.get(j));
        if (cmp != 0)
            return cmp;
    }
    return this.remaining() - that.remaining();
}

private static int compare(byte x, byte y) {
    return Byte.compare(x, y);
}
```

### 字节顺序方法

```
//获取
public final ByteOrder order() {
    return bigEndian ? ByteOrder.BIG_ENDIAN : ByteOrder.LITTLE_ENDIAN;
}

//修改
public final ByteBuffer order(ByteOrder bo) {
    bigEndian = (bo == ByteOrder.BIG_ENDIAN);
    nativeByteOrder =
        (bigEndian == (Bits.byteOrder() == ByteOrder.BIG_ENDIAN));
    return this;
}
```

### `ByteBufferAs-X-Buffer`使用

```
abstract byte _get(int i);                       
abstract void _put(int i, byte b);   
```

### 获取二进制数据(其他基础类型方法都一样)

```
//读取接下来的两个字节，组成一个字符 
public abstract char getChar();

//写入字符包含的两个字节 
public abstract ByteBuffer putChar(char value);

public abstract char getChar(int index);

public abstract ByteBuffer putChar(int index, char value);

//字符缓存试图 
public abstract CharBuffer asCharBuffer();
```