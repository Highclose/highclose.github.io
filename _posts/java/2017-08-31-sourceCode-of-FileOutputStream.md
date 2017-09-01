---
layout: post
category: java
title:  "Source code of FileOutputStream in java."
comments: true
description: "Analyze source code of FileOutputStream."
tags: [java,source]
---



`FileOutputStream`用作将数据写入到`File`或者`FileDescriptor`中。`FileOutputStream`用来写入字节流（比如图片），如果要写入字符，考虑使用`FileWriter`.

## Fields

### 文件描述符

     ```
     private final FileDescriptor fd;
     ```

### 添加操作时为`true`

     ```
     private final boolean append;
     ```

### 懒加载的通道。

     ```
     private FileChannel channel;
     ```

### 文件路径，用`FileDescriptor`构造时为空。

     ```
     private final String path;
     ```

### //

     ```
     private final Object closeLock = new Object();
     private volatile boolean closed = false;
     ```

### 静态代码块

     ```
     private static native void initIDs();

     static {
         initIDs();
     }
     ```

## methods

### 构造函数，首先如果有个安全管理器，会先用路径调用`checkWrite`方法。

     ```
     public FileOutputStream(String name) throws FileNotFoundException {
         this(name != null ? new File(name) : null, false);
     }
     ```

### 构造函数。写入时为追加的方式。

     ```
     public FileOutputStream(String name, boolean append)
         throws FileNotFoundException
     {
         this(name != null ? new File(name) : null, append);
     }
     ```

### 用`File`实例来构造实例。

     ```
     public FileOutputStream(File file) throws FileNotFoundException {
         this(file, false);
     }
     ```

### 前面三个构造函数实际都调用了这个构造函数。

     ```
     public FileOutputStream(File file, boolean append)
         throws FileNotFoundException
     {
         String name = (file != null ? file.getPath() : null);
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
         	//检查是否能写入
             security.checkWrite(name);
         }
         if (name == null) {
             throw new NullPointerException();
         }
         if (file.isInvalid()) {
             throw new FileNotFoundException("Invalid file path");
         }
         创建FileDescriptor实例
         this.fd = new FileDescriptor();
         fd.attach(this);
         this.append = append;
         this.path = name;

         open(name, append);
     }
     ```

### 用`FileDescriptor`来构造

     ```
     public FileOutputStream(FileDescriptor fdObj) {
         SecurityManager security = System.getSecurityManager();
         if (fdObj == null) {
             throw new NullPointerException();
         }
         if (security != null) {
             security.checkWrite(fdObj);
         }
         this.fd = fdObj;
         this.append = false;
         this.path = null;

         fd.attach(this);
     }
     ```

### 本地方法

     ```
     private native void open0(String name, boolean append)
         throws FileNotFoundException;
     ```

### 装饰本地方法


     ```
     private void open(String name, boolean append)
         throws FileNotFoundException {
         open0(name, append);
     }
     ```

### 本地方法写入一个字节

     ```
     private native void write(int b, boolean append) throws IOException;
     ```

### 调用了本地方法

     ```
     public void write(int b) throws IOException {
         write(b, append);
     }
     ```

### 写入一个数组

     ```
     private native void writeBytes(byte b[], int off, int len, boolean append)
         throws IOException;
     ```

### 调用了本地方法

     ```
     public void write(byte b[]) throws IOException {
         writeBytes(b, 0, b.length, append);
     }
     ```

### 调用了本地方法，将从`off`位置开始的长度为`len`的数组写入文件

     ```
     public void write(byte b[], int off, int len) throws IOException {
         writeBytes(b, off, len, append);
     }
     ```

### 关闭流与相关资源。如果有连接的通道，则通道也关闭。

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

### 本地方法关闭

     ```
     private native void close0() throws IOException;
     ```

### 获取文件描述符

     ```
     public final FileDescriptor getFD()  throws IOException {
        if (fd != null) {
            return fd;
        }
        throw new IOException();
     }
     ```

### 获取唯一的通道。通道的初始位置等于已写入的字节数或者再追加的情况下，等于文件的大小。将字节写入流中也会相应的增加通道的位置。调整通道的位置也会改变流中文件的位置。

     ```
     public FileChannel getChannel() {
         synchronized (this) {
             if (channel == null) {
                 channel = FileChannelImpl.open(fd, path, false, true, append, this);
             }
             return channel;
         }
     }
     ```

### 强制流写入文件。当流不再引用是，调用`close`方法。

     ```
     protected void finalize() throws IOException {
         if (fd != null) {
             if (fd == FileDescriptor.out || fd == FileDescriptor.err) {
                 flush();
             } else {
                 //只有安情况下，才会调用本方法。当对fd的引用都不可达时，则调用close方法
                 close();
             }
         }
     }
     ```