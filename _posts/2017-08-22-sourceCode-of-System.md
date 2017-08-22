---
layout: post
category: java
title:  "Source code of System in java."
comments: true
description: "Analyze source code of System."
tags: [java,source]
---



System的源码分析。

<!--more-->

1. fields

   * 静态代码块，VM会调用`initializeSystemClass`方法来完成初始化，与clinit相互独立。

     ```
         private static native void registerNatives();
         static {
             registerNatives();
         }
     ```

   * 标准输入流。这个流是已经打开了并且已经可以输入了。

     ```
     public final static InputStream in = null;
     ```

   * 标准输出流。这个流是已经打开的并且可以接受输出了。

     ```
     public final static PrintStream out = null;
     ```

   * 标准错误输出流。out被重定向到文件或其他路径用来做监控。

     ```
     public final static PrintStream err = null;
     ```

   * 安全管理器

     ```
     private static volatile SecurityManager security = null;
     ```

   * 控制台

     ```
     private static volatile Console cons = null;
     ```

   * 系统属性，以下属性保证会被定义：

     > java.version java版本
     >
     > java.vendor java提供商
     >
     > java.vendor.url   提供商网址
     >
     > java.home  java安装路径
     >
     > java.class.version  class版本
     >
     > java.class.path  java执行路径
     >
     > os.name   操作系统名称
     >
     > os.arch    操作系统构架
     >
     > os.version   操作系统版本
     >
     > file.separator    文件分隔符（Unix里为“/”）
     >
     > path separator  路径分隔符("Unix里为“:”)
     >
     > line.separator  换行符( Unix里为"\n")
     >
     > user.name    用户名
     >
     > user.home    用户home目录
     >
     > user.dir  用户当前目录

     ```
     private static Properties props;
     ```

   * 换行符

     ```
     private static String lineSeparator;
     ```

2. methods

   * 私有构造函数，保证不会被实例化。

     ```
         private System() {
         }
     ```

   * 由于java是支持多线程的，所以标准的输入输出是共享，因此它们必须受到特别的处理，在系统初始化完成之前，线程严禁使用这几个特殊对象；又因为这些对象都是静态的，因此java的类加载机制会在System类加载的时候就会初始化，这就造成了一对矛盾；为解决这对矛盾，System在加载是将它们初始化为null，等加在完成后，通过native方法在对它们进行赋值

     ```
     public static void setIn(InputStream in) {
         checkIO();
         setIn0(in);
     }
     public static void setOut(PrintStream out) {
             checkIO();
             setOut0(out);
         }
     public static void setErr(PrintStream err) {
             checkIO();
             setErr0(err);
         }
     private static void checkIO() {
         SecurityManager sm = getSecurityManager();
         if (sm != null) {
             sm.checkPermission(new RuntimePermission("setIO"));
         }
     }

     private static native void setIn0(InputStream in);
     private static native void setOut0(PrintStream out);
     private static native void setErr0(PrintStream err);

     public static SecurityManager getSecurityManager() {
     		return security;
     }

     ```

   * 返回一个当前JVM联结的唯一的控制台。

     ```
     public static Console console() {
         if (cons == null) {
             synchronized (System.class) {
                 cons = sun.misc.SharedSecrets.getJavaIOAccess().console();
             }
         }
         return cons;
     }
     ```

   * //

     ```
     public static Channel inheritedChannel() throws IOException {
         return SelectorProvider.provider().inheritedChannel();
     }
     ```