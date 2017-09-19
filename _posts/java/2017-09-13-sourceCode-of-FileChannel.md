---
layout: post
category: java
title:  "Source code of FileChannel in java."
comments: true
description: "Analyze source code of FileChannel ."
tags: [java,source]
---

## 说明

一个通道是用来读，写，映射，操作文件的。一个文件通道是就是一个连接文件的`SeekableByteChannel`，它有一个文件中的当前位置可以获取或修改。文件包含了一个可变长的字节序列既可以读也可以写，当前长度也可以获取。当在当前文件大小之后写入字节会增加文件大小，当被截取的时候会减小。文件还有其他一些元数据，比如：权限，内容类型，最后修改时间，本来没有定义方法来获取元数据。

文件通道在多线程下也可以安全使用。`close`动作可以在任何时间调用。那些会包含通道位置或改变文件大小的动作在任何指定时间只有一个在进行，当有第二个动作在第一个动作还在操作的时候初始化则会一直阻塞直到第一个动作完成（单线程）。其他操作，比如获取位置则可以并行进行（多线程），实际是怎么操作则取决于底层实现。

在同一个程序中，本类的不同实例反应的文件视图都是一样的。至于多线程下，则可能因为缓存或网络而不同。

实例可以通道调用本类的`open`方法来创建，也可以通过`java.io.FileInputStream` `java.io.FileOutputStream` `RandomAccessFile`的`getChannel`方法来获取。当通过流的`getChannel`方法获取时，则通道对文件进行的修改，流中可见，反之亦然。当通道是`FileInputStream.getChannel`获取的，则通道可读，`FileOutputStream.getChannel`获取的，则通道可写，如果是`RandomAccessFile.getChannel`获取的，则根据`RandomAccessFile`的读写来确定读写。

本类是一个抽象类。（下面的变量和方法写的是实现类`FileChannelImpl`里的）

## implements

```
extends AbstractInterruptibleChannel
implements SeekableByteChannel, GatheringByteChannel, ScatteringByteChannel
```

## fields

### 为映射缓存分配的内存大小

```
private static final long allocationGranularity;
```

### 用来执行本地的读写方法

```
private final FileDispatcher nd;
```

### 文件描述符

```
private final FileDescriptor fd;
```

### 文件属性标识（不可变）

```
private final boolean writable;//写
private final boolean readable;//读
private final boolean append;//追加
```

### 通道的来源流

```
private final Object parent;
```

### 文件路径

```
private final String path;
```

### 本地线程集

```
private final NativeThreadSet threads = new NativeThreadSet(2);
```

### 同步锁

```
private final Object positionLock = new Object();
```

### 认为底层核心支持`sendfile()`，如果之后发现不是，则设为`false`

```
private static volatile boolean transferSupported = true;
```

### 认为如果目标文件描述符是一个管道`sendfile()`会工作，如果之后发现不行，则设为`false`

```
private static volatile boolean pipeSupported = true;
```

### 认为如果目标文件描述符是一个文件`sendfile()`会工作，如果之后发现不行，则设为`false`

```
private static volatile boolean fileSupported = true;
```

### 使用映射缓存时映射的最大长度

```
private static final long MAPPED_TRANSFER_SIZE = 8388608L;
```

### //

```
private static final int TRANSFER_SIZE = 8192;
```

### 映射标识

```
private static final int MAP_RO = 0;
private static final int MAP_RW = 1;
private static final int MAP_PV = 2;
```

### 跟踪文件上的锁

```
private volatile FileLockTable fileLockTable;
```

### 标识系统级持有文件锁

```
private static boolean isSharedFileLockTable;
```

###  标识`disableSystemWideOverlappingFileLockCheck `属性是否被选择

```
private static volatile boolean propertyChecked;
```

## methods

### 构造器

```
private FileChannelImpl(FileDescriptor fd, boolean readable,
	boolean writable, boolean append, Object parent)
{
  this.fd = fd;
  this.readable = readable;
  this.writable = writable;
  this.append = append;
  this.parent = parent;
  this.nd = new FileDispatcherImpl(append);
}
```

### 使用`FileInputStream.getChannel()`  `RandomAccessFile.getChannel()`

```
public static FileChannel open(FileDescriptor fd,
    boolean readable, boolean writable,
    Object parent)
{
	return new FileChannelImpl(fd, readable, writable, false, parent);
}
```

### 使用`FileOutputStream.getChannel()`

```
public static FileChannel open(FileDescriptor fd,
    boolean readable, boolean writable,
    boolean append, Object parent)
{
	return new FileChannelImpl(fd, readable, writable, append, parent);
}
```

