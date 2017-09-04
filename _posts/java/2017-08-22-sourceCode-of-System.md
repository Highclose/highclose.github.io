---
layout: post
category: java
title:  "Source code of System in java."
description: "Analyze source code of System."
tags: [java,source]
---



## fields

### 静态代码块，VM会调用`initializeSystemClass`方法来完成初始化，与clinit相互独立。

     
         private static native void registerNatives();
         static {
             registerNatives();
         }
     

### 标准输入流。这个流是已经打开了并且已经可以输入了。

     
     public final static InputStream in = null;
     

### 标准输出流。这个流是已经打开的并且可以接受输出了。

     
     public final static PrintStream out = null;
     

### 标准错误输出流。out被重定向到文件或其他路径用来做监控。

     
     public final static PrintStream err = null;
     

### 安全管理器

     
     private static volatile SecurityManager security = null;
     

### 控制台

     
     private static volatile Console cons = null;
     

### 系统属性，以下属性保证会被定义：

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
    
     
     private static Properties props;
     

### 换行符

     
     private static String lineSeparator;
     

## methods

### 私有构造函数，保证不会被实例化。

     
         private System() {
         }
     

### 由于java是支持多线程的，所以标准的输入输出是共享，因此它们必须受到特别的处理，在系统初始化完成之前，线程严禁使用这几个特殊对象；又因为这些对象都是静态的，因此java的类加载机制会在System类加载的时候就会初始化，这就造成了一对矛盾；为解决这对矛盾，System在加载是将它们初始化为null，等加在完成后，通过native方法在对它们进行赋值

     
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
     

### 返回一个当前JVM联结的唯一的控制台。

     
     public static Console console() {
         if (cons == null) {
             synchronized (System.class) {
                 cons = sun.misc.SharedSecrets.getJavaIOAccess().console();
             }
         }
         return cons;
     }
     

### 返回继承于当前JVM创建的实体的通道。

     
     public static Channel inheritedChannel() throws IOException {
         return SelectorProvider.provider().inheritedChannel();
     }
     

### 如果已经部署了一个安全管理器，这个方法会用`RuntimePermission("setSecurityManager")`来调用这个安全管理器的`checkPermission`方法，来确定是否可以替换这个安全管理器。如果没有，则部署，如果参数为`null`，方法就不会进行操作并返回。

     
     public static
     void setSecurityManager(final SecurityManager s) {
         try {
             s.checkPackageAccess("java.lang");
         } catch (Exception e) {
         }
         setSecurityManager0(s);
     }
    
     private static synchronized
     void setSecurityManager0(final SecurityManager s) {
         SecurityManager sm = getSecurityManager();
         if (sm != null) {
             //已经部署了一个安全管理器，查看是否能替换。
             sm.checkPermission(new RuntimePermission
                                      ("setSecurityManager"));
         }
    
         if ((s != null) && (s.getClass().getClassLoader() != null)) {
         	//新的安全管理器类不在bootstrap类路径里。
         	//造成我们在安装新的安全管理器之前要初始化策略，
         	//为了防止初始化策略的死循环（策略通常会包含获取安全属性（或/和）系统属性，
         	//而系统属性反过来又会调用部署的安全管理器的checkPermission方法，
         	//如果在堆中只有非系统类（这里指新的安全管理器类）就会造成死循环。）
             AccessController.doPrivileged(new PrivilegedAction<Object>() {
                 public Object run() {
                     s.getClass().getProtectionDomain().implies
                         (SecurityConstants.ALL_PERMISSION);
                     return null;
                 }
             });
         }
    
         security = s;
     }
     

### native方法，返回毫秒

     
     public static native long currentTimeMillis();
     

### native方法，返回纳秒

     
     public static native long nanoTime();
     

### 拷贝数组，Arrays.copyOf方法就是调用了这个方法。

     
     //src 原数组，srcPos原数组起始位，desc目标数组，destPost目标数组起始位，length复制长度
     public static native void arraycopy(Object src,  int  srcPos,
                                         Object dest, int destPos,
                                         int length);
     

### 返回object的hash值，与object自身的hashCode()产生的hash值相等。

     
     public static native int identityHashCode(Object x);
     

