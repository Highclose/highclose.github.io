---
layout: post
category: java
title:  "Source code of AbstractInterruptibleChannel in java."
description: "Analyze source code of AbstractInterruptibleChannel ."
tags: [java,source]
---

## 说明

可中断通道的基类。这个类封装了异步关闭和中断通道的底层操作。一个具体的通道必须在阻塞的IO操作前后分别调用`begin` `end`方法。为了保证`end`方法一定会被执行，这些方法应该被用在try代码块里，比如：

```

boolean completed = false;
try {
     begin();
     completed = ...;    // 阻塞IO操作
     return ...;         // 返回值
 } finally {
     end(completed);
 }
```

`completed`变量表示IO操作是否已经完成。比如在读取字节的操作中，只有在字节实际已经传到目标缓存时才会为`true`。一个具体的通道还必须实现`implCloseChannel`方法，当别的线程对本通道执行一个本地IO操作而阻塞时会立即返回。如果一个线程被中断或者通道已经被异步关闭，通道的`end`方法会抛出一个适当的异常。本类要实现线程同步需要实现`java.nio.channels.Channel`接口。实现`implCloseChannel`并不能保证通道会被异步关闭。

## implements

```
implements Channel, InterruptibleChannel
```

## fields

### 同步锁

```
private final Object closeLock = new Object();
```

### 通道开启状态

```
private volatile boolean open = true;
```

### 中断操作

```
private Interruptible interruptor;//中断处理器
private volatile Thread interrupted;//中断的线程
```

## methods

### 构造器

```
protected AbstractInterruptibleChannel() { }
```

### 关闭通道

```
public final void close() throws IOException {
    synchronized (closeLock) {
        if (!open)
            return;
        open = false;
        implCloseChannel();
    }
}
```

### 该方法由`close`方法调用为了实际的关闭通道动作。这个方法不会被调用超过一次。本方法的实现必须可以使在本通道阻塞的线程可以立即返回。

```
protected abstract void implCloseChannel() throws IOException;
```

### 获取通道开启状态

```
public final boolean isOpen() {
    return open;
}
```

### 标志可能会阻塞的IO操作起点

```
protected final void begin() {
    if (interruptor == null) {
        interruptor = new Interruptible() {//定义中断器
                public void interrupt(Thread target) {
                    synchronized (closeLock) {
                        if (!open)
                            return;//通道已经关闭则直接返回
                        open = false;//设置状态关闭
                        interrupted = target;中断线程
                        try {
                            AbstractInterruptibleChannel.this.implCloseChannel();//关闭
                        } catch (IOException x) { }
                    }
                }};
    }
    blockedOn(interruptor);
    Thread me = Thread.currentThread();
    if (me.isInterrupted())
        interruptor.interrupt(me);//如果被中断则执行中断操作
}
```

### 标志阻塞的IO操作终点。只有当IO操作成功完成后才会返回`true`

```
protected final void end(boolean completed)
    throws AsynchronousCloseException
{
    blockedOn(null);
    Thread interrupted = this.interrupted;
    if (interrupted != null && interrupted == Thread.currentThread()) {
        interrupted = null;
        throw new ClosedByInterruptException();//线程阻塞被中断
    }
    if (!completed && !open)//IO没有成功完成且通道被异步关闭
        throw new AsynchronousCloseException();
}
```

### 阻塞

```
static void blockedOn(Interruptible intr) {         // package-private
        sun.misc.SharedSecrets.getJavaLangAccess().blockedOn(Thread.currentThread(),
                                                             intr);
    }
```