### 获取通道开启状态

```
private void ensureOpen() throws IOException {
    if (!isOpen())
        throw new ClosedChannelException();
}
```

### 关闭通道

```
protected void implCloseChannel() throws IOException {
    //释放并无效化我们还持有的所有锁
    if (fileLockTable != null) {
        for (FileLock fl: fileLockTable.removeAll()) {
            synchronized (fl) {
                if (fl.isValid()) {
                    nd.release(fd, fl.position(), fl.size());
                    ((FileLockImpl)fl).invalidate();
                }
            }
        }
    }

    nd.preClose(fd);
    threads.signalAndWait();

    if (parent != null) {
        //让通道的来源流执行close方法。流会再调用本类的close方法，这个是在父类中定义的
        //但是父类里的isOpen方法会避免重新调用。
        ((java.io.Closeable)parent).close();
    } else {
        nd.close(fd);
    }

}
```

### 从通道中读取字节到指定缓存中

```
public int read(ByteBuffer dst) throws IOException {
    ensureOpen();//通道未关闭
    if (!readable)
        throw new NonReadableChannelException();
    synchronized (positionLock) {
        int n = 0;
        int ti = -1;
        try {
            begin();
            ti = threads.add();//加入线程组
            if (!isOpen())
                return 0;
            do {
                n = IOUtil.read(fd, dst, -1, nd);
            } while ((n == IOStatus.INTERRUPTED) && isOpen());
            return IOStatus.normalize(n);
        } finally {
            threads.remove(ti);
            end(n > 0);
            assert IOStatus.check(n);
        }
    }
}
```

### 从通道中读取字节到缓存子序列中

```
public long read(ByteBuffer[] dsts, int offset, int length)
        throws IOException
{
    if ((offset < 0) || (length < 0) || (offset > dsts.length - length))
        throw new IndexOutOfBoundsException();
    ensureOpen();
    if (!readable)
        throw new NonReadableChannelException();
    synchronized (positionLock) {
        long n = 0;
        int ti = -1;
        try {
            begin();
            ti = threads.add();
            if (!isOpen())
                return 0;
            do {
                n = IOUtil.read(fd, dsts, offset, length, nd);
            } while ((n == IOStatus.INTERRUPTED) && isOpen());
            return IOStatus.normalize(n);
        } finally {
            threads.remove(ti);
            end(n > 0);
            assert IOStatus.check(n);
        }
    }
}
```

### 写入数组到缓存中

```
public int write(ByteBuffer src) throws IOException {
    ensureOpen();
    if (!writable)
        throw new NonWritableChannelException();
    synchronized (positionLock) {
        int n = 0;
        int ti = -1;
        try {
            begin();
            ti = threads.add();
            if (!isOpen())
                return 0;
            do {
                n = IOUtil.write(fd, src, -1, nd);
            } while ((n == IOStatus.INTERRUPTED) && isOpen());
            return IOStatus.normalize(n);
        } finally {
            threads.remove(ti);
            end(n > 0);
            assert IOStatus.check(n);
        }
    }
}

public long write(ByteBuffer[] srcs, int offset, int length)
        throws IOException
{
    if ((offset < 0) || (length < 0) || (offset > srcs.length - length))
        throw new IndexOutOfBoundsException();
    ensureOpen();
    if (!writable)
        throw new NonWritableChannelException();
    synchronized (positionLock) {
        long n = 0;
        int ti = -1;
        try {
            begin();
            ti = threads.add();
            if (!isOpen())
                return 0;
            do {
                n = IOUtil.write(fd, srcs, offset, length, nd);
            } while ((n == IOStatus.INTERRUPTED) && isOpen());
            return IOStatus.normalize(n);
        } finally {
            threads.remove(ti);
            end(n > 0);
            assert IOStatus.check(n);
        }
    }
}
```

### 获取文件中的位置

```
public long position() throws IOException {
    ensureOpen();
    synchronized (positionLock) {
        long p = -1;
        int ti = -1;
        try {
            begin();
            ti = threads.add();
            if (!isOpen())
                return 0;
            do {
                //在追加模式下，在写入前位置移动到文件末尾
                p = (append) ? nd.size(fd) : position0(fd, -1);
            } while ((p == IOStatus.INTERRUPTED) && isOpen());
            return IOStatus.normalize(p);
        } finally {
            threads.remove(ti);
            end(p > -1);
            assert IOStatus.check(p);
        }
    }
}
```

### 设置文件中的位置

