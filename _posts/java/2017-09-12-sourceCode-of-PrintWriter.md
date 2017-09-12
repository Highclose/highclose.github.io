---
layout: post
category: java
title:  "Source code of PrintWriter in java."
comments: true
description: "Analyze source code of PrintWriter ."
tags: [java,source]
---

## 说明

格式化输出各种对象。本来集成了`PrintStream`类中所有的`print`方法。本类不包含写入字节功能。不像`PrintStream`，本类在自动刷新开户时，只有`println` `printf` `format`方法调用时才会刷新，而在`PrintStream`中，当换行字符出现时就是刷新。

## implements

```
 extends Writer 
```

## fields

### 底层流

```
protected Writer out;
```

### 自动刷新

```
private final boolean autoFlush;
```

### 是否有报错

```
private boolean trouble = false;
```

### 格式化实例

```
private Formatter formatter;
```

### 包裹的`PrintStream`实例

```
private PrintStream psOut = null;
```

### 换行符

```
private final String lineSeparator;
```

## methods

### 构造器（`Writer`）

```
public PrintWriter (Writer out) {
	this(out, false);
}
public PrintWriter(Writer out,
                   boolean autoFlush) {
    super(out);
    this.out = out;
    this.autoFlush = autoFlush;
    lineSeparator = java.security.AccessController.doPrivileged(
        new sun.security.action.GetPropertyAction("line.separator"));
}
```

### 构造器（`OutputStream`）

```
public PrintWriter(OutputStream out) {
	this(out, false);
}
public PrintWriter(OutputStream out, boolean autoFlush) {
    this(new BufferedWriter(new OutputStreamWriter(out)), autoFlush);

    // 用作错误传播
    if (out instanceof java.io.PrintStream) {
        psOut = (PrintStream) out;
    }
}
```

### 构造器（文件）

```
public PrintWriter(String fileName) throws FileNotFoundException {
    this(new BufferedWriter(new OutputStreamWriter(new FileOutputStream(fileName))),
         false);
}

private PrintWriter(Charset charset, File file)
    throws FileNotFoundException
{
    this(new BufferedWriter(new OutputStreamWriter(new FileOutputStream(file), charset)),
         false);
}

public PrintWriter(String fileName, String csn)
	throws FileNotFoundException, UnsupportedEncodingException
{
	this(toCharset(csn), new File(fileName));
}

public PrintWriter(File file) throws FileNotFoundException {
  this(new BufferedWriter(new OutputStreamWriter(new FileOutputStream(file))),
  	false);
}

public PrintWriter(File file, String csn)
	throws FileNotFoundException, UnsupportedEncodingException
{
	this(toCharset(csn), file);
}
```

### 返回字符集对象

```
private static Charset toCharset(String csn)
    throws UnsupportedEncodingException
{
    Objects.requireNonNull(csn, "charsetName");
    try {
        return Charset.forName(csn);
    } catch (IllegalCharsetNameException|UnsupportedCharsetException unused) {
        throw new UnsupportedEncodingException(csn);
    }
}
```

### 检查流是否被关闭

```
private void ensureOpen() throws IOException {
    if (out == null)
        throw new IOException("Stream closed");
}
```

### 刷新

```
public void flush() {
    try {
        synchronized (lock) {
            ensureOpen();
            out.flush();
        }
    }
    catch (IOException x) {
        trouble = true;
    }
}
```

### 关闭流

```
public void close() {
    try {
        synchronized (lock) {
            if (out == null)
                return;
            out.close();
            out = null;
        }
    }
    catch (IOException x) {
        trouble = true;
    }
}
```

### 刷新流并检查流错误

```
public boolean checkError() {
    if (out != null) {
        flush();
    }
    if (out instanceof java.io.PrintWriter) {
        PrintWriter pw = (PrintWriter) out;
        return pw.checkError();
    } else if (psOut != null) {
        return psOut.checkError();
    }
    return trouble;
}
```

### 设置错误标识

```
protected void setError() {
    trouble = true;
}
```

### 清除错误标识

```
protected void clearError() {
    trouble = false;
}
```

### 写入一个字符

```
public void write(int c) {
    try {
        synchronized (lock) {
            ensureOpen();
            out.write(c);
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

### 写入字符数组

```
public void write(char buf[], int off, int len) {
    try {
        synchronized (lock) {
            ensureOpen();
            out.write(buf, off, len);
        }
    }
    catch (InterruptedIOException x) {
        Thread.currentThread().interrupt();
    }
    catch (IOException x) {
        trouble = true;
    }
}

public void write(char buf[]) {
	write(buf, 0, buf.length);
}
```

### 写入字符串

```
public void write(String s, int off, int len) {
    try {
        synchronized (lock) {
            ensureOpen();
            out.write(s, off, len);
        }
    }
    catch (InterruptedIOException x) {
        Thread.currentThread().interrupt();
    }
    catch (IOException x) {
        trouble = true;
    }
}

public void write(String s) {
	write(s, 0, s.length());
}
```

### 换行

```
private void newLine() {
    try {
        synchronized (lock) {
            ensureOpen();
            out.write(lineSeparator);
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

`print`方法（还有其他各种类型的）。

```
public void print(Object obj) {
    write(String.valueOf(obj));
}
```

`println`方法（还有其他各种类型）

```
public void println() {
    newLine();
}

public void println(Object x) {
  String s = String.valueOf(x);
  synchronized (lock) {
    print(s);
    println();
  }
}
```

### 格式化输出

```
public PrintWriter printf(String format, Object ... args) {
    return format(format, args);
}

public PrintWriter printf(Locale l, String format, Object ... args) {
	return format(l, format, args);
}
```

### 格式化输出具体方法

```
public PrintWriter format(String format, Object ... args) {
    try {
        synchronized (lock) {
            ensureOpen();
            if ((formatter == null)
                || (formatter.locale() != Locale.getDefault()))
                formatter = new Formatter(this);
            formatter.format(Locale.getDefault(), format, args);
            if (autoFlush)
                out.flush();
        }
    } catch (InterruptedIOException x) {
        Thread.currentThread().interrupt();
    } catch (IOException x) {
        trouble = true;
    }
    return this;
}

public PrintWriter format(Locale l, String format, Object ... args) {
  try {
    synchronized (lock) {
      ensureOpen();
      if ((formatter == null) || (formatter.locale() != l))
      formatter = new Formatter(this, l);
      formatter.format(l, format, args);
      if (autoFlush)
      out.flush();
  	}
  } catch (InterruptedIOException x) {
  	Thread.currentThread().interrupt();
  } catch (IOException x) {
  	trouble = true;
  }
  return this;
}
```

### 添加

```
public PrintWriter append(CharSequence csq) {
    if (csq == null)
        write("null");
    else
        write(csq.toString());
    return this;
}

public PrintWriter append(CharSequence csq, int start, int end) {
  CharSequence cs = (csq == null ? "null" : csq);
  write(cs.subSequence(start, end).toString());
  return this;
}

public PrintWriter append(char c) {
  write(c);
  return this;
}
```