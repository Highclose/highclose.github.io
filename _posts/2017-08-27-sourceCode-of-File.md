---
layout: post
category: java
title:  "Source code of File in java."
comments: true
description: "Analyze source code of File ."
tags: [java,source]
---



File 的源码分析。

<!--more-->

这个类是文件和文件夹的一个抽象的表示。用户接口或操作系统使用与系统相关的路径字符串来指定文件或文件夹。`File`类表示一个抽象的，与系统无关的分层路径。一个抽象的路径包含两部分：

​	一个可选的与系统相关的字符串前缀，比如磁盘路径。Unit根目录为“/”，Windows UNC路径为“\\\\\\\"。

​	一个0序列或更多的文件路径名字。

在一个抽象的路径名字中，第一个名字可能是一个文件夹名或者是一个主机名（在Windows UNC中）。抽象路径中的每个子序列表示一个文件夹。最后一个名字可能代表一个文件或一个文件夹。空的抽象路径没有前缀，也没有路径序列。

一个文件路径字符串转换成/自抽象路径名是与系统相关的。抽象路径转换成文件路径，每个名字中加一个默认的分隔字符（在file.separator中定义）。反之亦然。

一个路径不管是否是抽象或真实的，都有绝对或相对路径的区别。`java.io`包中的类默认使用的是相对路径与当前用户目录进行组合。这个目录由系统变量`user.dir`定义，典型的来说，JVM也是在这个目录中调用的。

File实例是不可变的。一旦创建，File对象表示的抽象路径就不会变。

1. implements

   ```
   Serializable, Comparable<File>
   ```

2. fields

   * 平台本地的文件系统。

     ```
     private static final FileSystem fs = DefaultFileSystem.getFileSystem();
     ```

   * 一个抽象的，标准化的路径名（使用了默认的名字分隔符，且不包含多余或重复的分隔符）。

     ```
     private final String path;
     ```

   * 文件路径状态的枚举类型。

     ```
     private static enum PathStatus { INVALID, CHECKED };
     ```

   * 文件路径状态标志位

     ```
     private transient PathStatus status = null;
     ```

   * 抽象文件名的前缀长度。

     ```
     private final transient int prefixLength;
     ```

   * 系统相关的默认系统分隔符。初始化时包含了系统变量`file.separator`的第一个字符值。Unix为”/“，windows为”\\\"

     ```
     public static final char separatorChar = fs.getSeparator();
     ```

   * 转化为String

     ```
     public static final String separator = "" + separatorChar;
     ```

   * 系统相关的路径分隔符。初始化时包含了系统变量`path.separator`的第一个字符值。Unix为”:“，windows为”;"

     ```
     public static final char pathSeparatorChar = fs.getPathSeparator();
     ```

   * 转化为字符串

     ```
     public static final String pathSeparator = "" + pathSeparatorChar;
     ```

   * 静态代码块，获取`path`,`prefixLength`相对于对象的内存偏移量。

     ```
     private static final long PATH_OFFSET;
     private static final long PREFIX_LENGTH_OFFSET;
     private static final sun.misc.Unsafe UNSAFE;
     static {
         try {
             sun.misc.Unsafe unsafe = sun.misc.Unsafe.getUnsafe();
             PATH_OFFSET = unsafe.objectFieldOffset(
                     File.class.getDeclaredField("path"));
             PREFIX_LENGTH_OFFSET = unsafe.objectFieldOffset(
                     File.class.getDeclaredField("prefixLength"));
             UNSAFE = unsafe;
         } catch (ReflectiveOperationException e) {
             throw new Error(e);
         }
     ```

3. methods

   * 内部构造函数。

     ```
     private File(String pathname, int prefixLength) {
         this.path = pathname;
         this.prefixLength = prefixLength;
     }
     ```

   * 内部构造函数。参数的顺序是为了与public(File, String)构造函数区分。

     ```
     private File(String child, File parent) {
         assert parent.path != null;
         assert (!parent.path.equals(""));
         this.path = fs.resolve(parent.path, child);
         this.prefixLength = parent.prefixLength;
     }
     ```

   * 构造函数。

     ```
     public File(String pathname) {
         if (pathname == null) {
             throw new NullPointerException();
         }
         this.path = fs.normalize(pathname);
         this.prefixLength = fs.prefixLength(this.path);
     }
     ```

   * 双参数构造函数。

     ```
     public File(String parent, String child) {
         if (child == null) {
             throw new NullPointerException();
         }
         if (parent != null) {
             if (parent.equals("")) {
             	//parent为空，则Child转换为抽象路径并与默认路径结合
             	//默认路径：Unix为“/”，Windows为“\\”
                 this.path = fs.resolve(fs.getDefaultParent(),
                                        fs.normalize(child));
             } else {
                 this.path = fs.resolve(fs.normalize(parent),
                                        fs.normalize(child));
             }
         } else {
         	//parent为空，则为单参数构造函数，参数为child
             this.path = fs.normalize(child);
         }
         this.prefixLength = fs.prefixLength(this.path);
     }
     ```

   * 双参数构造函数，与前一个类似。

     ```
     public File(File parent, String child) {
         if (child == null) {
             throw new NullPointerException();
         }
         if (parent != null) {
             if (parent.path.equals("")) {
                 this.path = fs.resolve(fs.getDefaultParent(),
                                        fs.normalize(child));
             } else {
                 this.path = fs.resolve(parent.path,
                                        fs.normalize(child));
             }
         } else {
             this.path = fs.normalize(child);
         }
         this.prefixLength = fs.prefixLength(this.path);
     }
     ```

   * 用`file:`URI创建一个新的`File`实例，具体的`file:`URI 格式是系统相关的，因此转换也是系统相关的，对于一个抽象路径来说，只要原抽象路径，URI，转换的抽象路径是在同一个JVM里生成，以下等式成立：new File(f.toURI()).equals(f.getAbsoluteFile())

     ```
     public File(URI uri) {
         if (!uri.isAbsolute())
             throw new IllegalArgumentException("URI is not absolute");
         if (uri.isOpaque())
             throw new IllegalArgumentException("URI is not hierarchical");
         String scheme = uri.getScheme();
         if ((scheme == null) || !scheme.equalsIgnoreCase("file"))
             throw new IllegalArgumentException("URI scheme is not \"file\"");
         if (uri.getAuthority() != null)
             throw new IllegalArgumentException("URI has an authority component");
         if (uri.getFragment() != null)
             throw new IllegalArgumentException("URI has a fragment component");
         if (uri.getQuery() != null)
             throw new IllegalArgumentException("URI has a query component");
         String p = uri.getPath();
         if (p.equals(""))
             throw new IllegalArgumentException("URI path component is empty");
         p = fs.fromURIPath(p);
         if (File.separatorChar != '/')
             p = p.replace('/', File.separatorChar);
         this.path = fs.normalize(p);
         this.prefixLength = fs.prefixLength(this.path);
     }
     ```

   * 检查文件路径是否有效。目前检测的方法非常有限。只是做了空字符检测。返回true代表文件路径无效，返回false不保证有效。

     ```
     final boolean isInvalid() {
         if (status == null) {
             status = (this.path.indexOf('\u0000') < 0) ? PathStatus.CHECKED
                                                        : PathStatus.INVALID;
         }
         return status == PathStatus.INVALID;
     }
     ```

   * 返回名字

     ```
     public String getName() {
         int index = path.lastIndexOf(separatorChar);
         //文件夹名
         if (index < prefixLength) return path.substring(prefixLength);
         //文件名
         return path.substring(index + 1);
     }
     ```