---
layout: post
category: java
title:  "Source code of InputStreamReader in java."
description: "Analyze source code of InputStreamReader ."
tags: [java,source]
---

## 说明

读取字节并用指定的字符集解码成字符。每一次的read操作都会造成至少一次的底层字节流read操作。为了更有效的进行解码，需要预读比本次读取操作字符数更多的字符数。为了效率最大化，考虑将`InputStreamReader`包裹在`BufferedReader`中，比如：`BufferedReader in = new BufferedReader(new InputStreamReader(System.in))`