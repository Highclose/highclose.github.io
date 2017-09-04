---
layout: post
category: java
title:  "Source code of OutputStreamWriterin java."
description: "Analyze source code of OutputStreamWriter."
tags: [java,source]
---

## 说明

用指定的字符集将字符编码成字节并写入。每一次`write()`调用都会编码指定的字符。输出字节在写入底层的输出流前被保存在一个缓存区中。缓存区的大小可以指定，但是默认值可以应付绝大数情况。传给`write()`方法的字符没有缓存。为了最好的效率，考虑在`OutputStreamwriter`外包裹一层`BufferedWriter`来避免频繁的转换调用。比如`Writer out = new BufferdWriter(new OutputStreamWriter(System.out))`