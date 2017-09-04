---
layout: post
category: java
title:  "Source code of Reader in java."
description: "Analyze source code of Reader ."
tags: [java,source]
---



##  说明

读取字符流的抽象类。继承必须实现的方法为`read(char[], int, int)`和`close()`。但是基于更好的性能和附加的功能，大多数的子类其他的方法。

## implements

```
Readable, Closeable
```

## fields

### 用来给流上同步锁。为了效率，一个字符流会使用一个对象而不是`this`来保护关键段。

```
protected Object lock;
```

### 最大跳过长度

```
private static final int maxSkipBufferSize = 8192;
```

### 跳过缓存，赋值前为空

```
private char skipBuffer[] = null;
```

## methods

### 构造器，用本身来同步关键段。

```
protected Reader() {
    this.lock = this;
}
```

### 构造器，用给定的对象来同步关键段

```
protected Reader(Object lock) {
    if (lock == null) {
        throw new NullPointerException();
    }
    this.lock = lock;
}
```

### 将字符读取至指定的字符缓存。缓存也是用来存储字符，唯一的不同是`put`操作的结果中，缓存的`flip`和`rewind`没有执行。

```
public int read(java.nio.CharBuffer target) throws IOException {
    int len = target.remaining();
    char[] cbuf = new char[len];
    int n = read(cbuf, 0, len);
    if (n > 0)
        target.put(cbuf, 0, n);
    return n;
}
```

### 读取一个字符。该方法直到字符有效时或异常抛出时或到达流末尾时，会一直阻塞

```
public int read() throws IOException {
    char cb[] = new char[1];
    if (read(cb, 0, 1) == -1)
        return -1;
    else
        return cb[0];
}
```

### 将字符读取至数组中。该方法直到输入有效或异常或到末尾会一直阻塞。

```
public int read(char cbuf[]) throws IOException {
    return read(cbuf, 0, cbuf.length);
}
```

### 抽象方法。读取字符至一个指定的数组。返回值为读取的字符数。

```
abstract public int read(char cbuf[], int off, int len) throws IOException;
```

### 跳过字符。该方法直到输入有效或异常或到末尾会一直阻塞。返回值为实际读取字符数。

```
public long skip(long n) throws IOException {
    if (n < 0L)
        throw new IllegalArgumentException("skip value is negative");
    int nn = (int) Math.min(n, maxSkipBufferSize);
    synchronized (lock) {
        if ((skipBuffer == null) || (skipBuffer.length < nn))
        	//创建缓存数组
            skipBuffer = new char[nn];
        long r = n;
        while (r > 0) {
            int nc = read(skipBuffer, 0, (int)Math.min(r, nn));
            if (nc == -1)
                break;
            r -= nc;
        }
        return n - r;
    }
}
```

### 流是否可以读取。`true`则下个`read`不会堵塞，反之为`false`，但是返回`false`不保证下个`read`一定会堵塞。

```
public boolean ready() throws IOException {
    return false;
}
```

### 流是否支持`mark`

```
public boolean markSupported() {
    return false;
}
```

### 记录流的当前位置。后续的`reset`动作会试着将位置移动到记录的位置。并不是所有的字符输出流都支持。`readAHeadLimit`标志表示在保存标记位置前能读取的字符数量。

```
public void mark(int readAheadLimit) throws IOException {
    throw new IOException("mark() not supported");
}
```

### 重置流。如果流被标记，则试着恢复到标记位。如果没有标记，根据特定的流来进行重置，比如重置到起始位。

```
public void reset() throws IOException {
    throw new IOException("reset() not supported");
}
```

### 关闭流

```
abstract public void close() throws IOException;
```