---
layout: post
category: java
title:  "Source code of BufferedOutputStream in java."
description: "Analyze source code of BufferedOutputStream ."
tags: [java,source]
---

## 说明

使用这个类，一个应用将字节写入底层流时不需要在每个字节写入时调用方法

## fields

### 内部缓存的数组

```
protected byte buf[];
```

### 缓存中的有效字节长度

```
protected int count;
```

## methods

### 构造器，默认缓存大小为8192

```
public BufferedOutputStream(OutputStream out) {
    this(out, 8192);
}
```

### 构造器2

```
public BufferedOutputStream(OutputStream out, int size) {
    super(out);
    if (size <= 0) {
        throw new IllegalArgumentException("Buffer size <= 0");
    }
    buf = new byte[size];
}
```

### 刷新，将内部流刷新

```
private void flushBuffer() throws IOException {
    if (count > 0) {
        out.write(buf, 0, count);
        count = 0;
    }
}
```

### 写入指定字节

```
public synchronized void write(int b) throws IOException {
    if (count >= buf.length) {
        flushBuffer();
    }
    //将字节保存在数组中，并未写入底层流
    buf[count++] = (byte)b;
}
```

### 写入指定数组

```
public synchronized void write(byte b[], int off, int len) throws IOException {
    if (len >= buf.length) {
        //如果数组长度大于等于缓存长度，刷新流并直接写入数据，并返回
        flushBuffer();
        out.write(b, off, len);
        return;
    }
    if (len > buf.length - count) {
    	//长度大于剩余有效长度，先刷新流
        flushBuffer();
    }
    //复制数组
    System.arraycopy(b, off, buf, count, len);
    count += len;
}
```

### 级联刷新

```
public synchronized void flush() throws IOException {
    flushBuffer();
    out.flush();
}
```