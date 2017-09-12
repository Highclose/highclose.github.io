---
layout: post
category: java
title:  "Source code of RandomAccessFile in java."
comments: true
description: "Analyze source code of RandomAccessFile ."
tags: [java,source]
---

## 说明

本类的实例支持同一个文件的读写。一个`RandomAccessFile`就像是保存在文件系统中的一个大的字节数组，数组上有一个指针或是索引，叫做`file pointer`。读取操作从这个指针的位置开始读取然后边读边移动指针。如果创建实例时设置成读/写模式，写入操作也同样可以。同样也是在指针位置开始写入，然后边写边移动指针。当写入操作超过文件当前的末尾会扩容数组。这个指针可以用`getFilePointer`获取或`seek`来设置。

## implements

```
 implements DataOutput, DataInput, Closeable 
```

## fields

### 文件描述符

```
private FileDescriptor fd;
```

### 文件通道

```
private FileChannel channel = null;
```

### 同时读写

```
private boolean rw;
```

### 文件的路径

```
private final String path;
```

### 锁

```
private Object closeLock = new Object();
```

### 关闭标志

```
private volatile boolean closed = false;
```

### 读写标志

```
private static final int O_RDONLY = 1;//只读
private static final int O_RDWR =   2;//读写
private static final int O_SYNC =   4;//将文件的内容或元数据变动立即写入底层存储设备
private static final int O_DSYNC =  8;//将文件的内容变动立即写入底层存储设备
```

### 静态方法

```
static {
    initIDs();
}

private static native void initIDs();

```

## methods

### 构造器

```
public RandomAccessFile(File file, String mode)
    throws FileNotFoundException
{
    String name = (file != null ? file.getPath() : null);
    int imode = -1;
    if (mode.equals("r"))
        imode = O_RDONLY;
    else if (mode.startsWith("rw")) {
        imode = O_RDWR;//如果文件不存在，会创建
        rw = true;
        if (mode.length() > 2) {
            if (mode.equals("rws"))
                imode |= O_SYNC;
            else if (mode.equals("rwd"))
                imode |= O_DSYNC;
            else
                imode = -1;
        }
    }
    if (imode < 0)
        throw new IllegalArgumentException("Illegal mode \"" + mode
                                           + "\" must be one of "
                                           + "\"r\", \"rw\", \"rws\","
                                           + " or \"rwd\"");
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkRead(name);//是否可读
        if (rw) {
            security.checkWrite(name);//是否可写
        }
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
    open(name, imode);//打开文件
}


public RandomAccessFile(String name, String mode)
	throws FileNotFoundException
{
	this(name != null ? new File(name) : null, mode);
}
```

### 获取文件描述符

```
public final FileDescriptor getFD() throws IOException {
    if (fd != null) {
        return fd;
    }
    throw new IOException();
}
```

### 获取文件通道，通道的位置与`getFilePointer`方法返回的指针位移量相等

```
public final FileChannel getChannel() {
    synchronized (this) {
        if (channel == null) {
            channel = FileChannelImpl.open(fd, path, true, rw, this);
        }
        return channel;
    }
}
```

### 打开文件

```
private native void open0(String name, int mode)
    throws FileNotFoundException;

private void open(String name, int mode)
	throws FileNotFoundException {
	open0(name, mode);
}
```

### 读取一个字节

```
public int read() throws IOException {
    return read0();
}

private native int read0() throws IOException;
```

### 读取字节到数组中

```
private native int readBytes(byte b[], int off, int len) throws IOException;


public int read(byte b[], int off, int len) throws IOException {
    return readBytes(b, off, len);
}

public int read(byte b[]) throws IOException {
    return readBytes(b, 0, b.length);
}
```

### 读满数组

```
public final void readFully(byte b[]) throws IOException {
    readFully(b, 0, b.length);
}


public final void readFully(byte b[], int off, int len) throws IOException {
    int n = 0;
    do {
        int count = this.read(b, off + n, len - n);
        if (count < 0)
            throw new EOFException();
        n += count;
    } while (n < len);
}
```

### 跳过字节

```
public int skipBytes(int n) throws IOException {
    long pos;
    long len;
    long newpos;

    if (n <= 0) {
        return 0;
    }
    pos = getFilePointer();//指针位置
    len = length();//文件长度
    newpos = pos + n;
    if (newpos > len) {
        newpos = len;
    }
    seek(newpos);//设置位置

    //返回实际跳过的字节数
    return (int) (newpos - pos);
}
```

### 写入一个字节

```
public void write(int b) throws IOException {
    write0(b);
}

private native void write0(int b) throws IOException;
```

### 将数组写入

```
private native void writeBytes(byte b[], int off, int len) throws IOException;


public void write(byte b[]) throws IOException {
    writeBytes(b, 0, b.length);
}


public void write(byte b[], int off, int len) throws IOException {
    writeBytes(b, off, len);
}
```

### 获取文件指针

```
public native long getFilePointer() throws IOException;
```

### 设置文件指针

```
public void seek(long pos) throws IOException {
    if (pos < 0) {
        throw new IOException("Negative seek offset");
    } else {
        seek0(pos);
    }
}

private native void seek0(long pos) throws IOException;
```

### 获取文件长度

```
public native long length() throws IOException;
```

### 设置文件长度，如果实际长度大于设置的长度，则文件会被截取，否则文件会被扩展，扩展部分未定义

```
public native void setLength(long newLength) throws IOException;
```

### 关闭

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

### 读写基础类型

```
//其他方法与DataInputStream,DataOutputStream一致
public final boolean readBoolean() throws IOException {
    int ch = this.read();
    if (ch < 0)
        throw new EOFException();
    return (ch != 0);
}
```

