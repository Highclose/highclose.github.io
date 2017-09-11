---
layout: post
category: java
title:  "Source code of BufferedInputStream in java."
description: "Analyze source code of BufferedInputStream ."
tags: [java,source]
---

## 说明

可以给别的输出流提供缓存功能，并且支持`mark`和`reset`方法。当`BufferedInputStream`创建时，会创建一个内部的数组。当字节读取或者跳过时，底层流一次读取多个字节不停的填充内部的数组。

## implements

```
FilterInputStream
```

## fields

###  默认缓存大小

```
private static int DEFAULT_BUFFER_SIZE = 8192;
```

### 数组最大长度

```
private static int MAX_BUFFER_SIZE = Integer.MAX_VALUE - 8;
```

### 缓存数组，有需求时，可以被替换成别的数组

```
protected volatile byte buf[]; 
```

### 原子更新器

```
private static final
    AtomicReferenceFieldUpdater<BufferedInputStream, byte[]> bufUpdater =
    AtomicReferenceFieldUpdater.newUpdater
    (BufferedInputStream.class,  byte[].class, "buf");
```

### 缓存中有效字节长度

```
protected int count;
```

### 缓存中的当前位置，这是数组中下一个字节的索引，如果`pos`与`count`相等，下一次`read`或`skip`操作会获取很多的字节

```
protected int pos;
```

### 上一次`mark`方法纪录的位置。如果`markpos`不等于`-1`，`buf[markpos]`到`buf[pos-1]`所有的字节都必须保存在缓存数组中，直到`pos`和`markpos`超过`marklimit`才会丢弃

```
protected int markpos = -1;
```

### 记录位置最大保留长度，丢弃后，`markpos`置为`-1`

```
protected int marklimit;
```

## methods

### 构造器，默认缓存大小

```
public BufferedInputStream(InputStream in) {
    this(in, DEFAULT_BUFFER_SIZE);
}
```

### 构造器2

```
public BufferedInputStream(InputStream in, int size) {
    super(in);
    if (size <= 0) {
        throw new IllegalArgumentException("Buffer size <= 0");
    }
    buf = new byte[size];
}
```

### 检查底层输入流没有被关闭

```
private InputStream getInIfOpen() throws IOException {
    InputStream input = in;
    if (input == null)
        throw new IOException("Stream closed");
    return input;
}
```

### 检查缓存数组没有因为关闭而被置空

```
private byte[] getBufIfOpen() throws IOException {
    byte[] buffer = buf;
    if (buffer == null)
        throw new IOException("Stream closed");
    return buffer;
}
```

### 填充数组

```
private void fill() throws IOException {
	//数组不为空
    byte[] buffer = getBufIfOpen();
    if (markpos < 0)
        pos = 0;            /* 没有记录：丢弃缓存 */
    else if (pos >= buffer.length)  /* 缓存没有可用空间 */
        if (markpos > 0) {  /* 丢弃markpos以前的缓存 */
            int sz = pos - markpos;
            System.arraycopy(buffer, markpos, buffer, 0, sz);
            pos = sz;
            markpos = 0;
        } else if (buffer.length >= marklimit) {
            markpos = -1;   /* mark失效 */
            pos = 0;        /* 丢弃数据 */
        } else if (buffer.length >= MAX_BUFFER_SIZE) {
            throw new OutOfMemoryError("Required array size too large");
        } else {            /* 扩容数组 */
            int nsz = (pos <= MAX_BUFFER_SIZE - pos) ?
                    pos * 2 : MAX_BUFFER_SIZE;
            if (nsz > marklimit)
                nsz = marklimit;
            byte nbuf[] = new byte[nsz];
            System.arraycopy(buffer, 0, nbuf, 0, pos);
            if (!bufUpdater.compareAndSet(this, buffer, nbuf)) {
            	//当this的buf等于buffer时，将buf替换为nbuf
                //当被异步被关闭了，则不能替换。
                throw new IOException("Stream closed");
            }
            //替换数组
            buffer = nbuf;
        }
    count = pos;
    //将缓存读满
    int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
    if (n > 0)
        count = n + pos;
}
```

### 读取

```
public synchronized int read() throws IOException {
    if (pos >= count) {
        fill();
        if (pos >= count)
            return -1;
    }
    //0xff作用：读取低8位
    return getBufIfOpen()[pos++] & 0xff;
}
```

### 私有方法，读取到指定数组

```
private int read1(byte[] b, int off, int len) throws IOException {
    int avail = count - pos;
    if (avail <= 0) {//需要填充
        //如果需求长度大于等于缓存，而且没有mark/reset动作，则不需要将字节复制到内部数组中
        //底层流直接读取
        if (len >= getBufIfOpen().length && markpos < 0) {
            return getInIfOpen().read(b, off, len);
        }
        fill();
        avail = count - pos;
        //已经到末尾
        if (avail <= 0) return -1;
    }
    int cnt = (avail < len) ? avail : len;
    System.arraycopy(getBufIfOpen(), pos, b, off, cnt);
    pos += cnt;
    return cnt;
}
```

### 读取到指定数组。

```
public synchronized int read(byte b[], int off, int len)
    throws IOException
{
    getBufIfOpen(); 
    if ((off | len | (off + len) | (b.length - (off + len))) < 0) {
        throw new IndexOutOfBoundsException();
    } else if (len == 0) {
        return 0;
    }

    int n = 0;
    for (;;) {
        int nread = read1(b, off + n, len - n);
        if (nread <= 0)
            return (n == 0) ? nread : n;
        n += nread;
        if (n >= len)
            return n;
        //如果没有关闭或无字节有效，返回
        InputStream input = in;
        if (input != null && input.available() <= 0)
            return n;
    }
}
```

### 跳过字节

```
public synchronized long skip(long n) throws IOException {
    getBufIfOpen(); 
    if (n <= 0) {
        return 0;
    }
    long avail = count - pos;

    if (avail <= 0) {
        //如果没有记录过，则不需要保存在缓存中。
        if (markpos <0)
            return getInIfOpen().skip(n);

        //填充
        fill();
        avail = count - pos;
        if (avail <= 0)
            return 0;
    }

    long skipped = (avail < n) ? avail : n;
    pos += skipped;
    return skipped;
}
```

### 有效字节

```
public synchronized int available() throws IOException {
    int n = count - pos;
    //底层返回的有效字节数
    int avail = getInIfOpen().available();
    return n > (Integer.MAX_VALUE - avail)
                ? Integer.MAX_VALUE
                : n + avail;
}
```

### 记录

```
public synchronized void mark(int readlimit) {
    marklimit = readlimit;
    markpos = pos;
}
```

### 重置

```
public synchronized void reset() throws IOException {
    getBufIfOpen(); 
    if (markpos < 0)
        throw new IOException("Resetting to invalid mark");
    pos = markpos;
}
```

### 返回支持记录功能

```
public boolean markSupported() {
    return true;
}
```

### 关闭资源

```
public void close() throws IOException {
    byte[] buffer;
    while ( (buffer = buf) != null) {
        if (bufUpdater.compareAndSet(this, buffer, null)) {
            InputStream input = in;
            in = null;
            if (input != null)
                input.close();
            return;
        }
    }
}
```
## 其他

所有的`public`方法，除了`close`和`markSupport`以外，都用了synchronized来修饰以保证线程安全。而在`close`中，用了原子更新器来将数组置空，防止置空的时候被别的线程在`fill`方法中替换数组的时候重新赋值使`close`方法释放资源失败。