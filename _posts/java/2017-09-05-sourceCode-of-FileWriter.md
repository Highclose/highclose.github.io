---
layout: post
category: java
title:  "Source code of FileWriter in java."
description: "Analyze source code of FileWriter ."
tags: [java,source]
---

## 说明

写入字符文件。文件是是否有效或是否可创建取决于底层平台。某些平台，一个文件只允许同时一个`FileWriter`写入，在这种情况下，已打开的文件不能用来创建实例。

## implements

```
OutputStreamWriter
```

## methods

### 文件名

```
public FileWriter(String fileName) throws IOException {
    super(new FileOutputStream(fileName));
}
```

### 是否追加

```
public FileWriter(String fileName, boolean append) throws IOException {
    super(new FileOutputStream(fileName, append));
}
```

### 文件实例

```
public FileWriter(File file) throws IOException {
    super(new FileOutputStream(file));
}
```

### 是否追加

```
public FileWriter(File file, boolean append) throws IOException {
    super(new FileOutputStream(file, append));
}
```

### 文件描述符

```
public FileWriter(FileDescriptor fd) {
    super(new FileOutputStream(fd));
}
```

## 备注

`new FileWriter(object)`相当于`new OutputStreamReader(new FileOutputStream(object))`