---
layout: post
category: java
title:  "Source code of OutputStream in java."
comments: true
description: "Analyze source code of OutputStream ."
tags: [java,source]
---



OutputStream 的源码分析。

<!--more-->

OutputStream 为一个抽象类。

1. implements

   ```
   Closeable, Flushable
   ```

2. methods

   * 写入一个字节到流中，写入`b`的低8位，忽略其高24位。

     ```
     public abstract void write(int b) throws IOException;
     ```

   * 将`byte b[]`写入到流中

     ```
     public void write(byte b[]) throws IOException {
         write(b, 0, b.length);
     }
     ```

   * 将`byte b[]`从`off`开始`len`长度的字节写入到流中。

     ```
     public void write(byte b[], int off, int len) throws IOException {
         if (b == null) {
             throw new NullPointerException();
         } else if ((off < 0) || (off > b.length) || (len < 0) ||
                    ((off + len) > b.length) || ((off + len) < 0)) {
             throw new IndexOutOfBoundsException();
         } else if (len == 0) {
             return;
         }
         for (int i = 0 ; i < len ; i++) {
             write(b[off + i]);
         }
     }
     ```

   * 刷新一个输出流并强制将缓存的字节输出。通常的规范是，调用`flush`代表：被`OutputStream`类的实现所缓存的字节应该直接写入到目的地。如果目的地是操作系统提供的一个抽象（比如文件），本方法确保先前写入流的字节传递给操作系统来写入，但是不保证可以写入物理设备（比如硬盘）。

     ```
     public void flush() throws IOException {
     }
     ```

   * 关闭流与相关系统资源。关闭的流不能再执行输出操作，也不能重新打开。

     ```
     public void close() throws IOException {
     }
     ```