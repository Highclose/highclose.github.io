---
layout: post
category: java
title:  "Source code of String in java."
comments: true
description: "Analyze source code of String."
tags: [java, source]
---

String的源码分析。

<!--more-->

1. implements

      java.io.Serializable, Comparable<String>, CharSequence
2. fields

* 保存字符数组. 

      private final char value[];

* 缓存字符串的hash码 

      private int hash;

* 序列号

      private static final long serialVersionUID = -6849794470754667710L;
* 用作序列化的数组

      private static final ObjectStreamField[] serialPersistentFields =  new ObjectStreamField[0];

* 忽略大小写的字符串比较类

      public static final Comparator<String> CASE_INSENSITIVE_ORDER = new CaseInsensitiveComparator();
  ​    

3. methods

* String的无参构造函数，创建一个空字符序列的实例，但是因为String类型是不可变的，所以这个构造函数是不需要用到的（毕竟一个空的字符串也没啥用，只要用“”表示就好了）。
  ​    
      public String() {
          this.value = "".value;
      }

* original的备份。

      public String(String original) {
          this.value = original.value;
          this.hash = original.hash;
      }

* 字符序列转化为字符串，用复制的方法保证字符串的不可变性（字符序列的变动不会影响字符串）

      public String(char value[]) {
          this.value = Arrays.copyOf(value, value.length);
      }

* 截取字符序列转化为字符串

      public String(char value[], int offset, int count) {
          if (offset < 0) {
          throw new StringIndexOutOfBoundsException(offset);
          }
          if (count <= 0) {
              if (count < 0) {
                  throw new StringIndexOutOfBoundsException(count);
              }
              if (offset <= value.length) {
              this.value = "".value;
              return;
              }
          }
          if (offset > value.length - count) {
              throw new StringIndexOutOfBoundsException(offset + count);
          }
          this.value = Arrays.copyOfRange(value, offset, offset+count);
      }

* 将代码点数组转化为字符串

      public String(int[] codePoints, int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count <= 0) {
            if (count < 0) {
                throw new StringIndexOutOfBoundsException(count);
            }
            if (offset <= codePoints.length) {
                this.value = "".value;
                return;
            }
        }
        if (offset > codePoints.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }
        final int end = offset + count;      
        int n = count;
        for (int i = offset; i < end; i++) {
            int c = codePoints[i];
            //判断代码点是否在Basic Multilingual Plane,BMP中
            //BMP code point是2字节，超出则属于有效的代码点
            if (Character.isBmpCodePoint(c))
                continue;
            //判断数字是否是有效的代码点
            else if (Character.isValidCodePoint(c))
                n++;
            else throw new IllegalArgumentException(Integer.toString(c));
        }
        final char[] v = new char[n];
        for (int i = offset, j = 0; i < end; i++, j++) {
            int c = codePoints[i];
            //如果在BMP中，则转化为char
            if (Character.isBmpCodePoint(c))
                v[j] = (char)c;
            else
            	  //2字节unicode只能表示65536个字符，超出这65536个字符表示
            	  //则是从65536个编号里拿出2048个，规定他们为surrogates，
            	  //让他们两个为一组，来表示其他的字符。
            	  //high surrogates1024个，low surrogates1024个
                Character.toSurrogates(c, v, j++);
        }
        this.value = v;
        }

* 将字节数组的子集用制定的字符集解码，新生成的字符串长度可能会与子集长度不一致。

      public String(byte bytes[], int offset, int length, String charsetName)
            throws UnsupportedEncodingException {
        if (charsetName == null)
            throw new NullPointerException("charsetName");
        //检查位移量与长度是否符合字节数组的长度。
        checkBounds(bytes, offset, length);
        this.value = StringCoding.decode(charsetName, bytes, offset, length);
      }

* 与上个方法差不多，只是用了charset的实例来指定字符集，这是JDK1.6以后出来的构造方法。

      public String(byte bytes[], int offset, int length, Charset charset) {
        if (charset == null)
            throw new NullPointerException("charset");
        checkBounds(bytes, offset, length);
        this.value =  StringCoding.decode(charset, bytes, offset, length);
        }

* 其他还有4个bytes的构造方法，原理都差不多。

      public String(byte bytes[])， public String(byte bytes[], int offset, int length)，public String(byte bytes[], Charset charset),public String(byte bytes[], Charset charset)

* StringBuffer的构造方法

      public String(StringBuffer buffer) {
      //线程安全。
        synchronized(buffer) {
            //用复制的方法保证字符串的不可变性
            this.value = Arrays.copyOf(buffer.getValue(), buffer.length());
        }
        }

