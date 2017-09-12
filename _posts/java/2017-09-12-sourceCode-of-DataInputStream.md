---
layout: post
category: java
title:  "Source code of DataInputStream in java."
description: "Analyze source code of DataInputStream ."
tags: [java,source]
---

## 说明

数据输入流可以让应用从底层流中读取java的基本数据类型。本类不是线程安全的。

## implements

```
 extends FilterInputStream implements DataInput 
```

## fields

### 工作数组，在`readUTF`方法中会初始化

```
private byte bytearr[] = new byte[80];
private char chararr[] = new char[80];
```

### 缓存数组

```
private byte readBuffer[] = new byte[8];
private char lineBuffer[];
```

## methods

### 构造器，传入一个`InputStream`实例

```
public DataInputStream(InputStream in) {
    super(in);//this.in = in;
}
```

### 从底层流中读入字节。

```
public final int read(byte b[]) throws IOException {
    return in.read(b, 0, b.length);
}
public final int read(byte b[], int off, int len) throws IOException {
	return in.read(b, off, len);
}

```

### 从底层流中读满字节

```
public final void readFully(byte b[]) throws IOException {
    readFully(b, 0, b.length);
}
public final void readFully(byte b[], int off, int len) throws IOException {
  if (len < 0)
  	throw new IndexOutOfBoundsException();
  int n = 0;
  while (n < len) {
    int count = in.read(b, off + n, len - n);
    if (count < 0)
    throw new EOFException();//如果数组没有读满，则抛出异常
    n += count;
  }
}
```

### 跳过字节

```
public final int skipBytes(int n) throws IOException {
    int total = 0;
    int cur = 0;

    while ((total<n) && ((cur = (int) in.skip(n-total)) > 0)) {
        total += cur;
    }

    return total;
}
```

### 读取`Boolean`值

```
public final boolean readBoolean() throws IOException {
    int ch = in.read();
    if (ch < 0)
        throw new EOFException();
    return (ch != 0);
}
```

### 读取一个字节

```
public final byte readByte() throws IOException {
    int ch = in.read();
    if (ch < 0)
        throw new EOFException();
    return (byte)(ch);
}
```

### 读取一个无符号字节

```
public final int readUnsignedByte() throws IOException {
    int ch = in.read();
    if (ch < 0)
        throw new EOFException();
    return ch;
}
```

### 读取一个`short`类型

```
public final short readShort() throws IOException {
    int ch1 = in.read();
    int ch2 = in.read();
    if ((ch1 | ch2) < 0)
        throw new EOFException();
    return (short)((ch1 << 8) + (ch2 << 0));
}
```

### 读取一个无符号`short`类型

```
public final int readUnsignedShort() throws IOException {
    int ch1 = in.read();
    int ch2 = in.read();
    if ((ch1 | ch2) < 0)
        throw new EOFException();
    return (ch1 << 8) + (ch2 << 0);
}
```

### 读取一个字符

```
public final char readChar() throws IOException {
    int ch1 = in.read();
    int ch2 = in.read();
    if ((ch1 | ch2) < 0)
        throw new EOFException();
    return (char)((ch1 << 8) + (ch2 << 0));
}
```

### 读取一个`int`类型

```
public final int readInt() throws IOException {
    int ch1 = in.read();
    int ch2 = in.read();
    int ch3 = in.read();
    int ch4 = in.read();
    if ((ch1 | ch2 | ch3 | ch4) < 0)
        throw new EOFException();
    return ((ch1 << 24) + (ch2 << 16) + (ch3 << 8) + (ch4 << 0));
}
```

### 读取一个`long`类型

```
public final long readLong() throws IOException {
    readFully(readBuffer, 0, 8);
    return (((long)readBuffer[0] << 56) +
            ((long)(readBuffer[1] & 255) << 48) +
            ((long)(readBuffer[2] & 255) << 40) +
            ((long)(readBuffer[3] & 255) << 32) +
            ((long)(readBuffer[4] & 255) << 24) +
            ((readBuffer[5] & 255) << 16) +
            ((readBuffer[6] & 255) <<  8) +
            ((readBuffer[7] & 255) <<  0));
}
```

### 读取一个`float`类型

```
public final float readFloat() throws IOException {
    return Float.intBitsToFloat(readInt());//一个本地方法
}
```

### 读取一个`double`类型

```
public final double readDouble() throws IOException {
    return Double.longBitsToDouble(readLong());
}
```

### 读取一个UTF-8格式的字符串

```
public final String readUTF() throws IOException {
	return readUTF(this);
}

public final static String readUTF(DataInput in) throws IOException {
    int utflen = in.readUnsignedShort();//还有多少字节需要读取
    byte[] bytearr = null;
    char[] chararr = null;
    if (in instanceof DataInputStream) {
        DataInputStream dis = (DataInputStream)in;
        if (dis.bytearr.length < utflen){
            dis.bytearr = new byte[utflen*2];
            dis.chararr = new char[utflen*2];
        }
        chararr = dis.chararr;
        bytearr = dis.bytearr;
    } else {
        bytearr = new byte[utflen];
        chararr = new char[utflen];
    }

    int c, char2, char3;
    int count = 0;
    int chararr_count=0;

    in.readFully(bytearr, 0, utflen);

    while (count < utflen) {
        c = (int) bytearr[count] & 0xff;
        if (c > 127) break;
        count++;
        chararr[chararr_count++]=(char)c;
    }

    while (count < utflen) {
        c = (int) bytearr[count] & 0xff;
        switch (c >> 4) {
            case 0: case 1: case 2: case 3: case 4: case 5: case 6: case 7:
                /* 0xxxxxxx*/
                count++;
                chararr[chararr_count++]=(char)c;
                break;
            case 12: case 13:
                /* 110x xxxx   10xx xxxx*/
                count += 2;
                if (count > utflen)
                    throw new UTFDataFormatException(
                        "malformed input: partial character at end");
                char2 = (int) bytearr[count-1];
                if ((char2 & 0xC0) != 0x80)
                    throw new UTFDataFormatException(
                        "malformed input around byte " + count);
                chararr[chararr_count++]=(char)(((c & 0x1F) << 6) |
                                                (char2 & 0x3F));
                break;
            case 14:
                /* 1110 xxxx  10xx xxxx  10xx xxxx */
                count += 3;
                if (count > utflen)
                    throw new UTFDataFormatException(
                        "malformed input: partial character at end");
                char2 = (int) bytearr[count-2];
                char3 = (int) bytearr[count-1];
                if (((char2 & 0xC0) != 0x80) || ((char3 & 0xC0) != 0x80))
                    throw new UTFDataFormatException(
                        "malformed input around byte " + (count-1));
                chararr[chararr_count++]=(char)(((c     & 0x0F) << 12) |
                                                ((char2 & 0x3F) << 6)  |
                                                ((char3 & 0x3F) << 0));
                break;
            default:
                /* 10xx xxxx,  1111 xxxx */
                throw new UTFDataFormatException(
                    "malformed input around byte " + count);
        }
    }
    //生成的字符数量可能小于utflen
    return new String(chararr, 0, chararr_count);
}
```