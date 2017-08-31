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

   * 返回父路径。

     ```
     public String getParent() {
         int index = path.lastIndexOf(separatorChar);
         if (index < prefixLength) {
             if ((prefixLength > 0) && (path.length() > prefixLength))
             	//文件夹上级目录
                 return path.substring(0, prefixLength);
             return null;
         }
         //文件父目录
         return path.substring(0, index);
     }
     ```

   * 获取父路径的`File`实例

     ```
     public File getParentFile() {
         String p = this.getParent();
         if (p == null) return null;
         return new File(p, this.prefixLength);
     }
     ```

   * 获取抽象路径

     ```
     public String getPath() {
         return path;
     }
     ```

   * 测试路径是否为绝对路径。Unix下为“/”开头，Windows下为，磁盘路径加“\\\”或者“\\\\\\\"

     ```
     public boolean isAbsolute() {
         return fs.isAbsolute(this);
     }
     ```

   * 获取绝对路径。

     ```
     public String getAbsolutePath() {
         return fs.resolve(this);
     }
     ```

   * 获取绝对路径的`File`实例

     ```
     public File getAbsoluteFile() {
         String absPath = getAbsolutePath();
         return new File(absPath, fs.prefixLength(absPath));
     }
     ```

   * 获取标准化路径。首先将路径转化为绝对路径，去除多余的路径名比如".","..",处理Unix的符号链接，转换Windows的磁盘字符为标准的大小写。

     ```
     public String getCanonicalPath() throws IOException {
         if (isInvalid()) {
             throw new IOException("Invalid file path");
         }
         return fs.canonicalize(fs.resolve(this));
     }
     ```

   * 获取标准化路径的`File`实例。与`new File(this.getCanonicalPath)`等价。

     ```
     public File getCanonicalFile() throws IOException {
         String canonPath = getCanonicalPath();
         return new File(canonPath, fs.prefixLength(canonPath));
     }
     ```

   * 将分割字符转化为"/"

     ```
     private static String slashify(String path, boolean isDirectory) {
         String p = path;
         if (File.separatorChar != '/')
             p = p.replace(File.separatorChar, '/');
         if (!p.startsWith("/"))
             p = "/" + p;
         if (!p.endsWith("/") && isDirectory)
             p = p + "/";
         return p;
     }
     ```

   * 将路径转化为URI

     ```
     public URI toURI() {
         try {
             File f = getAbsoluteFile();
             String sp = slashify(f.getPath(), f.isDirectory());
             if (sp.startsWith("//"))
                 sp = "//" + sp;
             return new URI("file", null, sp, null);
         } catch (URISyntaxException x) {
             throw new Error(x);         
         }
     }
     ```

   * 检查应用是否可以读取指定的文件。

     ```
     public boolean canRead() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkRead(path);
         }
         if (isInvalid()) {
             return false;
         }
         return fs.checkAccess(this, FileSystem.ACCESS_READ);
     }
     ```

   * 检查应用是否可以写入指定的文件。

     ```
     public boolean canWrite() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkWrite(path);
         }
         if (isInvalid()) {
             return false;
         }
         return fs.checkAccess(this, FileSystem.ACCESS_WRITE);
     }
     ```

   * 检查文件是否存在

     ```
     public boolean exists() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkRead(path);
         }
         if (isInvalid()) {
             return false;
         }
         return ((fs.getBooleanAttributes(this) & FileSystem.BA_EXISTS) != 0);
     }
     ```

   * 检测路径是否为文件夹

     ```
     public boolean isDirectory() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkRead(path);
         }
         if (isInvalid()) {
             return false;
         }
         return ((fs.getBooleanAttributes(this) & FileSystem.BA_DIRECTORY)
                 != 0);
     }
     ```

   * 是否为文件。

     ```
     public boolean isFile() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkRead(path);
         }
         if (isInvalid()) {
             return false;
         }
         return ((fs.getBooleanAttributes(this) & FileSystem.BA_REGULAR) != 0);
     }
     ```

   * 是否为隐藏文件

     ```
     public boolean isHidden() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkRead(path);
         }
         if (isInvalid()) {
             return false;
         }
         return ((fs.getBooleanAttributes(this) & FileSystem.BA_HIDDEN) != 0);
     }
     ```

   * 上次修改时间

     ```
     public long lastModified() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkRead(path);
         }
         if (isInvalid()) {
             return 0L;
         }
         return fs.getLastModifiedTime(this);
     }
     ```

   * 文件大小，单位为字节。

     ```
     public long length() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkRead(path);
         }
         if (isInvalid()) {
             return 0L;
         }
         return fs.getLength(this);
     }
     ```

   * 在文件不存在的状况下，原子级的创建一个空文件。判断文件是否存在和如果不存在则创建文件是一个原子操作。

     ```
     public boolean createNewFile() throws IOException {
         SecurityManager security = System.getSecurityManager();
         if (security != null) security.checkWrite(path);
         if (isInvalid()) {
             throw new IOException("Invalid file path");
         }
         return fs.createFileExclusively(path);
     }
     ```

   * 删除文件。如果为文件夹，则文件夹里必须为空。

     ```
     public boolean delete() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkDelete(path);
         }
         if (isInvalid()) {
             return false;
         }
         return fs.delete(this);
     }
     ```

   * VM终止时删除文件。只有正常终止时，动作才会生效。操作一旦提交，则不能再撤销。

     ```
     public void deleteOnExit() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkDelete(path);
         }
         if (isInvalid()) {
             return;
         }
         DeleteOnExitHook.add(path);
     }
     ```

   * 返回文件和文件夹名字的字符串数组。如果当前路径不是文件夹，则返回`null`，数组是无序的。

     ```
     public String[] list() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkRead(path);
         }
         if (isInvalid()) {
             return null;
         }
         return fs.list(this);
     }
     ```

   * 过滤返回的文件名和文件夹名数组。

     ```
     public String[] list(FilenameFilter filter) {
         String names[] = list();
         if ((names == null) || (filter == null)) {
             return names;
         }
         List<String> v = new ArrayList<>();
         for (int i = 0 ; i < names.length ; i++) {
             if (filter.accept(this, names[i])) {
                 v.add(names[i]);
             }
         }
         return v.toArray(new String[v.size()]);
     }
     ```

   * 返回当前路径下所有的`File`实例

     ```
     public File[] listFiles() {
         String[] ss = list();
         if (ss == null) return null;
         int n = ss.length;
         File[] fs = new File[n];
         for (int i = 0; i < n; i++) {
             fs[i] = new File(ss[i], this);
         }
         return fs;
     }
     ```

   * 过滤当前路径下的`File`实例数组

     ```
     public File[] listFiles(FilenameFilter filter) {
         String ss[] = list();
         if (ss == null) return null;
         ArrayList<File> files = new ArrayList<>();
         for (String s : ss)
             if ((filter == null) || filter.accept(this, s))
                 files.add(new File(s, this));
         return files.toArray(new File[files.size()]);
     }
     ```

   * 过滤

     ```
     public File[] listFiles(FileFilter filter) {
         String ss[] = list();
         if (ss == null) return null;
         ArrayList<File> files = new ArrayList<>();
         for (String s : ss) {
             File f = new File(s, this);
             if ((filter == null) || filter.accept(f))
                 files.add(f);
         }
         return files.toArray(new File[files.size()]);
     }
     ```

   * 创建文件夹

     ```
     public boolean mkdir() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkWrite(path);
         }
         if (isInvalid()) {
             return false;
         }
         return fs.createDirectory(this);
     }
     ```

   * 创建文件夹，包括创建不存在的父文件夹。如果文件夹创建失败，父目录是有可能已创建成功的

     ```
     public boolean mkdirs() {
     	//如果已存在
         if (exists()) {
             return false;
         }
         //如果不需要创建父目录
         if (mkdir()) {
             return true;
         }
         File canonFile = null;
         try {
             canonFile = getCanonicalFile();
         } catch (IOException e) {
             return false;
         }

         File parent = canonFile.getParentFile();
         //递归
         return (parent != null && (parent.mkdirs() || parent.exists()) &&
                 canonFile.mkdir());
     }
     ```

   * 重命名。操作可能不能把文件从一个文件系统中移动另外一个，操作可能不是原子的，如果目的路径已存在也会失败。

     ```
     public boolean renameTo(File dest) {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkWrite(path);
             security.checkWrite(dest.path);
         }
         if (dest == null) {
             throw new NullPointerException();
         }
         if (this.isInvalid() || dest.isInvalid()) {
             return false;
         }
         return fs.rename(this, dest);
     }
     ```

   * 修改最后修改时间

     ```
     public boolean setLastModified(long time) {
         if (time < 0) throw new IllegalArgumentException("Negative time");
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkWrite(path);
         }
         if (isInvalid()) {
             return false;
         }
         return fs.setLastModifiedTime(this, time);
     }
     ```

   * 设为只读

     ```
     public boolean setReadOnly() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkWrite(path);
         }
         if (isInvalid()) {
             return false;
         }
         return fs.setReadOnly(this);
     }
     ```

   * 设为可写入，`writable`表示写入权限，`ownerOnly`表示是否只有所有者才能写入。

     ```
     public boolean setWritable(boolean writable, boolean ownerOnly) {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkWrite(path);
         }
         if (isInvalid()) {
             return false;
         }
         return fs.setPermission(this, FileSystem.ACCESS_WRITE, writable, ownerOnly);
     }
     ```

   * 所有者才能写入的简单方法。其他类似的`setReadable`和`setExecutable`都有这两个方法。

     ```
     public boolean setWritable(boolean writable) {
         return setWritable(writable, true);
     }
     ```

   * 检测文件是否可执行。

     ```
     public boolean canExecute() {
         SecurityManager security = System.getSecurityManager();
         if (security != null) {
             security.checkExec(path);
         }
         if (isInvalid()) {
             return false;
         }
         return fs.checkAccess(this, FileSystem.ACCESS_EXECUTE);
     }
     ```

   * 文件系统根目录。标准化路径就是以更目录开头。

     ```
     public static File[] listRoots() {
         return fs.listRoots();
     }
     ```

   * 返回分区总大小。

     ```
     public long getTotalSpace() {
         SecurityManager sm = System.getSecurityManager();
         if (sm != null) {
             sm.checkPermission(new RuntimePermission("getFileSystemAttributes"));
             sm.checkRead(path);
         }
         if (isInvalid()) {
             return 0L;
         }
         return fs.getSpace(this, FileSystem.SPACE_TOTAL);
     }
     ```

   * 返回分区空余空间

     ```
     public long getFreeSpace() {
         SecurityManager sm = System.getSecurityManager();
         if (sm != null) {
             sm.checkPermission(new RuntimePermission("getFileSystemAttributes"));
             sm.checkRead(path);
         }
         if (isInvalid()) {
             return 0L;
         }
         return fs.getSpace(this, FileSystem.SPACE_FREE);
     }
     ```

   * 返回已使用空间

     ```
     public long getUsableSpace() {
         SecurityManager sm = System.getSecurityManager();
         if (sm != null) {
             sm.checkPermission(new RuntimePermission("getFileSystemAttributes"));
             sm.checkRead(path);
         }
         if (isInvalid()) {
             return 0L;
         }
         return fs.getSpace(this, FileSystem.SPACE_USABLE);
     }
     ```

   * 临时文件类

     ```
     private static class TempDirectory {
         private TempDirectory() { }

         //临时文件路径
         private static final File tmpdir = new File(AccessController
             .doPrivileged(new GetPropertyAction("java.io.tmpdir")));
         static File location() {
             return tmpdir;
         }

         //生成文件名
         private static final SecureRandom random = new SecureRandom();
         static File generateFile(String prefix, String suffix, File dir)
             throws IOException
         {
             long n = random.nextLong();
             if (n == Long.MIN_VALUE) {
                 n = 0;      // corner case
             } else {
                 n = Math.abs(n);
             }

             //使用指定前缀的文件名
             prefix = (new File(prefix)).getName();

             String name = prefix + Long.toString(n) + suffix;
             File f = new File(dir, name);
             if (!name.equals(f.getName()) || f.isInvalid()) {
                 if (System.getSecurityManager() != null)
                     throw new IOException("Unable to create temporary file");
                 else
                     throw new IOException("Unable to create temporary file, " + f);
             }
             return f;
         }
     }
     ```

   * 在指定的文件夹中生成一个空白的文件。方法调用成功表示方法调用前文件不存在，而且在当前VM的调用中，不会再生成相同的路径。

     ```
     public static File createTempFile(String prefix, String suffix,
                                       File directory)
         throws IOException
     {
         if (prefix.length() < 3)
             throw new IllegalArgumentException("Prefix string too short");
         if (suffix == null)
             suffix = ".tmp";

         File tmpdir = (directory != null) ? directory
                                           : TempDirectory.location();
         SecurityManager sm = System.getSecurityManager();
         File f;
         do {
             f = TempDirectory.generateFile(prefix, suffix, tmpdir);

             if (sm != null) {
                 try {
                     sm.checkWrite(f.getPath());
                 } catch (SecurityException se) {
                     // don't reveal temporary directory location
                     if (directory == null)
                         throw new SecurityException("Unable to create temporary file");
                     throw se;
                 }
             }
         } while ((fs.getBooleanAttributes(f) & FileSystem.BA_EXISTS) != 0);

         if (!fs.createFileExclusively(f.getPath()))
             throw new IOException("Unable to create temporary file");

         return f;
     }
     ```