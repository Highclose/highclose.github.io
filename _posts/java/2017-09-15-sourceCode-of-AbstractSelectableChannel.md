---
layout: post
category: java
title:  "Source code of AbstractSelectableChannel in java."
comments: true
description: "Analyze source code of AbstractSelectableChannel."
tags: [java,source]
---

## 说明

这是可选择通道的基类。本类定义了处理通道注册，撤销，关闭的方法。它保持着当前通道的阻塞模式和当前的`SelectionKey`集合。本类在实现父类（`SelectableChannel`）的方法时按需都提供了同步功能。本类提供的抽象方法都不需要提供同步功能。

## implements

```
extends SelectableChannel
```

## fields

### 创建本通道的提供者

```
private final SelectorProvider provider;
```

### 注册通道时创建的keys，因为通道关闭的时候需要撤销key所以要保存着。被`keyLock`保护

```
private SelectionKey[] keys = null;
private int keyCount = 0;
```

### 给集合和计数加锁

```
private final Object keyLock = new Object();
```

### 给注册和阻塞配置加锁

```
private final Object regLock = new Object();
```

### 阻塞配置

```
boolean blocking = true;
```

## methods

### 构造器

```
protected AbstractSelectableChannel(SelectorProvider provider) {
    this.provider = provider;
}
```

### 返回创建本通道的提供者

```
public final SelectorProvider provider() {
    return provider;
}
```

### 加入key

```
private void addKey(SelectionKey k) {
    assert Thread.holdsLock(keyLock);
    int i = 0;
    if ((keys != null) && (keyCount < keys.length)) {
        //查找key数组中的空元素
        for (i = 0; i < keys.length; i++)
            if (keys[i] == null)
                break;
    } else if (keys == null) {//如果集合为空
        keys =  new SelectionKey[3];
    } else {
        //扩展数组
        int n = keys.length * 2;
        SelectionKey[] ks =  new SelectionKey[n];
        for (i = 0; i < keys.length; i++)
            ks[i] = keys[i];
        keys = ks;
        i = keyCount;
    }
    keys[i] = k;
    keyCount++;
}
```

### 查找key

```
private SelectionKey findKey(Selector sel) {
    synchronized (keyLock) {
        if (keys == null)
            return null;
        for (int i = 0; i < keys.length; i++)
            if ((keys[i] != null) && (keys[i].selector() == sel))
                return keys[i];
        return null;
    }
}
```

### 移除key

```
void removeKey(SelectionKey k) {                  
    synchronized (keyLock) {
        for (int i = 0; i < keys.length; i++)
            if (keys[i] == k) {
                keys[i] = null;
                keyCount--;
            }
        ((AbstractSelectionKey)k).invalidate();
    }
}
```

### 是否有有效的key

```
private boolean haveValidKeys() {
    synchronized (keyLock) {
        if (keyCount == 0)
            return false;
        for (int i = 0; i < keys.length; i++) {
            if ((keys[i] != null) && keys[i].isValid())
                return true;
        }
        return false;
    }
}
```

### 是否有已经注册的key

```
public final boolean isRegistered() {
    synchronized (keyLock) {
        return keyCount != 0;
    }
}
```

### 查找key

```
public final SelectionKey keyFor(Selector sel) {
    return findKey(sel);
}
```

### 用给定的`Selector`注册通道，返回一个`SelectionKey`。本方法首次验证通道还是打开状态，而且给定的初始`interest`集合也是有效的。如果本通道已经用给定的`Selector`注册，则会将通道的`interest`集合设置成传入的参数并返回原先的`SelectionKey`

```
public final SelectionKey register(Selector sel, int ops,
                                   Object att)
    throws ClosedChannelException
{
    synchronized (regLock) {
        if (!isOpen())
            throw new ClosedChannelException();
        if ((ops & ~validOps()) != 0)
            throw new IllegalArgumentException();
        if (blocking)//只适用于非阻塞通道
            throw new IllegalBlockingModeException();
        SelectionKey k = findKey(sel);
        if (k != null) {
            k.interestOps(ops);
            k.attach(att);
        }
        if (k == null) {
            //新注册
            synchronized (keyLock) {
                if (!isOpen())
                    throw new ClosedChannelException();
                k = ((AbstractSelector)sel).register(this, ops, att);
                addKey(k);
            }
        }
        return k;
    }
}
```

### 关闭通道，实际的关闭方法。取消了所有通道的key

```
protected final void implCloseChannel() throws IOException {
    implCloseSelectableChannel();
    synchronized (keyLock) {
        int count = (keys == null) ? 0 : keys.length;
        for (int i = 0; i < count; i++) {
            SelectionKey k = keys[i];
            if (k != null)
                k.cancel();
        }
    }
}
```

### 关闭本`SelectableChannel`，该方法只有在通道还未关闭才会调用且调用不超过一次。本方法的实现必须安排使因为IO操作阻塞在通道上的别的线程立即返回。

```
protected abstract void implCloseSelectableChannel() throws IOException;
```

### 是否阻塞

```
public final boolean isBlocking() {
    synchronized (regLock) {
        return blocking;
    }
}

public final Object blockingLock() {
    return regLock;
}

public final SelectableChannel configureBlocking(boolean block)
	throws IOException
{
    synchronized (regLock) {
        if (!isOpen())
        	throw new ClosedChannelException();
        if (blocking == block)
        	return this;
        if (block && haveValidKeys())
        	throw new IllegalBlockingModeException();
        implConfigureBlocking(block);
        blocking = block;
    }
    return this;
}

//模式修改，只有在新模式与当前模式不同时才会调用。
protected abstract void implConfigureBlocking(boolean block)
	throws IOException;
```