```
public FileChannel position(long newPosition) throws IOException {
    ensureOpen();
    if (newPosition < 0)
        throw new IllegalArgumentException();
    synchronized (positionLock) {
        long p = -1;
        int ti = -1;
        try {
            begin();
            ti = threads.add();
            if (!isOpen())
                return null;
            do {
                p  = position0(fd, newPosition);
            } while ((p == IOStatus.INTERRUPTED) && isOpen());
            return this;
        } finally {
            threads.remove(ti);
            end(p > -1);
            assert IOStatus.check(p);
        }
    }
}
```

### 获取通道中的文件的当前大小

```
public long size() throws IOException {
    ensureOpen();
    synchronized (positionLock) {
        long s = -1;
        int ti = -1;
        try {
            begin();
            ti = threads.add();
            if (!isOpen())
                return -1;
            do {
                s = nd.size(fd);
            } while ((s == IOStatus.INTERRUPTED) && isOpen());
            return IOStatus.normalize(s);
        } finally {
            threads.remove(ti);
            end(s > -1);
            assert IOStatus.check(s);
        }
    }
}
```

### 截取文件到指定大小，如果指定大小小于文件当前大小，多出的部分会被丢弃。如果大于等于，则不会修改文件。但是不管是哪种情况，文件中的位置大于指定大小，文件应该被修改到那个位置。

```
public FileChannel truncate(long newSize) throws IOException {
    ensureOpen();
    if (newSize < 0)
        throw new IllegalArgumentException("Negative size");
    if (!writable)
        throw new NonWritableChannelException();
    synchronized (positionLock) {
        int rv = -1;
        long p = -1;
        int ti = -1;
        try {
            begin();
            ti = threads.add();
            if (!isOpen())
                return null;

            //获取当前大小
            long size;
            do {
                size = nd.size(fd);
            } while ((size == IOStatus.INTERRUPTED) && isOpen());
            if (!isOpen())
                return null;

            // get current position
            do {
                p = position0(fd, -1);
            } while ((p == IOStatus.INTERRUPTED) && isOpen());
            if (!isOpen())
                return null;
            assert p >= 0;

            // truncate file if given size is less than the current size
            if (newSize < size) {
                do {
                    rv = nd.truncate(fd, newSize);
                } while ((rv == IOStatus.INTERRUPTED) && isOpen());
                if (!isOpen())
                    return null;
            }

            // if position is beyond new size then adjust it
            if (p > newSize)
                p = newSize;
            do {
                rv = (int)position0(fd, p);
            } while ((rv == IOStatus.INTERRUPTED) && isOpen());
            return this;
        } finally {
            threads.remove(ti);
            end(rv > -1);
            assert IOStatus.check(rv);
        }
    }
}
```

### 对通道的文件的任何修改写入到存储设备中。

```
public void force(boolean metaData) throws IOException {
    ensureOpen();
    int rv = -1;
    int ti = -1;
    try {
        begin();
        ti = threads.add();
        if (!isOpen())
            return;
        do {
            rv = nd.force(fd, metaData);
        } while ((rv == IOStatus.INTERRUPTED) && isOpen());
    } finally {
        threads.remove(ti);
        end(rv > -1);
        assert IOStatus.check(rv);
    }
}
```

### 直接写入到指定通道中

```
private long transferToDirectly(long position, int icount,
                                WritableByteChannel target)
        throws IOException
{
    if (!transferSupported)
        return IOStatus.UNSUPPORTED;

    FileDescriptor targetFD = null;
    if (target instanceof FileChannelImpl) {
        if (!fileSupported)
            return IOStatus.UNSUPPORTED_CASE;
        targetFD = ((FileChannelImpl)target).fd;
    } else if (target instanceof SelChImpl) {
        //直接传输到管道在某些配置中会造成EINVAL（参数无效）
        if ((target instanceof SinkChannelImpl) && !pipeSupported)
            return IOStatus.UNSUPPORTED_CASE;
        targetFD = ((SelChImpl)target).getFD();
    }
    if (targetFD == null)
        return IOStatus.UNSUPPORTED;
    int thisFDVal = IOUtil.fdVal(fd);
    int targetFDVal = IOUtil.fdVal(targetFD);
    if (thisFDVal == targetFDVal) // 在某些配置中不支持
        return IOStatus.UNSUPPORTED;

    long n = -1;
    int ti = -1;
    try {
        begin();
        ti = threads.add();
        if (!isOpen())
            return -1;
        do {
            n = transferTo0(thisFDVal, position, icount, targetFDVal);
        } while ((n == IOStatus.INTERRUPTED) && isOpen());
        if (n == IOStatus.UNSUPPORTED_CASE) {//如果不支持，修改标识
            if (target instanceof SinkChannelImpl)
                pipeSupported = false;
            if (target instanceof FileChannelImpl)
                fileSupported = false;
            return IOStatus.UNSUPPORTED_CASE;
        }
        if (n == IOStatus.UNSUPPORTED) {
            //再试一遍
            //如果不支持，修改标识
            transferSupported = false;
            return IOStatus.UNSUPPORTED;
        }
        return IOStatus.normalize(n);
    } finally {
        threads.remove(ti);
        end (n > -1);
    }
}
```

