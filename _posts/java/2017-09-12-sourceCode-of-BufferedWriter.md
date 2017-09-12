---
layout: post
category: java
title:  "Source code of BufferedWriter in java."
comments: true
description: "Analyze source code of BufferedWriter ."
tags: [java,source]
---

## 说明

向字符输出流写入文本，缓存字符避免过多的IO。建议在IO操作比较昂贵的流外包裹本类，比如：`PrintWriter out = new PrintWriter(new BufferedWriter(new FileWriter(file)))`。没有缓存的时候，每次调用`print`方法造成一次IO，非常没效率。

## implements

```
 extends Writer 
```

## fields

### 包裹的底层`Writer`

```
private Writer out;
```

### 缓存数组

```
private char cb[];
```

### 前者为实际缓存大小，后者为下个字符的索引

```
private int nChars, nextChar;
```

### 默认缓存大小

```
private static int defaultCharBufferSize = 8192;
```

### 换行符

```
private String lineSeparator;
```

## methods

### 构造器

```
public BufferedWriter(Writer out) {
    this(out, defaultCharBufferSize);
}
```

### 构造器2

```
public BufferedWriter(Writer out, int sz) {
    super(out);
    if (sz <= 0)
        throw new IllegalArgumentException("Buffer size <= 0");
    this.out = out;
    cb = new char[sz];
    nChars = sz;
    nextChar = 0;

    lineSeparator = java.security.AccessController.doPrivileged(
        new sun.security.action.GetPropertyAction("line.separator"));
}
```

### 检查流是否被关闭

```
private void ensureOpen() throws IOException {
    if (out == null)
        throw new IOException("Stream closed");
}
```

### 刷新缓存到底层流

```
void flushBuffer() throws IOException {
    synchronized (lock) {
        ensureOpen();
        if (nextChar == 0)//如果缓存还未读取，直接返回
            return;
        out.write(cb, 0, nextChar);
        nextChar = 0;
    }
}
```

### 写入一个字符

```
public void write(int c) throws IOException {
    synchronized (lock) {
        ensureOpen();
        if (nextChar >= nChars)
            flushBuffer();
        cb[nextChar++] = (char) c;
    }
}
```

### 自己的`min`方法，避免调用`java.lang.Math`

```
private int min(int a, int b) {
    if (a < b) return a;
    return b;
}
```

### 写入一个数组

```
public void write(char cbuf[], int off, int len) throws IOException {
    synchronized (lock) {
        ensureOpen();
        if ((off < 0) || (off > cbuf.length) || (len < 0) ||
            ((off + len) > cbuf.length) || ((off + len) < 0)) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }

        if (len >= nChars) {
            //如果传入的数组长度超过缓存长度，刷新缓存然后透过本类直接写入数据
            flushBuffer();
            out.write(cbuf, off, len);
            return;
        }

        int b = off, t = off + len;
        while (b < t) {
            int d = min(nChars - nextChar, t - b);
            System.arraycopy(cbuf, b, cb, nextChar, d);
            b += d;
            nextChar += d;
            if (nextChar >= nChars)
                flushBuffer();
        }
    }
}
```

### 写入一个字符串

```
public void write(String s, int off, int len) throws IOException {
    synchronized (lock) {
        ensureOpen();

        int b = off, t = off + len;
        while (b < t) {
            int d = min(nChars - nextChar, t - b);
            s.getChars(b, b + d, cb, nextChar);
            b += d;
            nextChar += d;
            if (nextChar >= nChars)
                flushBuffer();
        }
    }
}
```

### 换行

```
public void newLine() throws IOException {
    write(lineSeparator);
}
```

### 刷新流

```
public void flush() throws IOException {
    synchronized (lock) {
        flushBuffer();
        out.flush();
    }
}
```

### 关闭流

```
public void close() throws IOException {
    synchronized (lock) {
        if (out == null) {
            return;
        }
        try (Writer w = out) {//关闭前先刷新
            flushBuffer();
        } finally {
            out = null;
            cb = null;
        }
    }
}
```