---
layout: post
category: java
title:  "Source code of FilterOutputStream in java."
description: "Analyze source code of FilterOutputStream ."
tags: [java,source]
---

## 说明

这是所有过滤输出流的类的父类。这些流在一个已存在的输出流的最外层，转换数据或添加附加功能。本类只是简单的重写了`OutputStream`的方法。

## implements

```
OutputStream
```

## fields

### 底层流，用来过滤

```
protected OutputStream out;
```

## methods

### 构造函数，底层数据。

```
public FilterOutputStream(OutputStream out) {
    this.out = out;
}
```

### 写入一个字节，父类是一个抽象方法

```
public void write(int b) throws IOException {
    out.write(b);
}
```
### 写入一个数组， 调用自己的方法，不会调用底层的`write`方法

```
public void write(byte b[]) throws IOException {
    write(b, 0, b.length);
}
```

### 写入方法，每个字节调用一次`write`方法，不会调用底层的`write`方法

```
public void write(byte b[], int off, int len) throws IOException {
    if ((off | len | (b.length - (len + off)) | (off + len)) < 0)
        throw new IndexOutOfBoundsException();

    for (int i = 0 ; i < len ; i++) {
        write(b[off + i]);
    }
}
```

### 写入部分数组，不会调用底层的`write`方法

```
public void write(byte b[], int off, int len) throws IOException {
    if ((off | len | (b.length - (len + off)) | (off + len)) < 0)
        throw new IndexOutOfBoundsException();

    for (int i = 0 ; i < len ; i++) {
        write(b[off + i]);
    }
}
```

### 刷新流，会调用底层的`flush`方法

```
public void flush() throws IOException {
    out.flush();
}
```

### 关闭流，先刷新

```
public void close() throws IOException {
    try (OutputStream ostream = out) {
        flush();
    }
}
```