### property相关方法。

     
     private static native Properties initProperties(Properties props);
     public static Properties getProperties() {
             SecurityManager sm = getSecurityManager();
             if (sm != null) {
                 sm.checkPropertiesAccess();
             }
    
             return props;
         }
     public static void setProperties(Properties props) {
       SecurityManager sm = getSecurityManager();
         if (sm != null) {
         	sm.checkPropertiesAccess();
         }
         if (props == null) {
         	props = new Properties();
         	initProperties(props);
         }
       	System.props = props;
       }
     //获取单个属性值
     public static String getProperty(String key) {
       checkKey(key);
       SecurityManager sm = getSecurityManager();
       if (sm != null) {
       	sm.checkPropertyAccess(key);
       }
    
        return props.getProperty(key);
     }
     //获取属性值，如果null,则返回传入的默认值
     public static String getProperty(String key, String def) {
     	checkKey(key);
     	SecurityManager sm = getSecurityManager();
     	if (sm != null) {
     	sm.checkPropertyAccess(key);
     }
    
     return props.getProperty(key, def);
         }
         
     //设置属性值，返回的是原先的属性值
     public static String setProperty(String key, String value) {
         checkKey(key);
         SecurityManager sm = getSecurityManager();
         if (sm != null) {
         	sm.checkPermission(new PropertyPermission(key,
         	SecurityConstants.PROPERTY_WRITE_ACTION));
         }
    
     	return (String) props.setProperty(key, value);
     }
     //移除指定属性，返回的是移除的属性值
     public static String clearProperty(String key) {
         checkKey(key);
         SecurityManager sm = getSecurityManager();
         if (sm != null) {
         	sm.checkPermission(new PropertyPermission(key, "write"));
         }
    
     	return (String) props.remove(key);
     }
     //不能为null或空字符串	
     private static void checkKey(String key) {
         if (key == null) {
         	throw new NullPointerException("key can't be null");
         }
         if (key.equals("")) {
         	throw new IllegalArgumentException("key can't be empty");
         }
     }
     

### 获取环境变量

     
     public static String getenv(String name) {
         SecurityManager sm = getSecurityManager();
         if (sm != null) {
             sm.checkPermission(new RuntimePermission("getenv."+name));
         }
    
         return ProcessEnvironment.getenv(name);
     }
     public static java.util.Map<String,String> getenv() {
     	SecurityManager sm = getSecurityManager();
     	if (sm != null) {
     		sm.checkPermission(new RuntimePermission("getenv.*"));
     	}
     	return ProcessEnvironment.getenv();
     }
     

### 终止当前JVM  。非0`status`表示非正常终止。

     
     public static void exit(int status) {
         Runtime.getRuntime().exit(status);
     }
     

### 执行垃圾回收。

     
     public static void gc() {
         Runtime.getRuntime().gc();
     }
     

### 执行Finalization方法

     
     public static void runFinalization() {
         Runtime.getRuntime().runFinalization();
     }
     

### 通过文件名加载native library。文件名中的路径必须为绝对路径。

     
     @CallerSensitive
     public static void load(String filename) {
         Runtime.getRuntime().load0(Reflection.getCallerClass(), filename);
     }
     

### 通过`libname`加载native library.

     
     @CallerSensitive
     public static void loadLibrary(String libname) {
         Runtime.getRuntime().loadLibrary0(Reflection.getCallerClass(), libname);
     }
     

### 以字符串类型返回`libname`映射的本地库文件

     
     public static native String mapLibraryName(String libname);
     

### 在当前编码下，创建一个标准输出或标准错误的`PrintStream`

     
     private static PrintStream newPrintStream(FileOutputStream fos, String enc) {
        if (enc != null) {
             try {
                 return new PrintStream(new BufferedOutputStream(fos, 128), true, enc);
             } catch (UnsupportedEncodingException uee) {}
         }
         return new PrintStream(new BufferedOutputStream(fos, 128), true);
     }
     