### 传输到信任通道

```
private long transferToTrustedChannel(long position, long count,
                                      WritableByteChannel target)
        throws IOException
{
    boolean isSelChImpl = (target instanceof SelChImpl);
    if (!((target instanceof FileChannelImpl) || isSelChImpl))
        return IOStatus.UNSUPPORTED;

    //信任目标：用一个映射的缓存
    long remaining = count;
    while (remaining > 0L) {
        long size = Math.min(remaining, MAPPED_TRANSFER_SIZE);
        try {
            MappedByteBuffer dbb = map(MapMode.READ_ONLY, position, size);
            try {
                //Bug：关闭通道不能终止写入动作
                int n = target.write(dbb);
                assert n >= 0;
                remaining -= n;
                if (isSelChImpl) {
                    //试图写入到SelectableChannel
                    break;
                }
                assert n > 0;
                position += n;
            } finally {
                unmap(dbb);
            }
        } catch (ClosedByInterruptException e) {
            //目标被中断则在关闭本通道后需要再抛出异常
            assert !target.isOpen();
            try {
                close();
            } catch (Throwable suppressed) {
                e.addSuppressed(suppressed);
            }
            throw e;
        } catch (IOException ioe) {
            //只有在没有字节被写入时才抛出
            if (remaining == count)
                throw ioe;
            break;
        }
    }
    return count - remaining;
}
```
### 传输到任意的通道	

```
private long transferToArbitraryChannel(long position, int icount,
                                        WritableByteChannel target)
        throws IOException
{
    //不信任的通道 ：使用一个临时的通道 
    int c = Math.min(icount, TRANSFER_SIZE);
    ByteBuffer bb = Util.getTemporaryDirectBuffer(c);
    long tw = 0;                    // 写入的字节总数
    long pos = position;
    try {
        Util.erase(bb);
        while (tw < icount) {
            bb.limit(Math.min((int)(icount - tw), TRANSFER_SIZE));
            int nr = read(bb, pos);
            if (nr <= 0)
                break;
            bb.flip();
            //Bug：如果通道被异步关闭，写入的通道会阻塞
            int nw = target.write(bb);
            tw += nw;
            if (nw != nr)
                break;
            pos += nw;
            bb.clear();
        }
        return tw;
    } catch (IOException x) {
        if (tw > 0)
            return tw;
        throw x;
    } finally {
        Util.releaseTemporaryDirectBuffer(bb);
    }
}
```

### 从本通道的文件传输字节到目标通道。会试图读取通道中从当前位置开始的总字节数并将他们写入目标通道 。本方法可能不会传输所有需求的字节，取决于类型或通道的状态。比需求字节更少可能因为通道中文件从当前位置开始字节数不足，或可以因为目标通道是非阻塞的且它的输出缓存上空闲资源不足。本方法不会修改文件位置 ，如果指定的位置文件当前长度，则不会传输字节。许多操作系统会直接从文件系统缓存中传输字节去目标通道而不是复制。

```
public long transferTo(long position, long count,
                       WritableByteChannel target)
        throws IOException
{
    ensureOpen();
    if (!target.isOpen())//目标关闭
        throw new ClosedChannelException();
    if (!readable)
        throw new NonReadableChannelException();
    if (target instanceof FileChannelImpl &&
            !((FileChannelImpl)target).writable)
        throw new NonWritableChannelException();//目标不可写
    if ((position < 0) || (count < 0))
        throw new IllegalArgumentException();
    long sz = size();
    if (position > sz)
        return 0;
    int icount = (int)Math.min(count, Integer.MAX_VALUE);
    if ((sz - position) < icount)
        icount = (int)(sz - position);

    long n;

    //如果内核支持，试图直接传输
    if ((n = transferToDirectly(position, icount, target)) >= 0)
        return n;

    //试图用映射传输去信任通道类型
    if ((n = transferToTrustedChannel(position, icount, target)) >= 0)
        return n;

    // 低速的不信任目标 
    return transferToArbitraryChannel(position, icount, target);
}
```

