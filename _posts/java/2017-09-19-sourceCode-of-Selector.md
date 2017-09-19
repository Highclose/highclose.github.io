---
layout: post
category: java
title:  "Source code of Selector in java."
comments: true
description: "Analyze source code of Selector ."
tags: [java,source]
---

## 说明

`SelectableChannel`对象的多路复用器。一个选择器可以通过调用`open`方法（会使用系统默认的`SelectorProvider`）创建。也可以通过`SelectorProvider.openSelector`来创建自定义的选择器。直到关闭前，选择器都会是打开状态。

一个可选择通道向选择器的注册由一个`SelectionKey`对象来表示。一个选择器包含了三个选择键集合：

* 包含了代表当前向选择器注册的键集合。该集合通过调用`keys`方法获取。
* 已选择的键集合，是一个包含键对应的通道已经可以执行键关注的动作的键集合。集合通过`selectedKeys()`返回，该集合是前一个集合的子集。
* 已取消的键集合，是一个包含键已经取消但是通道还未注销的键集合。该集合无法直接获取，也是第一个集合的子集

在一个新创建的选择器里，这三个集合都是空的。

将键加入到选择器的键集合是用方法`SelectableChannel.register(Selector,int)`注册通道的副作用，在选择动作执行中，键被移除集合。集合本身不直接进行修改。

当键被取消，无论是关闭通道还是调用`SelectionKey.cancel`方法，都会加入到已取消的键集合。取消一个键会导致在下一次选择动作执行中注销通道，同时，键也会被移除所有的集合。

键选择操作时，键被加入到已选择键集合。一个键可以通过调用`Set.remove(Object)`或`Iterator.remove()`（从集合中获取）直接从已选择键集合中移除。键不应该直接加入到已选择键集合中。

选择通过调用`select()`  `select(long)` `selectNow()`方法来执行并包括以下三个步骤：

* 已取消的键集合中的每个键会被从每个键集合中移除，并且对应的通道会被注销。该步骤会清空已取消键集合。

* 底层操作系统会执行每个剩余通道的更新来执行键关注的动作。当一个通道已经就绪至少一个操作，以下两个动作会执行

  * 如果一个通道的键还没有在已选择键集合中，则会被加入集合且他的就绪操作集合会修改成通道报告的一样。其他以前在就绪集合的信息都会被丢弃。
  * 如果键已经在已选择键集合中，则他的就绪操作集合会跟通道报告的一样，先前记录的信息会被保留，换句话说，底层系统返回的就绪集合按位分离进键当前的就绪集合中。

  如果在本步骤前键集合中的键关注的集合时空的，则不会有键的就绪操作集合被更新。

* 如果在步骤2时有任意一个键被加入到已取消键集合，则会想步骤1一样执行

无论通道是否因为等待至少一个通道就绪而阻塞，等待多久是这三个方法唯一的区别

选择器是线程安全的，但是他们的集合不是。选择动作按选择器本身，键集合，已选择键集合的顺序做同步，步骤1，3时也会用已取消键集合做同步。在选择操作过程中对一个键的关注集合做出的变动在当此选择操作中不会起作用，下次时方可见。键可以在任意时间取消，通道也可以在任意时间关闭，所以集合中的键一定时有效的或代表的通道不一定时打开的。一个因为`Select`方法阻塞的通道可以由以下三个方式被别的线程中断：

* 调用选择器的`wakeup`方法
* 调用选择器的`close`方法
* 调用阻塞线程的`interrupt`方法，在这种情况下，该线程的终端状态会被设置，选择器的`wakeup`会被执行

本类是一个抽象类

## implements

```
implements Closeable
```

## methods

### 构造器

```
protected Selector() { }
```

### 打开一个选择器

```
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}
```

### 是否打开

```
public abstract boolean isOpen();
```

## 返回创建通道的提供者

```
public abstract SelectorProvider provider();
```

### 键集合。该集合不能被直接修改。

```
public abstract Set<SelectionKey> keys();
```

### 已选择键集合。该集合可以被移除但不能被直接添加

```
public abstract Set<SelectionKey> selectedKeys();
```

### 选择键对应的已经可以进行IO操作的通道。本操作执行一次非阻塞的选择操作。如果前一次选择操作后没有通道可选择则直接返回0。调用本方法会清除前一次`wakeup`方法引起的效果。

```
public abstract int selectNow() throws IOException;
```

### 选择。是阻塞的选择操作。只有在至少有一个通道已经选择或者选择器的`wakeup`方法执行了或当前线程被中断或超时才会返回。

```
public abstract int select(long timeout)
    throws IOException;
```

###  阻塞的选择。

```
public abstract int select() throws IOException;
```

### 唤醒。使还未返回的第一个选择操作立即返回。如果有另外一个线程因为`select` `select(long)`方法阻塞，则该线程会立即返回。如果当前没有选择操作进行中，除非`selectNow`同时调用，否则下一次调用`select` `select(long)`会立即返回。

```
public abstract Selector wakeup();
```

### 关闭选择器。如果线程当前正因为选择动作而阻塞则会被中断。任何未取消的键都失效，他们的通道被关闭，选择器其他资源也都释放。

```
public abstract void close() throws IOException;
```