### 初始化系统类，在线程初始化完成以后调用。

     
     private static void initializeSystemClass() {
     	//VM可能会在"props"初始化的过程中调用JNU_NewStringPlatform()方法
     	//来设置与编码有关的属性(user.home, user.name, boot.class.path, etc.)
     	//初始过程可能会需要调用System.getProperty()来获取先前已经生成的（放入'props'中）相关系统编码属性来获取某些许可。
     	//所以要保证"props"在最开始初始化的时候是有效的，并且所有的系统属性都直接放入"props"中。
         props = new Properties();
         initProperties(props);  // 由VM初始化
    
         //VM options可能会控制一些特定的系统配置，比如直接内存的最大容量和用来支持对象自动装箱的Integer cache.通常，库通常会在VM设置的属性中获取这些值。
         //如果属性只是内部使用，那就应该在系统属性中移除这些属性。
         //可以参考java.lang.Integer.IntegerCache和sun.misc.VM.saveAndRemoveProperties方法
         //保存一个只能由内部接口获取的系统属性对象的私有拷贝。移除不是用来公开获取的系统属性
         sun.misc.VM.saveAndRemoveProperties(props);
    
         lineSeparator = props.getProperty("line.separator");
         sun.misc.Version.init();
      
         FileInputStream fdIn = new FileInputStream(FileDescriptor.in);
         FileOutputStream fdOut = new FileOutputStream(FileDescriptor.out);
         FileOutputStream fdErr = new FileOutputStream(FileDescriptor.err);
         setIn0(new BufferedInputStream(fdIn));
         setOut0(newPrintStream(fdOut, props.getProperty("sun.stdout.encoding")));
         setErr0(newPrintStream(fdErr, props.getProperty("sun.stderr.encoding")));
      
     	//载入zip库，使java.util.zip.ZipFile之后不需要自己来加载
         loadLibrary("zip");
      
         // 为HUP, TERM, and INT (如果存在).配置Java信号
         Terminator.setup();
      
         //初始化类库需要的各种操作系统配置。
         //目前这个操作不会起到作用，除了在windows中，java.io类使用前就会设置进程范围错误模式。
         sun.misc.VM.initializeOSEnvironment();
      
     	//主线程还没有像其他线程一样被添加到到主线程的线程组中；这里我们手动来添加。
         Thread current = Thread.currentThread();
         current.getThreadGroup().add(current);
      
         //注册共享密钥
         setJavaLangAccess();
      
         //子系统在初始化过程中可以调用sun.misc.VM.isBooted()来避免执行一些应该等到应用class loader设置完成前的动作。
         //重要：保证这个方法在初始化动作的最后再调用
         sun.misc.VM.booted();
     }
     

### //

     
     private static void setJavaLangAccess() {
         //允许java.lang之外经过授权的类
         sun.misc.SharedSecrets.setJavaLangAccess(new sun.misc.JavaLangAccess(){
             public sun.reflect.ConstantPool getConstantPool(Class<?> klass) {
                 return klass.getConstantPool();
             }
             public boolean casAnnotationType(Class<?> klass, AnnotationType oldType, AnnotationType newType) {
                 return klass.casAnnotationType(oldType, newType);
             }
             public AnnotationType getAnnotationType(Class<?> klass) {
                 return klass.getAnnotationType();
             }
             public Map<Class<? extends Annotation>, Annotation> getDeclaredAnnotationMap(Class<?> klass) {
                 return klass.getDeclaredAnnotationMap();
             }
             public byte[] getRawClassAnnotations(Class<?> klass) {
                 return klass.getRawAnnotations();
             }
             public byte[] getRawClassTypeAnnotations(Class<?> klass) {
                 return klass.getRawTypeAnnotations();
             }
             public byte[] getRawExecutableTypeAnnotations(Executable executable) {
                 return Class.getExecutableTypeAnnotationBytes(executable);
             }
             public <E extends Enum<E>>
                     E[] getEnumConstantsShared(Class<E> klass) {
                 return klass.getEnumConstantsShared();
             }
             public void blockedOn(Thread t, Interruptible b) {
                 t.blockedOn(b);
             }
             public void registerShutdownHook(int slot, boolean registerShutdownInProgress, Runnable hook) {
                 Shutdown.add(slot, registerShutdownInProgress, hook);
             }
             public int getStackTraceDepth(Throwable t) {
                 return t.getStackTraceDepth();
             }
             public StackTraceElement getStackTraceElement(Throwable t, int i) {
                 return t.getStackTraceElement(i);
             }
             public String newStringUnsafe(char[] chars) {
                 return new String(chars, true);
             }
             public Thread newThreadWithAcc(Runnable target, AccessControlContext acc) {
                 return new Thread(target, acc);
             }
             public void invokeFinalize(Object o) throws Throwable {
                 o.finalize();
             }
         });
     }
     