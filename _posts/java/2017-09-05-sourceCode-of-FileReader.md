---
layout: post
category: java
title:  "Source code of FileReader in java."
description: "Analyze source code of FileReader ."
tags: [java,source]
---

## 说明

读取字符文件。构造器假定默认字符编码和默认字节缓存大小合适。如果要自定义，使用`InputStreamReader`或`FileInputStream`

## implements

```
InputStreamReader
```

## methods

### 创建一个实例，给定文件名

```
public FileReader(String fileName) throws FileNotFoundException {
    super(new FileInputStream(fileName));
}
```

###  创建一个实例，给定文件实例

```
public FileReader(File file) throws FileNotFoundException {
    super(new FileInputStream(file));
}
```

###   创建一个实例，给定文件描述符实例

```
public FileReader(FileDescriptor fd) {
    super(new FileInputStream(fd));
}
```

## 附加说明

其实`new FileReader(object)`相当于`new InputStreamReader(new FileInputStream(object))`