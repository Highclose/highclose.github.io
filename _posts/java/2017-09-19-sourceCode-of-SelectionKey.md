---
layout: post
category: java
title:  "Source code of SelectionKey in java."
comments: true
description: "Analyze source code of SelectionKey ."
tags: [java,source]
---

## 说明

可选择通道想选择器注册的一个符号。每一次通道向选择器注册时都会创建一个`SelectionKey`。一个键会一直有效直到它被取消。取消一个键不会立即把该键从选择器中移除，而是会先加入到选择器的已取消键集合中在下次选择操作时移除。可以调用`isValid`方法验证键的有效性。

一个`SelectionKey`包含了用数字表示的两个操作集合。操作集合的每一位都表示了键对应通道的一个可选择操作的种类。

* 关注集：决定了选择器下一次的选择方法执行时会检测是否就绪的种类。关注集用给定的值来创建，之后可以用`interestOps(int)`来修改。
* 就绪集：表示选择器检测已经就绪的操作集合。就绪集初始化时设为0，之后可以在选择动作中被选择器更新，但不能直接修改。

一个选择键的就绪集表示它的通道已经为某些操作准备好，但是只是一个提示，不是一个保证，这类集合中的操作可能不会是线程阻塞。一个就绪集在选择操作完成后马上就会很准确。不准确可能是由外部事件或者相应线程的IO操作引起的。

本类定义了所有已知的操作集，但是根据通道的类型，支持的集合也有所不同。

经常会需要给一些应用相关的数据关联一个`SelectionKey`，比如，一个对象代表一个高层协议的状态并且操作就绪通知来具体实现协议。`SelectionKey`支持任意对象附带一个键。一个对象可以调用`attach`方法附加一个键之后可以用`attachment`方法来获取。

本类线程安全。

## fields

### 读操作集。比如在选择操作开始时一个`SelectionKey`包含`OP_READ`。如果选择器检测到相对应的通道已经可以读取，已经到达文件末尾，被远程关闭，或者在等待过程中出错，则会添加`OP_READ`到键的就绪操作集并将键添加到它的已选择键集中。

```
public static final int OP_READ = 1 << 0;//1 << 0 == 1
```

### 写操作集

```
public static final int OP_WRITE = 1 << 2;//1 << 2 == 4
```

### 连接操作集

```
public static final int OP_CONNECT = 1 << 3;//8
```

### 接收操作集

```
public static final int OP_ACCEPT = 1 << 4;//16
```

### 附件

```
private volatile Object attachment = null;

private static final AtomicReferenceFieldUpdater<SelectionKey,Object>
    attachmentUpdater = AtomicReferenceFieldUpdater.newUpdater(
        SelectionKey.class, Object.class, "attachment"
    );
```

## methods

### 构造器

```
protected SelectionKey() { }
```

### 返回创建键的通道。本方法在键被取消后也会返回通道。

```
public abstract SelectableChannel channel();
```

### 返回创建的选择器。键被取消后也会返回

```
public abstract Selector selector();
```

### 键是否有效

```
public abstract boolean isValid();
```

### 取消键。键会失效并加入到已取消集中。在下一次选择操作时会从所有集合中移除。

```
public abstract void cancel();
```

### 返回键的关注集。保证会返回在相应的通道上有效的操作位。

```
public abstract int interestOps();
```

### 设置关注集

```
public abstract SelectionKey interestOps(int ops);
```

### 获取键的就绪操作集。

```
public abstract int readyOps();
```

### 测试通道是否已经可以读。

```
public final boolean isReadable() {
    return (readyOps() & OP_READ) != 0;
}
```

### 通道是否已经可以写

```
public final boolean isWritable() {
    return (readyOps() & OP_WRITE) != 0;
}
```

### 通道是否已经完成连接。

```
public final boolean isConnectable() {
    return (readyOps() & OP_CONNECT) != 0;
}
```

### 通道是否可以接收

```
public final boolean isAcceptable() {
    return (readyOps() & OP_ACCEPT) != 0;
}
```

### 对象附带一个`SelectionKey`

```
public final Object attach(Object ob) {
    return attachmentUpdater.getAndSet(this, ob);
}
```

### 附带`SelectionKey`的对象

```
public final Object attachment() {
    return attachment;
}
```