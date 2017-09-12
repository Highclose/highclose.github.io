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

`completed`变量表示IO操作是否已经完成。