* StringBuilder的构造方法（非线程安全，并且通常，StringBuild的toString()方法速度更快，所以更推荐）

  ```
  public String(StringBuilder builder) {
  	//复制保证不可变性
      this.value = Arrays.copyOf(builder.getValue(), builder.length());
  }
  ```

* 私有的构造方法，与不带boolean值的public构造方法不同的是，没有采用copy的方式，而是共享了数组。

  ```
  String(char[] value, boolean share) {
      // assert share : "unshared not supported";
      this.value = value;
  }
  ```

* 静态方法，调用实例的toString方法。

  ```
  public static String valueOf(Object obj) {
      return (obj == null) ? "null" : obj.toString();
  }
  ```

* 调用了自己的构造方法，使用字符数组生成要给新的字符串实例，其他还有诸如copyValueOf(char data[])，copyValueOf(char data[], int offset, int count)， valueOf(char data[], int offset, int count) 基本都差不多。

  ```
  public static String valueOf(char data[]) {
      return new String(data);
  }
  ```

* boolean

  ```
  public static String valueOf(boolean b) {
      return b ? "true" : "false";
  }
  ```

* 调用了私有的构造方法。

  ```
  public static String valueOf(char c) {
      char data[] = {c};
      return new String(data, true);
  }
  ```

* 调用基础类型封装类的toString方法，相同的还有long,float,double

  ```
  public static String valueOf(int i) {
      return Integer.toString(i);
  }
  ```

* String转化为char数组

  ```
  public char[] toCharArray() {
      // 因为类加载的顺序关系不能使用Arrays.copyOf方法
      char result[] = new char[value.length];
      System.arraycopy(value, 0, result, 0, value.length);
      return result;
  }
  ```

* 将String头尾编码在空格及以前的字符删除，

  ```
  public String trim() {
      int len = value.length;
      int st = 0;
      char[] val = value;   

      while ((st < len) && (val[st] <= ' ')) {
          st++;
      }
      while ((st < len) && (val[len - 1] <= ' ')) {
          len--;
      }
      return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
  }
  ```

* startsWith(String prefix)，endsWith(String suffix)都调用了这个方法

  ```
  public boolean startsWith(String prefix, int toffset) {
  	//本字符数组
      char ta[] = value;
      //位移量
      int to = toffset;
      //匹配字符数组
      char pa[] = prefix.value;
      int po = 0;
      //匹配字符数组长度
      int pc = prefix.value.length;
      if ((toffset < 0) || (toffset > value.length - pc)) {
          return false;
      }
      //逐个进行比较
      while (--pc >= 0) {
          if (ta[to++] != pa[po++]) {
              return false;
          }
      }
      return true;
  }
  ```

* 计算hashcode

  ```
  public int hashCode() {
      int h = hash;
      if (h == 0 && value.length > 0) {
          char val[] = value;

          for (int i = 0; i < value.length; i++) {
              h = 31 * h + val[i];
          }
          hash = h;
      }
      return h;
  }
  ```

* 连接字符串

  ```
  public String concat(String str) {
      int otherLen = str.length();
      if (otherLen == 0) {
          return this;
      }
      int len = value.length;
      //生成一个新的数组，长度为两个String的长度之和，前面部分为本字符串
      char buf[] = Arrays.copyOf(value, len + otherLen);
      //在buf数组中，从len位置开始，加入str的字符数组
      str.getChars(buf, len);
      //返回新的字符串
      return new String(buf, true);
  }
  ```

* 其中的getChars方法

  ```
  void getChars(char dst[], int dstBegin) {
      //value:原数组
      //0：原数组起始位
      //dst:目标数组
      //dstBegin:目标数组起始位
      //value.length：将要复制的字符数组长度
      System.arraycopy(value, 0, dst, dstBegin, value.length);
  }
  ```

* 判断相等

  ```
  public boolean equals(Object anObject) {
  	//是否为同一个实例
      if (this == anObject) {
          return true;
      }
      //是否为字符串实例
      if (anObject instanceof String) {
          String anotherString = (String)anObject;
          int n = value.length;
          //先判断两个字符串长度是否相等，是则循环比较
          if (n == anotherString.value.length) {
              char v1[] = value;
              char v2[] = anotherString.value;
              int i = 0;
              while (n-- != 0) {
                  if (v1[i] != v2[i])
                      return false;
                  i++;
              }
              return true;
          }
      }
      return false;
  }
  ```

