---
layout: post
category: java
title:  "Source code of HeapByteBuffer in java."
comments: true
description: "Analyze source code of HeapByteBuffer."
tags: [java,source]
---

## 与饥饿

## implements

```
extends ByteBuffer
```

## methods

### 构造器

```
HeapByteBuffer(int cap, int lim) { 
    super(-1, 0, lim, cap, new byte[cap], 0);
}

HeapByteBuffer(byte[] buf, int off, int len) { 
    super(-1, off, off + len, buf.length, buf, 0);
}

protected HeapByteBuffer(byte[] buf, int mark, int pos, int lim, int cap, int off)
{
    super(mark, pos, lim, cap, buf, off);
}
```

### 切片，拷贝，只读，压缩

```
public ByteBuffer slice() {
    return new HeapByteBuffer(hb, -1, 0, this.remaining(), his.remaining(), this.position() + offset);
}

public ByteBuffer duplicate() {
    return new HeapByteBuffer(hb, this.markValue(), this.position(), this.limit(), this.capacity(), offset);
}

public ByteBuffer asReadOnlyBuffer() {
    return new HeapByteBufferR(hb, this.markValue(), this.position(), this.limit(), this.capacity(), offset);
}

public ByteBuffer compact() {
        System.arraycopy(hb, ix(position()), hb, ix(0), remaining());
        position(remaining());
        limit(capacity());
        discardMark();
        return this;
}
```

### 读写单个字节

```
protected int ix(int i) {
    return i + offset;
}

public byte get() {
    return hb[ix(nextGetIndex())];
}

public byte get(int i) {
    return hb[ix(checkIndex(i))];
}

public ByteBuffer put(byte x) {
    hb[ix(nextPutIndex())] = x;
    return this;
}

public ByteBuffer put(int i, byte x) {
    hb[ix(checkIndex(i))] = x;
    return this;
}
```

### 读写多个字节

```
public ByteBuffer get(byte[] dst, int offset, int length) {
    checkBounds(offset, length, dst.length);
    if (length > remaining())
        throw new BufferUnderflowException();
    System.arraycopy(hb, ix(position()), dst, offset, length);
    position(position() + length);
    return this;
}


public ByteBuffer put(byte x) {

    hb[ix(nextPutIndex())] = x;
    return this;



}

public ByteBuffer put(int i, byte x) {

    hb[ix(checkIndex(i))] = x;
    return this;



}

public ByteBuffer put(byte[] src, int offset, int length) {

    checkBounds(offset, length, src.length);
    if (length > remaining())
        throw new BufferOverflowException();
    System.arraycopy(src, offset, hb, ix(position()), length);
    position(position() + length);
    return this;



}

public ByteBuffer put(ByteBuffer src) {

    if (src instanceof HeapByteBuffer) {
        if (src == this)
            throw new IllegalArgumentException();
        HeapByteBuffer sb = (HeapByteBuffer)src;
        int n = sb.remaining();
        if (n > remaining())
            throw new BufferOverflowException();
        System.arraycopy(sb.hb, sb.ix(sb.position()),
                         hb, ix(position()), n);
        sb.position(sb.position() + n);
        position(position() + n);
    } else if (src.isDirect()) {
        int n = src.remaining();
        if (n > remaining())
            throw new BufferOverflowException();
        src.get(hb, ix(position()), n);
        position(position() + n);
    } else {
        super.put(src);
    }
    return this;

}
```

### 通道属性

```
public boolean isDirect() {
    return false;
}

public boolean isReadOnly() {
    return false;
}
```

### 私有办法

```
byte _get(int i) {                           
    return hb[i];
}

void _put(int i, byte b) {                  
    hb[i] = b;
}
```

### 读写字符及字符缓存试图，其他基础类型也一样

```
public char getChar() {
    return Bits.getChar(this, ix(nextGetIndex(2)), bigEndian);
}

public char getChar(int i) {
    return Bits.getChar(this, ix(checkIndex(i, 2)), bigEndian);
}

public ByteBuffer putChar(char x) {
    Bits.putChar(this, ix(nextPutIndex(2)), x, bigEndian);
    return this;
}

public ByteBuffer putChar(int i, char x) {
    Bits.putChar(this, ix(checkIndex(i, 2)), x, bigEndian);
    return this;
}

public CharBuffer asCharBuffer() {
    int size = this.remaining() >> 1;
    int off = offset + position();
    return (bigEndian
            ? (CharBuffer)(new ByteBufferAsCharBufferB(this, -1, 0, size, size, off))
            : (CharBuffer)(new ByteBufferAsCharBufferL(this, -1, 0, size, size, off)));
}
```