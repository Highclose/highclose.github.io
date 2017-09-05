---
layout: post
category: java
title:  "Source code of PrintStream in java."
description: "Analyze source code of PrintStream ."
tags: [java,source]
---

## 说明

`System.out`和`System.err`就是这个类的实例。`PrintStream`给别的输出流添加了附加功能，字面上看就是可以便利的输出各种数据。不像其他的输出流，`PrintStream`不会抛出`IOException`,异常只是设置了一个标志位，可以通过`checkError`方法来获取。作为一个可选项，`PrintStream`可以自动调动`flush`方法。意味着，在字节数组被写入，`println`方法被调用后，或者写入了一个换行符或`\n`字节后，自动调用`flush`

`PrintStream`打印的所有字符都用平台默认的字符编码转换成字节。如果需要打印字符而不是字节，考虑使用`PrintWriter`

## implements

```
extends FilterOutputStream implements Appendable, Closeable
```

## fields

### 自动刷新标志，错误标志，格式化实例

```
private final boolean autoFlush;
private boolean trouble = false;
private Formatter formatter;
```

### 监测字符流和文本流，这样可以单独刷新这两个流而不是刷新整个流

```
private BufferedWriter textOut;
private OutputStreamWriter charOut;
```

###  关闭标志，避免递归关闭

```
private boolean closing = false;
```

## methods

### 判断对象是否为null,不为null则返回对象本身。在这里显示定义这个方法，则不需要再创建一个对象来调用`java.util.Objects.requireNonNull`

```
private static <T> T requireNonNull(T obj, String message) {
    if (obj == null)
        throw new NullPointerException(message);
    return obj;
}
```

###  返回给定字符集名字的字符集实例

```
private static Charset toCharset(String csn)
    throws UnsupportedEncodingException
{
    requireNonNull(csn, "charsetName");
    try {
        return Charset.forName(csn);
    } catch (IllegalCharsetNameException|UnsupportedCharsetException unused) {
        throw new UnsupportedEncodingException(csn);
    }
}
```

### 私有构造函数

```
private PrintStream(boolean autoFlush, OutputStream out) {
    super(out);
    this.autoFlush = autoFlush;
    this.charOut = new OutputStreamWriter(this);
    this.textOut = new BufferedWriter(charOut);
}

private PrintStream(boolean autoFlush, OutputStream out, Charset charset) {
  super(out);
  this.autoFlush = autoFlush;
  this.charOut = new OutputStreamWriter(this, charset);
  this.textOut = new BufferedWriter(charOut);
}
```

### 私有构造函数的变体，在检验`OutputStream`参数前就可以验证字符集名

```
private PrintStream(boolean autoFlush, Charset charset, OutputStream out)
    throws UnsupportedEncodingException
{
    this(autoFlush, out, charset);
}
```

### 公有构造函数 默认不是自动刷新

```
public PrintStream(OutputStream out) {
    this(out, false);
}
```

### 自动刷新 

```
public PrintStream(OutputStream out, boolean autoFlush) {
    this(autoFlush, requireNonNull(out, "Null output stream"));
}
```

### 指定字符集

```
public PrintStream(OutputStream out, boolean autoFlush, String encoding)
    throws UnsupportedEncodingException
{
    this(autoFlush,
         requireNonNull(out, "Null output stream"),
         toCharset(encoding));
}
```

### 通过文件路径构造。

```
public PrintStream(String fileName) throws FileNotFoundException {
    this(false, new FileOutputStream(fileName));
}
```

### 通过文件名与字符集

```
public PrintStream(String fileName, String csn)
    throws FileNotFoundException, UnsupportedEncodingException
{
    //确保字符集在文件打开前验证
    this(false, toCharset(csn), new FileOutputStream(fileName));
}
```

### 通过文件实例

```
public PrintStream(File file) throws FileNotFoundException {
    this(false, new FileOutputStream(file));
}
public PrintStream(File file, String csn)
throws FileNotFoundException, UnsupportedEncodingException
{
  this(false, toCharset(csn), new FileOutputStream(file));
}
```

### 检查流还未被关闭

```
private void ensureOpen() throws IOException {
    if (out == null)
        throw new IOException("Stream closed");
}
```

### 刷新流，

