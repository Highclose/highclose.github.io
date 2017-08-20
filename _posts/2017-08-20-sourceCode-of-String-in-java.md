---
layout: post
title:  "Source code of String in java"
date:   2017-08-20 13:08:09 -0700
categories: java source
tags: java source
---

# sourceCode of String in java



1.implements
>java.io.Serializable, Comparable<String>, CharSequence

2.fields
> /**保存字符数组. */
> private final char value[];
> /**缓存字符串的hash码 */
> private int hash;
> /**序列号*/
> private static final long serialVersionUID = -6849794470754667710L;
> /**用作序列化的数组*/
> private static final ObjectStreamField[] serialPersistentFields =  new ObjectStreamField[0];
> /**忽略大小写的字符串比较类.*/
> public static final Comparator<String> CASE_INSENSITIVE_ORDER = new CaseInsensitiveComparator();

3.

