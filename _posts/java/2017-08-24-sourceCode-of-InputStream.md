---
layout: post
category: java
title:  "Source code of InputStream in java."
comments: true
description: "Analyze source code of InputStream ."
tags: [java,source]
---



InputStream为一个抽象类。

## implements

   ```
   Closeable //扩展自AutoCloseable，用来释放资源
   ```

## fields

### 可跳过的最大缓冲长度

     ```
     private static final int MAX_SKIP_BUFFER_SIZE = 2048;
     ```

## methods

### 从input stream中读取下一个字节数据。字节的值为从0到255的int类型。如果字节已经读完，到达stream的末尾，返回-1。这个方法会一直阻塞直到stream末尾或者有异常抛出。

     ```
     public abstract int read() throws IOException;
     ```

### 从input stream中读取指定长度的字节保存到缓冲数组中`byte b[]`,返回实际读取的字节数量。方法直到文件末尾或异常会一直阻塞。如果b的长度为0，则不会读取字节并返回0;否则，则至少尝试读取一个字节，如果没有有效字节（到达末尾），则返回-1；否则至少有1个字节会读取并保存到数组中。

     ```
     public int read(byte b[]) throws IOException {
         return read(b, 0, b.length);
     }
     ```

### 读取最大为`len`长度的字节保存在从`off`位置开始的数组`b`中。

     ```
     public int read(byte b[], int off, int len) throws IOException {
         if (b == null) {
         	//b为null
             throw new NullPointerException();
         } else if (off < 0 || len < 0 || len > b.length - off) {
         	//如果索引错误
             throw new IndexOutOfBoundsException();
         } else if (len == 0) {
         	//长度为0，则不会进行操作并返回0
             return 0;
         }
     	//如果这个调用发生错误，会在read()方法里抛出IOException
         int c = read();
         if (c == -1) {
         	//已经到达末尾，则返回-1
             return -1;
         }
         b[off] = (byte)c;

         int i = 1;
         try {
             for (; i < len ; i++) {
                 c = read();
                 if (c == -1) {
                 	//只读取了一个字节
                     break;
                 }
                 b[off + i] = (byte)c;
             }
         } 	//I/O错误。
         	catch (IOException ee) {
         }
         return i;
     }
     ```

### 跳过或丢弃input stream中`n`字节的数据，由于某些原因该方法可能会跳过的数据小于`n`或等于0，在`n`bytes前到达末尾就是其中一种可能。具体跳过的数量将最为返回值返回。如果`n`为负值，则不跳过任何字节并返回0

     这个方法会创建一个字节数组，并不停的把字节写入到这个数组中至少数量到达`n`或者到达末尾。

     ```
     public long skip(long n) throws IOException {

         long remaining = n;
         int nr;

         if (n <= 0) {
         	//小于等于0，不操作，返回0
             return 0;
         }
     	//判断是否超过最大跳过长度
         int size = (int)Math.min(MAX_SKIP_BUFFER_SIZE, remaining);
         byte[] skipBuffer = new byte[size];
         while (remaining > 0) {
             nr = read(skipBuffer, 0, (int)Math.min(size, remaining));
             if (nr < 0) {
             	//到达文件末尾
                 break;
             }
             remaining -= nr;
         }

         return n - remaining;
     }
     ```

### 返回下一个方法可以不阻塞的从这个input stream中读取或跳过的预计字节数量。下一个方法可以是本线程或是别的线程。一些`InputStream`的实现会返回input stream的所有字节数量，一些则不会返回。 所以用这个返回值来确定一个缓冲长度来确定接受input stream中的数据是不准确的。

     ```
     public int available() throws IOException {
         return 0;
     }
     ```

### 关闭流比释放流相关的系统资源。

     ```
     public void close() throws IOException {}
     ```

### 标记input stream中的当前位置。接下来如果调用`reset`方法将流的位置重置到标记的位置，则后续的`read`方法则会重复读取一些字节。`readlimit`标志标记位置是失效前可以读取多少字节。通常的规范是，如果`markSupported`方法返回`true`,则stream会在调用`mark`方法之后记住所有读取过的字节来保证调用`reset`方法之后可以重复读取这些数据。但是如果这些数据超过`readlimit`则不需要记住。

     `mark`方法作用在一个关闭的流中没有什么效果。

     ```
     public synchronized void mark(int readlimit) {}
     ```

### 将位置重置到上个`mark`方法标记的位置。

     如果`markSupported`返回`true`,如果流创建之后`mark`方法还没调用，或者`mark`方法调用之后读取的字节数超过`mark`方法的参数，则可能会抛出`IOException`,如果错误没有抛出，则stream会重置到这样一个状态：最近一个`mark`之后读取的所有字节（`mark`没调用则重置到文件开头）提供给后续的`read`方法，后面跟着`reset`方法调用时下个数据字节。

     如果`markSupported`返回`false`,调用本方法可能会抛出`IOException`,如果错误没有抛出，则根据流的类型和创建方法重置为一个固定状态。根据特定的input stream类型决定接下来的`read`方法处理的字节。

     ```
     public synchronized void reset() throws IOException {
         throw new IOException("mark/reset not supported");
     }
     ```

### 测试这个流是否支持`mark`和`reset`方法。

     ```
     public boolean markSupported() {
         return false;
     }
     ```