### 传输自文件通道

```
private long transferFromFileChannel(FileChannelImpl src,
                                     long position, long count)
        throws IOException
{
    if (!src.readable)
        throw new NonReadableChannelException();
    synchronized (src.positionLock) {
        long pos = src.position();
        long max = Math.min(count, src.size() - pos);

        long remaining = max;
        long p = pos;
        while (remaining > 0L) {
            long size = Math.min(remaining, MAPPED_TRANSFER_SIZE);
            //Bug：关闭通道不会终止写入
            MappedByteBuffer bb = src.map(MapMode.READ_ONLY, p, size);
            try {
                long n = write(bb, position);
                assert n > 0;
                p += n;
                position += n;
                remaining -= n;
            } catch (IOException ioe) {
                //只有在没有字节写入的情况下会抛出异常
                if (remaining == max)
                    throw ioe;
                break;
            } finally {
                unmap(bb);
            }
        }
        long nwritten = max - remaining;
        src.position(pos + nwritten);
        return nwritten;
    }
}
```

### 传输自任意通道

```
private long transferFromArbitraryChannel(ReadableByteChannel src,
                                          long position, long count)
        throws IOException
{
    //不信任的通道：使用一个临时的缓存
    int c = (int)Math.min(count, TRANSFER_SIZE);
    ByteBuffer bb = Util.getTemporaryDirectBuffer(c);
    long tw = 0;                    // 写入总字节
    long pos = position;
    try {
        Util.erase(bb);
        while (tw < count) {
            bb.limit((int)Math.min((count - tw), (long)TRANSFER_SIZE));
            //Bug: 如果本通道被异步关闭会导致源通道会阻塞
            int nr = src.read(bb);
            if (nr <= 0)
                break;
            bb.flip();
            int nw = write(bb, pos);
            tw += nw;
            if (nw != nr)
                break;
            pos += nw;
            bb.clear();
        }
        return tw;
    } catch (IOException x) {
        if (tw > 0)
            return tw;
        throw x;
    } finally {
        Util.releaseTemporaryDirectBuffer(bb);
    }
}
```

### 传输自通道。 

```
public long transferFrom(ReadableByteChannel src,
                         long position, long count)
        throws IOException
{
    ensureOpen();
    if (!src.isOpen())
        throw new ClosedChannelException();
    if (!writable)
        throw new NonWritableChannelException();
    if ((position < 0) || (count < 0))
        throw new IllegalArgumentException();
    if (position > size())
        return 0;
    if (src instanceof FileChannelImpl)
        return transferFromFileChannel((FileChannelImpl)src,
                position, count);

    return transferFromArbitraryChannel(src, position, count);
}
```

### 从通道中文件的指定起始位置开始读取字节到缓存中，本方法与`read(ByteBuffer)`方法一致，除了字节从指定位置开始而不是当前位置。

```
public int read(ByteBuffer dst, long position) throws IOException {
    if (dst == null)
        throw new NullPointerException();
    if (position < 0)
        throw new IllegalArgumentException("Negative position");
    if (!readable)
        throw new NonReadableChannelException();
    ensureOpen();
    if (nd.needsPositionLock()) {
        synchronized (positionLock) {
            return readInternal(dst, position);
        }
    } else {
        return readInternal(dst, position);
    }
}
```

### 读取方法

```
private int readInternal(ByteBuffer dst, long position) throws IOException {
    assert !nd.needsPositionLock() || Thread.holdsLock(positionLock);
    int n = 0;
    int ti = -1;
    try {
        begin();
        ti = threads.add();
        if (!isOpen())
            return -1;
        do {
            n = IOUtil.read(fd, dst, position, nd);
        } while ((n == IOStatus.INTERRUPTED) && isOpen());
        return IOStatus.normalize(n);
    } finally {
        threads.remove(ti);
        end(n > 0);
        assert IOStatus.check(n);
    }
}
```

### 通道中的文件从指定位置开始写传入的缓存

```
public int write(ByteBuffer src, long position) throws IOException {
    if (src == null)
        throw new NullPointerException();
    if (position < 0)
        throw new IllegalArgumentException("Negative position");
    if (!writable)
        throw new NonWritableChannelException();
    ensureOpen();
    if (nd.needsPositionLock()) {
        synchronized (positionLock) {
            return writeInternal(src, position);
        }
    } else {
        return writeInternal(src, position);
    }
}
```

### 写入方法