* 判断内容是否相等

  ```
  public boolean contentEquals(CharSequence cs) {
      if (cs instanceof AbstractStringBuilder) {
          if (cs instanceof StringBuffer) {
          	//线程安全
              synchronized(cs) {
                 return nonSyncContentEquals((AbstractStringBuilder)cs);
              }
          } else {
          	//StringBuilder
              return nonSyncContentEquals((AbstractStringBuilder)cs);
          }
      }
      // 如果是字符串，调用上一个方法
      if (cs instanceof String) {
          return equals(cs);
      }
      //CharSequence
      char v1[] = value;
      int n = v1.length;
      if (n != cs.length()) {
          return false;
      }
      for (int i = 0; i < n; i++) {
          if (v1[i] != cs.charAt(i)) {
              return false;
          }
      }
      return true;
  }
  ```

* 非同步比较方法，StringBuffer在调用处用synchronized来保证线程安全

  ```
  private boolean nonSyncContentEquals(AbstractStringBuilder sb) {
      char v1[] = value;
      char v2[] = sb.getValue();
      int n = v1.length;
      if (n != sb.length()) {
          return false;
      }
      for (int i = 0; i < n; i++) {
          if (v1[i] != v2[i]) {
              return false;
          }
      }
      return true;
  }
  ```

* 忽略大小写判断是否相等

  ```
  public boolean equalsIgnoreCase(String anotherString) {
      return (this == anotherString) ? true
              : (anotherString != null)
              && (anotherString.value.length == value.length)
              && regionMatches(true, 0, anotherString, 0, value.length);
  }
  ```

* 其中的regionMatches方法

  ```
  public boolean regionMatches(boolean ignoreCase, int toffset,
          String other, int ooffset, int len) {
      char ta[] = value;
      int to = toffset;
      char pa[] = other.value;
      int po = ooffset;
      if ((ooffset < 0) || (toffset < 0)
              || (toffset > (long)value.length - len)
              || (ooffset > (long)other.value.length - len)) {
          return false;
      }
      while (len-- > 0) {
          char c1 = ta[to++];
          char c2 = pa[po++];
          if (c1 == c2) {
              continue;
          }
          if (ignoreCase) {
  			//都转化为大写进行比较
              char u1 = Character.toUpperCase(c1);
              char u2 = Character.toUpperCase(c2);
              if (u1 == u2) {
                  continue;
              }
              // 格鲁吉亚语的大写规则会比较独特，所以转化为小写再比较一遍
              if (Character.toLowerCase(u1) == Character.toLowerCase(u2)) {
                  continue;
              }
          }
          return false;
      }
      return true;
  }
  ```

* 字符串比较（第一个不同的字符，返回两者的unicode差值，0则为相等）

  ```
  public int compareTo(String anotherString) {
      int len1 = value.length;
      int len2 = anotherString.value.length;
      int lim = Math.min(len1, len2);
      char v1[] = value;
      char v2[] = anotherString.value;

      int k = 0;
      while (k < lim) {
          char c1 = v1[k];
          char c2 = v2[k];
          if (c1 != c2) {
              return c1 - c2;
          }
          k++;
      }
      return len1 - len2;
  }
  ```

* 忽略大小写进行比较

  ```
  public int compareToIgnoreCase(String str) {
  	//在内部类中进行比较
      return CASE_INSENSITIVE_ORDER.compare(this, str);
  }
  ```

* 内部类

  ```
  public static final Comparator<String> CASE_INSENSITIVE_ORDER
                                           = new CaseInsensitiveComparator();
  private static class CaseInsensitiveComparator
          implements Comparator<String>, java.io.Serializable {
      private static final long serialVersionUID = 8575799808933029326L;

      public int compare(String s1, String s2) {
          int n1 = s1.length();
          int n2 = s2.length();
          int min = Math.min(n1, n2);
          for (int i = 0; i < min; i++) {
              char c1 = s1.charAt(i);
              char c2 = s2.charAt(i);
              if (c1 != c2) {
                  c1 = Character.toUpperCase(c1);
                  c2 = Character.toUpperCase(c2);
                  if (c1 != c2) {
                      c1 = Character.toLowerCase(c1);
                      c2 = Character.toLowerCase(c2);
                      if (c1 != c2) {
                          return c1 - c2;
                      }
                  }
              }
          }
          return n1 - n2;
      }

      private Object readResolve() { return CASE_INSENSITIVE_ORDER; }
  }
  ```

* 获取字节数组（默认为"ISO-8859-1"，也可以指定）

  ```
  public byte[] getBytes() {
      return StringCoding.encode(value, 0, value.length);
  }
  ```

* 替换（其他的replace以及matches都调用了Pattern类的方法）

  ```
  public String replace(char oldChar, char newChar) {
      if (oldChar != newChar) {
          int len = value.length;
          int i = -1;
          char[] val = value; /* avoid getfield opcode */

          while (++i < len) {
              if (val[i] == oldChar) {
                  break;//i<len
              }
          }
          if (i < len) {
              char buf[] = new char[len];
              //因为原数组是final,所以先复制
              for (int j = 0; j < i; j++) {
                  buf[j] = val[j];
              }
              //替换
              while (i < len) {
                  char c = val[i];
                  buf[i] = (c == oldChar) ? newChar : c;
                  i++;
              }
              return new String(buf, true);
          }
      }
      return this;
  }
  ```

