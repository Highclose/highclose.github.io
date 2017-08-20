---
layout: post
title:  "Source code of String in java."
description: "Analyze source code of String."
categories: [java]
tags: [source, java]
---

# sourceCode of String in java



1.implements
>
	    java.io.Serializable, Comparable<String>, CharSequence

2.fields
> //保存字符数组. 
>
	    private final char value[];

> //缓存字符串的hash码 
>
	    private int hash;

> //序列号.
>
	    private static final long serialVersionUID = -6849794470754667710L;

> //用作序列化的数组
>
	    private static final ObjectStreamField[] serialPersistentFields =  new ObjectStreamField[0];

> //忽略大小写的字符串比较类.
>
	    public static final Comparator<String> CASE_INSENSITIVE_ORDER = new CaseInsensitiveComparator();
>
>

3.methods

* constructor

  > //String的无参构造函数，创建一个空字符序列的实例，但是因为String类型是不可变的，所以这个构造函数是不需要用到的（毕竟一个空的字符串也没啥用，只要用“”表示就好了）。
  >
      public String() {
          this.value = "".value;
      }

  >//original的备份。
  >
      public String(String original) {
          this.value = original.value;
          this.hash = original.hash;
      }

  >//字符序列转化为字符串，用复制的方法保证字符串的不可变性（字符序列的变动不会影响字符串）
  >
      public String(char value[]) {
          this.value = Arrays.copyOf(value, value.length);
      }

  >//截取字符序列转化为字符串
  >
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

  >//将代码点数组转化为字符串
  >
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

* others