```
private int writeInternal(ByteBuffer src, long position) throws IOException {
    assert !nd.needsPositionLock() || Thread.holdsLock(positionLock);
    int n = 0;
    int ti = -1;
    try {
        begin();
        ti = threads.add();
        if (!isOpen())
            return -1;
        do {
            n = IOUtil.write(fd, src, position, nd);
        } while ((n == IOStatus.INTERRUPTED) && isOpen());
        return IOStatus.normalize(n);
    } finally {
        threads.remove(ti);
        end(n > 0);
        assert IOStatus.check(n);
    }
}
```

### 释放映射缓存

```
private static void unmap(MappedByteBuffer bb) {
    Cleaner cl = ((DirectBuffer)bb).cleaner();
    if (cl != null)
        cl.clean();
}
```

### 将通道中的文件域映射到内存中。

映射有三种模式：

* `Read-only`：试图修改结果缓存会抛出异常
* `Read/write`：对结果缓存做的修改最后都会被更新到文件中，对其他映射同个文件的程序不一定是可见的
* `Private`：对结果缓存做的修改最后不会被更新到文件中，对其他映射同个文件的程序也不可见的，而是会对缓存修改的部分进行私有化的拷贝。

缓存和映射直到垃圾回收之前都会是有效的。映射一旦创建，就和通道相互独立，关闭通道不会使映射失效。

对大多数操作系统而言，映射文件到内存比起使用常用的`read` `write`方法处理几十KB的文件来说是非常昂贵的。所以，为了最优的性能，只有在处理特别大的文件的时候才值得

```
public MappedByteBuffer map(MapMode mode, long position, long size)
        throws IOException
{
    ensureOpen();
    if (mode == null)
        throw new NullPointerException("Mode is null");
    if (position < 0L)
        throw new IllegalArgumentException("Negative position");
    if (size < 0L)
        throw new IllegalArgumentException("Negative size");
    if (position + size < 0)
        throw new IllegalArgumentException("Position + size overflow");
    if (size > Integer.MAX_VALUE)
        throw new IllegalArgumentException("Size exceeds Integer.MAX_VALUE");

    int imode = -1;
    if (mode == MapMode.READ_ONLY)
        imode = MAP_RO;
    else if (mode == MapMode.READ_WRITE)
        imode = MAP_RW;
    else if (mode == MapMode.PRIVATE)
        imode = MAP_PV;
    assert (imode >= 0);
    if ((mode != MapMode.READ_ONLY) && !writable)
        throw new NonWritableChannelException();
    if (!readable)
        throw new NonReadableChannelException();

    long addr = -1;
    int ti = -1;
    try {
        begin();
        ti = threads.add();
        if (!isOpen())
            return null;

        long filesize;
        do {
            filesize = nd.size(fd);
        } while ((filesize == IOStatus.INTERRUPTED) && isOpen());
        if (!isOpen())
            return null;

        if (filesize < position + size) { // 扩展文件大小
            if (!writable) {
                throw new IOException("Channel not open for writing " +
                        "- cannot extend file to required size");
            }
            int rv;
            do {
                rv = nd.truncate(fd, position + size);//截取文件大小
            } while ((rv == IOStatus.INTERRUPTED) && isOpen());
            if (!isOpen())
                return null;
        }
        if (size == 0) {
            addr = 0;
            //一个有效的文件描述符不是必须的
            FileDescriptor dummy = new FileDescriptor();
            if ((!writable) || (imode == MAP_RO))
                return Util.newMappedByteBufferR(0, 0, dummy, null);
            else
                return Util.newMappedByteBuffer(0, 0, dummy, null);
        }

        int pagePosition = (int)(position % allocationGranularity);
        long mapPosition = position - pagePosition;
        long mapSize = size + pagePosition;
        try {
            //如果map0没有抛出异常，则地址是有效的
            addr = map0(imode, mapPosition, mapSize);
        } catch (OutOfMemoryError x) {
            //如果OutOfMemoryError抛出表示已经耗尽内存，则强制垃圾回收并重新映射
            System.gc();
            try {
                Thread.sleep(100);
            } catch (InterruptedException y) {
                Thread.currentThread().interrupt();
            }
            try {
                addr = map0(imode, mapPosition, mapSize);
            } catch (OutOfMemoryError y) {
                //第二次也失败
                throw new IOException("Map failed", y);
            }
        }

        //在windows，或可能的其他平台，我们一个打开的文件描述符来执行某些映射操作
        FileDescriptor mfd;
        try {
            mfd = nd.duplicateForMapping(fd);
        } catch (IOException ioe) {
            unmap0(addr, mapSize);
            throw ioe;
        }

        assert (IOStatus.checkAll(addr));
        assert (addr % allocationGranularity == 0);
        int isize = (int)size;
        Unmapper um = new Unmapper(addr, mapSize, isize, mfd);
        if ((!writable) || (imode == MAP_RO)) {
            return Util.newMappedByteBufferR(isize,
                    addr + pagePosition,
                    mfd,
                    um);
        } else {
            return Util.newMappedByteBuffer(isize,
                    addr + pagePosition,
                    mfd,
                    um);
        }
    } finally {
        threads.remove(ti);
        end(IOStatus.checkAll(addr));
    }
}
```