```
public void flush() {
    synchronized (this) {
        try {
            ensureOpen();
            out.flush();
        }
        catch (IOException x) {
            trouble = true;
        }
    }
}
```

### 关闭流

```
public void close() {
    synchronized (this) {
        if (! closing) {
            closing = true;
            try {
                textOut.close();
                out.close();
            }
            catch (IOException x) {
                trouble = true;
            }
            textOut = null;
            charOut = null;
            out = null;
        }
    }
}
```

### 刷新流并且检测错误状态。当底层流抛出`IOException`时，调用`setError`方法，错误标志被设置为`true`，当底层流抛出`InterruptedIOException`时，则会执行`Thread.currentThread().interrupt()`

```
public boolean checkError() {
    if (out != null)
        flush();
    if (out instanceof java.io.PrintStream) {
        PrintStream ps = (PrintStream) out;
        return ps.checkError();
    }
    return trouble;
}
```

### 设置错误标志位

```
protected void setError() {
    trouble = true;
}
```

### 清除错误标志位

```
protected void clearError() {
    trouble = false;
}
```

### 写入一个字节，如果该字节是一个换行符且自动刷新开启则`flush`方法会被调用。写入字符使用`print(char)`和`println(char)`

```
public void write(int b) {
    try {
        synchronized (this) {
            ensureOpen();
            out.write(b);
            if ((b == '\n') && autoFlush)
                out.flush();
        }
    }
    catch (InterruptedIOException x) {
        Thread.currentThread().interrupt();
    }
    catch (IOException x) {
        trouble = true;
    }
}
```

### 写入数组

```
public void write(byte buf[], int off, int len) {
    try {
        synchronized (this) {
            ensureOpen();
            out.write(buf, off, len);
            if (autoFlush)
                out.flush();
        }
    }
    catch (InterruptedIOException x) {
        Thread.currentThread().interrupt();
    }
    catch (IOException x) {
        trouble = true;
    }
}
```

### 私有方法，文本流与字符流会一直刷新 ，所以写入动作会非常敏捷。

```
private void write(char buf[]) {
    try {
        synchronized (this) {
            ensureOpen();
            textOut.write(buf);
            textOut.flushBuffer();
            charOut.flushBuffer();
            if (autoFlush) {
                for (int i = 0; i < buf.length; i++)
                    if (buf[i] == '\n')
                        out.flush();
            }
        }
    }
    catch (InterruptedIOException x) {
        Thread.currentThread().interrupt();
    }
    catch (IOException x) {
        trouble = true;
    }
}

private void write(String s) {
    try {
        synchronized (this) {
            ensureOpen();
            textOut.write(s);
            textOut.flushBuffer();
            charOut.flushBuffer();
            if (autoFlush && (s.indexOf('\n') >= 0))
                out.flush();
        }
    }
    catch (InterruptedIOException x) {
        Thread.currentThread().interrupt();
    }
    catch (IOException x) {
        trouble = true;
    }
}

private void newLine() {
    try {
        synchronized (this) {
            ensureOpen();
            textOut.newLine();
            textOut.flushBuffer();
            charOut.flushBuffer();
            if (autoFlush)
                out.flush();
        }
    }
    catch (InterruptedIOException x) {
        Thread.currentThread().interrupt();
    }
    catch (IOException x) {
        trouble = true;
    }
}
```

###  写入各种实例

    public void print(boolean b) {
        write(b ? "true" : "false");
    }
    
    public void print(char c) {
    	write(String.valueOf(c));
    }
    
    public void print(int i) {
    	write(String.valueOf(i));
    }
    
    public void print(long l) {
    	write(String.valueOf(l));
    }
    
    public void print(float f) {
    	write(String.valueOf(f));
    }
    
    public void print(double d) {
    	write(String.valueOf(d));
    }
    
    public void print(char s[]) {
    	write(s);
    }
    
    public void print(String s) {
      if (s == null) {
        s = "null";
        }
      write(s);
    }
    
    public void print(Object obj) {
    	write(String.valueOf(obj));
    }
    
    
### 换行输出

```
public void println() {
    newLine();
}
```

### 换行输出实例，其他实例类似

```
public void println(Object x) {
    String s = String.valueOf(x);
    synchronized (this) {
        print(s);
        newLine();
    }
}
```

