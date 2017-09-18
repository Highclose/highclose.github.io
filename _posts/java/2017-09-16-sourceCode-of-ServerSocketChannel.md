---
layout: post
category: java
title:  "Source code of ServerSocketChannel in java."
comments: true
description: "Analyze source code of ServerSocketChannel ."
tags: [java,source]
---

## 说明

代表一个面向流的监听端的可选择通道。使用本类的`open`方法创建一个`ServerSocketChannel`实例，不能创建一个随意或已存在的服务端。一个新创建的服务端通道已是打开状态但还没有绑定。当试图在未绑定的通道上调用`accept()`方法会抛出`NotYetBoundException`异常。绑定动作可以调用本类的`bind`方法。

端选项可以用`setOption`方法设定。支持以下选项：

| 选项名                                | 描述       |
| ---------------------------------- | -------- |
| StandardSocketOptions.SO_RCVBUF    | 端接受的缓存大小 |
| StandardSocketOptions.SO_REUSEADDR | 重用地址     |

 本类是线程安全。

##implements

```
extends AbstractSelectableChannel implements NetworkChannel
```

## methods

### 构造器

```
protected ServerSocketChannel(SelectorProvider provider) {
    super(provider);
}
```

### 创建一个服务端通道

```
public static ServerSocketChannel open() throws IOException {
    return SelectorProvider.provider().openServerSocketChannel();
}
```

### 返回通道支持的操作集合。服务端通道只支持接受新的链接。

```
public final int validOps() {
    return SelectionKey.OP_ACCEPT;
}
```

### 将通道端与本地地址绑定并监听链接。

```
public final ServerSocketChannel bind(SocketAddress local)
    throws IOException
{
    return bind(local, 0);
}
```

### 本方法调用之后，端和本地地址就建立了联系，一旦联系建立之后，端直到被关闭以前都是绑定状态。`backlog`参数是端上的最大挂起数量。

```
public abstract ServerSocketChannel bind(SocketAddress local, int backlog)
    throws IOException
```
### 设置选项

```
public abstract <T> ServerSocketChannel setOption(SocketOption<T> name, T value)
    throws IOException;
```

### 获取一个与通道连接的服务端

```
public abstract ServerSocket socket();
```

### 接受一个连接。非阻塞模式下，如果没有挂起的连接，该方法会立即返回`null`。如果是阻塞的，则会一直阻塞直到一个新的连接或者异常。该方法返回的`SocketChannel`无论本通道是否阻塞，都是阻塞的。本方法的安全检查与`ServerSocket.accpet()`方法完全一样。也就是说，如果已存在一个安全管理器则本方法会验证每个新建连接的的地址和端口（在`SecurityManager.checkAccpet`中允许）

```
public abstract SocketChannel accept() throws IOException;
```

### 如果配置了一个安全管理器，则管理器会用本地地址和`-1`做参数调用`checkConnect`方法去检测一个操作是否是许可的。如果操作是不被许可的，代表环回地址和本地端口的`SocketAddress`会被返回。如果是许可的，返回绑定到的端的`SocketAddress`或`null`(未绑定)

```
public abstract SocketAddress getLocalAddress() throws IOException;
```