### 被`sun.management.ManagementFactoryHelper`调用给映射缓存创建一个管理接口

```
public static sun.misc.JavaNioAccess.BufferPool getMappedBufferPool() {
    return new sun.misc.JavaNioAccess.BufferPool() {
        @Override
        public String getName() {
            return "mapped";
        }
        @Override
        public long getCount() {
            return Unmapper.count;
        }
        @Override
        public long getTotalCapacity() {
            return Unmapper.totalCapacity;
        }
        @Override
        public long getMemoryUsed() {
            return Unmapper.totalSize;
        }
    };
}
```

### 保存在每个`FileChannel`实例本地的锁序列（J2SE 1.4/5.0中），当同个文件上有多少通道，重叠检测不是系统级的。

```
private static boolean isSharedFileLockTable() {
    if (!propertyChecked) {
        synchronized (FileChannelImpl.class) {
            if (!propertyChecked) {
                String value = AccessController.doPrivileged(
                        new GetPropertyAction(
                                "sun.nio.ch.disableSystemWideOverlappingFileLockCheck"));
                isSharedFileLockTable = ((value == null) || value.equals("false"));
                propertyChecked = true;
            }
        }
    }
    return isSharedFileLockTable;
}
```

### 跟踪文件锁

```
private FileLockTable fileLockTable() throws IOException {
    if (fileLockTable == null) {
        synchronized (this) {
            if (fileLockTable == null) {
                if (isSharedFileLockTable()) {
                    int ti = threads.add();
                    try {
                        ensureOpen();
                        fileLockTable = FileLockTable.newSharedFileLockTable(this, fd);
                    } finally {
                        threads.remove(ti);
                    }
                } else {
                    fileLockTable = new SimpleFileLockTable();
                }
            }
        }
    }
    return fileLockTable;
}
```

### 获取文件上的锁。调用该方法会阻塞直到文件可以重新上锁或本通道关闭或调用文件的线程被中断。当本通道调用该方法时被中断会抛出`AsynchronousCloseException`，当调用的线程在等待获取锁的时候被中断了，则会设置该线程的中断状态并用抛出一个`FileLockInterruptionException`,当方法调用时调用者的中断状态会被设置，异常立即抛出，线程的中断状态不会改变。加锁的部分由`position` `size`指定，但不是必须在真实文件中。如果当加锁区域由文件的末尾初始化然后文件扩展到该区域之外 ，则文件新增部分不会被锁覆盖。如果预期文件会扩展而且需要一个全文件的锁，则区域需要从0开始到预期的最大数量 。`Lock()`方法就是简单的锁了从0开始到`Long.MAX_VALUE`。有些操作系统不支持共享锁，则获取共享锁会被自动转换成获取排他锁，新获取的锁是共享的还是排他的可以用`FileLock.isShared()`方法来获取。

```
public FileLock lock(long position, long size, boolean shared)
        throws IOException
{
    ensureOpen();
    if (shared && !readable)
        throw new NonReadableChannelException();
    if (!shared && !writable)
        throw new NonWritableChannelException();
    FileLockImpl fli = new FileLockImpl(this, position, size, shared);
    FileLockTable flt = fileLockTable();
    flt.add(fli);
    boolean completed = false;
    int ti = -1;
    try {
        begin();
        ti = threads.add();
        if (!isOpen())
            return null;
        int n;
        do {
            n = nd.lock(fd, true, position, size, shared);
        } while ((n == FileDispatcher.INTERRUPTED) && isOpen());
        if (isOpen()) {
            if (n == FileDispatcher.RET_EX_LOCK) {
                assert shared;
                FileLockImpl fli2 = new FileLockImpl(this, position, size,
                        false);
                flt.replace(fli, fli2);
                fli = fli2;
            }
            completed = true;
        }
    } finally {
        if (!completed)
            flt.remove(fli);
        threads.remove(ti);
        try {
            end(completed);
        } catch (ClosedByInterruptException e) {
            throw new FileLockInterruptionException();
        }
    }
    return fli;
}
```