* Split(平时常用的split(String regex)调用的是这个函数，limit=0)

  ```
  public String[] split(String regex, int limit) {
      char ch = 0;
      //如果regex长度为1，且不在".$|()[{^?*+\\"中
      //如果regex长度为2，第一位是"\",第二位不是数字或者字母
      if (((regex.value.length == 1 &&
           ".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) == -1) ||
           (regex.length() == 2 &&
            regex.charAt(0) == '\\' &&
            (((ch = regex.charAt(1))-'0')|('9'-ch)) < 0 &&
            ((ch-'a')|('z'-ch)) < 0 &&
            ((ch-'A')|('Z'-ch)) < 0)) &&
          (ch < Character.MIN_HIGH_SURROGATE ||
           ch > Character.MAX_LOW_SURROGATE))
      {
          int off = 0;
          int next = 0;
          boolean limited = limit > 0;
          ArrayList<String> list = new ArrayList<>();
          while ((next = indexOf(ch, off)) != -1) {
              if (!limited || list.size() < limit - 1) {
                  list.add(substring(off, next));
                  off = next + 1;
              } else {    // 最后一个
                  //断言 (list.size() == limit - 1);
                  list.add(substring(off, value.length));
                  off = value.length;
                  break;
              }
          }
          //如果没有匹配，则
          if (off == 0)
              return new String[]{this};

          // 把字符串剩下的子字符串放入列表
          if (!limited || list.size() < limit)
              list.add(substring(off, value.length));

          // 结果
          int resultSize = list.size();
          if (limit == 0) {
              while (resultSize > 0 && list.get(resultSize - 1).length() == 0) {
                  resultSize--;
              }
          }
          String[] result = new String[resultSize];
          //列表转化为数组
          return list.subList(0, resultSize).toArray(result);
      }
      //其他条件调用了Pattern
      return Pattern.compile(regex).split(this, limit);
  }
  ```

* indexOf

  ```
  //source 原字符数组
  //sourceOffset 原数组位移
  //sourceCount 原数组长度
  //target 查找字符数组
  //targetOffset 查找字符数组位移
  //targetCount 目标长度
  //fromIndex 查找开始位置
  static int indexOf(char[] source, int sourceOffset, int sourceCount,
          char[] target, int targetOffset, int targetCount,
          int fromIndex) {
      //位置超过原数组长度，目标长度如果为0，则返回原数组长度，否则返回-1
      if (fromIndex >= sourceCount) {
          return (targetCount == 0 ? sourceCount : -1);
      }
      if (fromIndex < 0) {
          fromIndex = 0;
      }
      //目标长度为0，返回fromIndex
      if (targetCount == 0) {
          return fromIndex;
      }
      //目标字符数组第一个字符
      char first = target[targetOffset];
      //原数组最大查找位置
      int max = sourceOffset + (sourceCount - targetCount);

      for (int i = sourceOffset + fromIndex; i <= max; i++) {
          //先比较第一个字符是否相等
          if (source[i] != first) {
              while (++i <= max && source[i] != first);
          }

          //第一个字符相等
          if (i <= max) {
              int j = i + 1;
              int end = j + targetCount - 1;
              for (int k = targetOffset + 1; j < end && source[j]
                      == target[k]; j++, k++);

              if (j == end) {
                  //找到
                  return i - sourceOffset;
              }
          }
      }
      return -1;
  }
  ```

* 连接

  ```
  //delimiter 分隔符
  //elements 需要连接的数据(String,StringBuilder,StringBuffer)
  public static String join(CharSequence delimiter, CharSequence... elements) {
      Objects.requireNonNull(delimiter);
      Objects.requireNonNull(elements);
      StringJoiner joiner = new StringJoiner(delimiter);
      for (CharSequence cs: elements) {
          joiner.add(cs);
      }
      return joiner.toString();
  }
  ```

* public native String intern()

  返回一个规范化的字符串。String类私有的维护一个初始为空的字符串池。当本方法调用时，如果池中已经有一个相同值的字符串（使用equals(String)方法），则返回池中的字符串，否则，把本字符串添加到池中，并返回这个字符串的引用。

  ​

4. String的不可变性

* final修饰符，保证不被继承
* 成员变量为private,且没有setter方法，保证不会被修改
* 传入的成员都进行拷贝，保证不会被传入的数据改变。
* 返回时也进行拷贝