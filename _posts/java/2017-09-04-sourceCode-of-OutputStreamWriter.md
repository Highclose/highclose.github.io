---
layout: post
category: java
title:  "Source code of OutputStreamWriterin java."
description: "Analyze source code of OutputStreamWriter."
tags: [java,source]
---

## 说明

用指定的字符集将字符编码成字节并写入。每一次`write()`调用都会编码指定的字符。输出字节在写入底层的输出流前被保存在一个缓存区中。缓存区的大小可以指定，但是默认值可以应付绝大数情况。传给`write()`方法的字符没有缓存。为了最好的效率，考虑在`OutputStreamwriter`外包裹一层`BufferedWriter`来避免频繁的转换调用。比如`Writer out = new BufferdWriter(new OutputStreamWriter(System.out))`

`surrogate pairs`是用两个字符序列表示一个字符，高`surrogate`从`\uD800`到`\uDBFF`,后跟低`surrogate`范围是从`\uDC00`到`\uDFFF`，这个类会用字符集的默认替换序列来替换畸形的代理对和无法映射的字符序列。

## fields

### 流解码器

```
private final StreamEncoder se;
```

## methods

### 构造器，用指定的字符集

```
public OutputStreamWriter(OutputStream out, String charsetName)
    throws UnsupportedEncodingException
{
	//out作为同步锁
    super(out);
    if (charsetName == null)
        throw new NullPointerException("charsetName");
    se = StreamEncoder.forOutputStreamWriter(out, this, charsetName);
}
```

### 构造器，用默认的字符集

```
public OutputStreamWriter(OutputStream out) {
    super(out);
    try {
        se = StreamEncoder.forOutputStreamWriter(out, this, (String)null);
    } catch (UnsupportedEncodingException e) {
        throw new Error(e);
    }
```

### 用字符集实例构造

```
public OutputStreamWriter(OutputStream out, Charset cs) {
    super(out);
    if (cs == null)
        throw new NullPointerException("charset");
    se = StreamEncoder.forOutputStreamWriter(out, this, cs);
}
```

### 用解码器构造

```
public OutputStreamWriter(OutputStream out, CharsetEncoder enc) {
    super(out);
    if (enc == null)
        throw new NullPointerException("charset encoder");
    se = StreamEncoder.forOutputStreamWriter(out, this, enc);
}
```

### 返回使用的解码器名字

```
public String getEncoding() {
    return se.getEncoding();
}
```

###  刷新流到底层的字节流而不刷新字节流本身。这个方法不是私有的只是为了可以让`PrintStream`调用

```
void flushBuffer() throws IOException {
    se.flushBuffer();
}
```

### 写入一个字符

```
public void write(int c) throws IOException {
    se.write(c);
}
```

### 写入一个字符数组的一部分

```
public void write(char cbuf[], int off, int len) throws IOException {
    se.write(cbuf, off, len);
}
```

### 写入一部分字符串

```
public void write(String str, int off, int len) throws IOException {
    se.write(str, off, len);
}
```

### 刷新流，关闭流

```
public void flush() throws IOException {
    se.flush();
}

public void close() throws IOException {
    se.close();
}
```