### 试图获取一个锁，该方法不会阻塞。调用都会立即返回，或是获得锁或是失败。当因为做重叠检测的时候发现锁被别的程序持有，返回`null`，其他原因则会抛出异常。

```
public FileLock tryLock(long position, long size, boolean shared)
        throws IOException
{
    ensureOpen();
    if (shared && !readable)
        throw new NonReadableChannelException();
    if (!shared && !writable)
        throw new NonWritableChannelException();
    FileLockImpl fli = new FileLockImpl(this, position, size, shared);
    FileLockTable flt = fileLockTable();
    flt.add(fli);
    int result;

    int ti = threads.add();
    try {
        try {
            ensureOpen();
            result = nd.lock(fd, false, position, size, shared);
        } catch (IOException e) {
            flt.remove(fli);
            throw e;
        }
        if (result == FileDispatcher.NO_LOCK) {
            flt.remove(fli);
            return null;
        }
        if (result == FileDispatcher.RET_EX_LOCK) {
            assert shared;
            FileLockImpl fli2 = new FileLockImpl(this, position, size,
                    false);
            flt.replace(fli, fli2);
            return fli2;
        }
        return fli;
    } finally {
        threads.remove(ti);
    }
}
```

### 释放锁

```
void release(FileLockImpl fli) throws IOException {
    int ti = threads.add();
    try {
        ensureOpen();
        nd.release(fd, fli.position(), fli.size());
    } finally {
        threads.remove(ti);
    }
    assert fileLockTable != null;
    fileLockTable.remove(fli);
}
```

### 创建映射

```
private native long map0(int prot, long position, long length)
        throws IOException;
```

### 移除已存在的映射

```
private static native int unmap0(long address, long length);
```

### 从源传输到目的地，如果内核不支持会返回`-2`

```
private native long transferTo0(int src, long position, long count, int dst);
```

### 获取或设置位置，如果`offset=-1`，返回当前位置，否则设置到该位置

```
private native long position0(FileDescriptor fd, long offset);
```

### 缓存`fieldIDs`

```
private static native long initIDs();

static {
    IOUtil.load();
    allocationGranularity = initIDs();
}
```

### 内部类，一个简单的文件锁表维护文件通道获取的文件锁列表。

```
private static class SimpleFileLockTable extends FileLockTable {
    //供获取的同步list
    private final List<FileLock> lockList = new ArrayList<FileLock>(2);

    public SimpleFileLockTable() {
    }

    private void checkList(long position, long size)
            throws OverlappingFileLockException
    {
        assert Thread.holdsLock(lockList);
        for (FileLock fl: lockList) {
            if (fl.overlaps(position, size)) {
                throw new OverlappingFileLockException();
            }
        }
    }

    public void add(FileLock fl) throws OverlappingFileLockException {
        synchronized (lockList) {
            checkList(fl.position(), fl.size());
            lockList.add(fl);
        }
    }

    public void remove(FileLock fl) {
        synchronized (lockList) {
            lockList.remove(fl);
        }
    }

    public List<FileLock> removeAll() {
        synchronized(lockList) {
            List<FileLock> result = new ArrayList<FileLock>(lockList);
            lockList.clear();
            return result;
        }
    }

    public void replace(FileLock fl1, FileLock fl2) {
        synchronized (lockList) {
            lockList.remove(fl1);
            lockList.add(fl2);
        }
    }
}
```

### 内部类，内存映射缓存 

```
private static class Unmapper
        implements Runnable
{
    //可能获取用来关闭文件
    private static final NativeDispatcher nd = new FileDispatcherImpl();

    //跟踪映射缓存使用量
    static volatile int count;
    static volatile long totalSize;
    static volatile long totalCapacity;

    private volatile long address;
    private final long size;
    private final int cap;
    private final FileDescriptor fd;

    private Unmapper(long address, long size, int cap,
                     FileDescriptor fd)
    {
        assert (address != 0);
        this.address = address;
        this.size = size;
        this.cap = cap;
        this.fd = fd;

        synchronized (Unmapper.class) {
            count++;
            totalSize += size;
            totalCapacity += cap;
        }
    }

    public void run() {
        if (address == 0)
            return;
        unmap0(address, size);
        address = 0;

        // if this mapping has a valid file descriptor then we close it
        if (fd.valid()) {
            try {
                nd.close(fd);
            } catch (IOException ignore) {
                // nothing we can do
            }
        }

        synchronized (Unmapper.class) {
            count--;
            totalSize -= size;
            totalCapacity -= cap;
        }
    }
}
```