---
layout: post
category: java
title:  "Source code of InputStreamReader in java."
description: "Analyze source code of InputStreamReader ."
tags: [java,source]
---

## 说明

读取字节并用指定的字符集解码成字符。每一次的read操作都会造成至少一次的底层字节流read操作。为了更有效的进行解码，需要预读比本次读取操作字符数更多的字符数。为了效率最大化，考虑将`InputStreamReader`包裹在`BufferedReader`中，比如：`BufferedReader in = new BufferedReader(new InputStreamReader(System.in))`

## implements

```
Reader
```

## fields

### 流解码器

```
private final StreamDecoder sd;
```

### 构造器，用默认的字符集构造实例。

```
public InputStreamReader(InputStream in) {
	//同步锁为in
    super(in);
    try {
    	//默认解码器
        sd = StreamDecoder.forInputStreamReader(in, this, (String)null); 
    } catch (UnsupportedEncodingException e) {
        throw new Error(e);
    }
}
```

###  指定字符集名称

```
public InputStreamReader(InputStream in, String charsetName)
    throws UnsupportedEncodingException
{
    super(in);
    if (charsetName == null)
        throw new NullPointerException("charsetName");
    sd = StreamDecoder.forInputStreamReader(in, this, charsetName);
}
```

### 指定字符集实例

```
public InputStreamReader(InputStream in, Charset cs) {
    super(in);
    if (cs == null)
        throw new NullPointerException("charset");
    sd = StreamDecoder.forInputStreamReader(in, this, cs);
}
```

### 指定解码器

```
public InputStreamReader(InputStream in, CharsetDecoder dec) {
    super(in);
    if (dec == null)
        throw new NullPointerException("charset decoder");
    sd = StreamDecoder.forInputStreamReader(in, this, dec);
}
```

### 返回当前流用作解码的字符集名称，有历史名则返回历史名，否则返回规范名，用`InputStreamReader(InputStream, String)`创建的实例，返回的名字可能与传参不同。

```
public String getEncoding() {
    return sd.getEncoding();
}
```

### 读取一个字符

```
public int read() throws IOException {
    return sd.read();
}
```

### 将字符读取至数组中，返回值为实际读取的字节

```
public int read(char cbuf[], int offset, int length) throws IOException {
    return sd.read(cbuf, offset, length);
}
```

###  流是否已可读取。当输入缓存区不为空时或底层字节流有字节可以读取里，为`true`

```
public boolean ready() throws IOException {
    return sd.ready();
}
```

### 关闭流

```
public void close() throws IOException {
    sd.close();
}
```