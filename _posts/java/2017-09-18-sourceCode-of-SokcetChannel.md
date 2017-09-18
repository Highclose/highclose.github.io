---
layout: post
category: java
title:  "Source code of SocketChannel in java."
comments: true
description: "Analyze source code of SocketChannel ."
tags: [java,source]
---

## 说明

面向流的连接端可选择通道。调用本类的`open`方法创建一个端通道。不能创建一个任意的，已经存在的端的通道。一个新创建的端通道已经打开但是没有连接。企图在未连接的通道上执行IO操作会抛出`NotYetConnectedException`异常。一个端通道可以调用`connect`方法来连接，在关闭前一直保持连接状态。可以用`isConnected`方法来获取通道的连接状态。

端通道支持非阻塞连接：一个端通道可以被创建并且建连接的过程可以由`connect`方法初始化，`finishConnect`方法完成初始化。连接是否在进行中则可以调用`isConnectionPending`。

端通道支持异步关闭。非常类似于`Channel`类中的异步关闭操作。当端的输入方在别的线程因为读操作而阻塞在通道上是被另一个线程关闭，则阻塞的线程会不读取任何字节并返回`-1`。如果端的输出方被异步关闭，则阻塞的线程会收到`AsynchronousCloseException`

端选项可以用`setOption`方法配置。支持以下选项。
| 选项名                                      | 描述                    |
| ---------------------------------------- | --------------------- |
| java.net.StandardSocketOptions.SO_SNDBUF | 端发送缓存大小               |
| java.net.StandardSocketOptions.SO_RCVBUF | 端接收缓存大小               |
| java.net.StandardSocketOptions.SO_KEEPALIVE | 保持连接                  |
| java.net.StandardSocketOptions.SO_REUSEADDR | 重用地址                  |
| java.net.StandardSocketOptions.SO_LINGER | 如果有数据则延迟关闭（只有在阻塞模式）   |
| java.net.StandardSocketOptions.TCP_NODELAY | 关闭`Nagle algorithm`算法 |

端通道在多线程下可以安全使用。虽然在某个指定的时间点，最多只有一个线程在读，最多只有一个线程在写。

## implements

```
extends AbstractSelectableChannel
implements ByteChannel, ScatteringByteChannel, GatheringByteChannel, NetworkChannel
```

## methods

### 构造器

```
protected SocketChannel(SelectorProvider provider) {
    super(provider);
}
```

### 打开一个通道

```
public static SocketChannel open() throws IOException {
    return SelectorProvider.provider().openSocketChannel();
}
```

### 打开一个通道并且连接到远程地址。

```
public static SocketChannel open(SocketAddress remote)
    throws IOException
{
    SocketChannel sc = open();
    try {
        sc.connect(remote);
    } catch (Throwable x) {
        try {
            sc.close();
        } catch (Throwable suppressed) {
            x.addSuppressed(suppressed);
        }
        throw x;
    }
    assert sc.isConnected();
    return sc;
}
```

### 返回通道支持的操作集

```
public final int validOps() {
    return (SelectionKey.OP_READ
            | SelectionKey.OP_WRITE
            | SelectionKey.OP_CONNECT);
}
```

### 绑定

```
public abstract SocketChannel bind(SocketAddress local)
    throws IOException;
```

### 设置选项

```
public abstract <T> SocketChannel setOption(SocketOption<T> name, T value)
    throws IOException;
```

### 关闭读取连接而不关闭通道，一旦关闭读取则后续的读取操作都会返回`-1`。如果输入已经关闭了，则调用本方法没有作用。

```
public abstract SocketChannel shutdownInput() throws IOException;
```

### 关闭输出而不关闭通道，关闭之后再写入则会抛出`ClosedChannelException`

```
public abstract SocketChannel shutdownOutput() throws IOException;
```

### 返回与本通道关联的`Socket`

```
public abstract Socket socket();
```

### 是否已连接

```
public abstract boolean isConnected();
```

### 通道中的一个连接操作是否在进行中。只有在连接操作已经初始化但没有通过`finishConnect`方法完成时才返回`true`

```
public abstract boolean isConnectionPending();
```

### 连接本通道的端。如果本通道是非阻塞的，调用本方法表示一个非阻塞连接操作。如果连接立即建立，就好像本地连接，本方法就会返回`true`。否则本方法返回`false`，之后调用`finishConnect`完成连接操作。如果通道是阻塞的，本方法会一直阻塞直到连接建立。本方法会调用跟`Socket`一样的安全检查。本方法可以在任意时间调用。如果一个读取或写入操作在本方法过程中调用，则这些操作会阻塞到本方法完成。

```
public abstract boolean connect(SocketAddress remote) throws IOException;
```

### 完成连接端通道的动作。一个非阻塞操作连接操作的初始化首先配置通道的非阻塞模式然后调用`connect`方法。一旦连接建立，或者试图建立失败，则通道变成可连接并调用本方法完成连接。如果连接操作失败，调用本方法会抛出`IOException`。如果本通道是已经连接的，本方法不会阻塞并立即返回`true`。如果通道是非阻塞模式并且还未完成则会返回`false`。如果通道是阻塞模式，本方法会在连接完成或失败之前一直阻塞。

```
public abstract boolean finishConnect() throws IOException;
```

### 获取本通道连接的远端地址。

```
public abstract SocketAddress getRemoteAddress() throws IOException;
```

### 读写操作

```
public abstract int read(ByteBuffer dst) throws IOException;

public abstract long read(ByteBuffer[] dsts, int offset, int length)
    throws IOException;

public final long read(ByteBuffer[] dsts) throws IOException {
    return read(dsts, 0, dsts.length);
}

public abstract int write(ByteBuffer src) throws IOException;

public abstract long write(ByteBuffer[] srcs, int offset, int length)
    throws IOException;

public final long write(ByteBuffer[] srcs) throws IOException {
    return write(srcs, 0, srcs.length);
}
```

### 获取本地地址

```
public abstract SocketAddress getLocalAddress() throws IOException;
```