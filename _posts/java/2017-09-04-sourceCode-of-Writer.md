---
layout: post
category: java
title:  "Source code of Writer in java."
description: "Analyze source code of Writer ."
tags: [java,source]
---

## 说明

写入字符的抽象类。

## implements

```
Appendable, Closeable, Flushable
```

## fields

### 临时缓存数组

```
private char[] writeBuffer;
```

### 写入缓存大小

```
private static final int WRITE_BUFFER_SIZE = 1024;
```

### 同步锁

```
protected Object lock;
```

## methods

### 构造器 用本身作为锁

```
protected Writer() {
    this.lock = this;
}
```

### 构造器，用指定的对象作为锁

```
protected Writer(Object lock) {
    if (lock == null) {
        throw new NullPointerException();
    }
    this.lock = lock;
}
```

### 写入一个字符，字符为给定数字的低16位，高16位则被忽略

```
public void write(int c) throws IOException {
    synchronized (lock) {
        if (writeBuffer == null){
            writeBuffer = new char[WRITE_BUFFER_SIZE];
        }
        writeBuffer[0] = (char) c;
        write(writeBuffer, 0, 1);
    }
}
```

### 写入字符数组

```
public void write(char cbuf[]) throws IOException {
    write(cbuf, 0, cbuf.length);
}
```

### 写入字符数组的一部分

```
abstract public void write(char cbuf[], int off, int len) throws IOException;
```

### 写入一个字符串

```
public void write(String str) throws IOException {
    write(str, 0, str.length());
}
```

### 写入字符串的一部分

```
public void write(String str, int off, int len) throws IOException {
    synchronized (lock) {
        char cbuf[];
        if (len <= WRITE_BUFFER_SIZE) {
            if (writeBuffer == null) {
                writeBuffer = new char[WRITE_BUFFER_SIZE];
            }
            cbuf = writeBuffer;
        } else { 
        	// 不要永久的分配太大的缓存区
            cbuf = new char[len];
        }
        // 获取字符串的字符数组
        str.getChars(off, (off + len), cbuf, 0);
        write(cbuf, 0, len);
    }
}
```

### 添加指定的字符序列。`out.append(csq)`与`out.write(csq.toString())`是等价的。调用`toString`方法时，可能没有添加整个序列。比如，在字符缓存序列中使用toString方法，可能会因为缓存的位置和限制返回一个子序列。

```
public Writer append(CharSequence csq) throws IOException {
    if (csq == null)
        write("null");
    else
        write(csq.toString());
    return this;
}
```
### 写入子序列。

```
public Writer append(CharSequence csq, int start, int end) throws IOException {
    CharSequence cs = (csq == null ? "null" : csq);
    write(cs.subSequence(start, end).toString());
    return this;
}
```

### 添加一个字符

```
public Writer append(char c) throws IOException {
    write(c);
    return this;
}
```

### 刷新流，如果流中的缓存保存了一些字符，会马上写入目的地。如果目的地时另一个流，则刷新这个流。因此一个刷新动作或刷新一个链上的所有流。如果目的地时底层操作系统的一个抽象，比如一个文件，刷新动作保证会传递流给操作系统来写入，不保证操作系统一定能写入。

```
abstract public void flush() throws IOException;
```

### 关闭流

```
abstract public void close() throws IOException;
```