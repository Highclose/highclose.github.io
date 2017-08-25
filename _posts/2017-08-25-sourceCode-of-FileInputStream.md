---
layout: post
category: java
title:  "Source code of FileInputStream in java."
comments: true
description: "Analyze source code of FileInputStream ."
tags: [java,source]
---



FileInputStream 的源码分析。

<!--more-->

`FileInputStream`用来读取字节流（例如图片）。如果要读取字符，考虑使用`FileReader`

1. fields

   * 文件描述符

     ```
     private final FileDescriptor fd;
     ```

   * 文件路径，如果由`FileDescriptor`创建，也值为null

     ```
     private final String path;
     ```

   * 文件通道（NIO）

     ```
     private FileChannel channel = null;
     ```

   * //

     ```
     private final Object closeLock = new Object();
     ```

   * //

     ```
     private volatile boolean closed = false;
     ```

   * 静态代码块

     ```
     static {
         initIDs();
     }
     private static native void initIDs();
     ```

2. methods

   * `name`为文件在系统中的路径，新建一个`FileDescriptor`来表示这个文件，通过打开这个文件来创建`FileInputStream`。首先，

     ```
     public FileInputStream(String name) throws FileNotFoundException {
         this(name != null ? new File(name) : null);
     }
     ```

   * 首先，如果有安全管理器，调用`checkRead`方法来检测这个文件。

     ```
     public FileInputStream(File file) throws FileNotFoundException {
         String name = (file != null ? file.getPath() : null);
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkRead(name);
         }
         if (name == null) {
             throw new NullPointerException();
         }
         if (file.isInvalid()) {
             throw new FileNotFoundException("Invalid file path");
         }
         fd = new FileDescriptor();
         fd.attach(this);
         path = name;
         open(name);
     }
     ```

   * 如果在产生的流上调用这个方法来尝试获取流的I/O,会抛出`IOException`异常

     ```
     public FileInputStream(FileDescriptor fdObj) {
         SecurityManager security = System.getSecurityManager();
         if (fdObj == null) {
             throw new NullPointerException();
         }
         if (security != null) {
             security.checkRead(fdObj);
         }
         fd = fdObj;
         path = null;

         /*
          * FileDescriptor is being shared by streams.
          * Register this stream with FileDescriptor tracker.
          */
         fd.attach(this);
     }
     ```

   * 打开文件，本地方法

     ```
     private native void open0(String name) throws FileNotFoundException;
     ```

   * 装饰本地方法

     ```
     private void open(String name) throws FileNotFoundException {
         open0(name);
     }
     ```

   * 从流中读取一个字节。

     ```
     public int read() throws IOException {
         return read0();
     }

     private native int read0() throws IOException;
     ```

   * 读取指定长度的字节内容至数组中。

     ```
     private native int readBytes(byte b[], int off, int len) throws IOException;
     public int read(byte b[]) throws IOException {
             return readBytes(b, 0, b.length);
         }
     public int read(byte b[], int off, int len) throws IOException {
             return readBytes(b, off, len);
         }
     ```

   * 跳过指定长度

     ```
     public native long skip(long n) throws IOException;
     ```

   * 返回可读取字节的预估值

     ```
     public native int available() throws IOException;
     ```

   * 关闭流

     ```
     public void close() throws IOException {
         synchronized (closeLock) {
             if (closed) {
                 return;
             }
             closed = true;
         }
         if (channel != null) {
            channel.close();
         }

         fd.closeAll(new Closeable() {
             public void close() throws IOException {
                close0();
            }
         });
     }
     ```