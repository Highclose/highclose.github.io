---
layout: post
category: java
title:  "Source code of URL in java."
comments: true
description: "Analyze source code of URL."
tags: [java,source]
---

## 说明

统一资源定位符，是万维网上资源的指针。资源可以是文件或目录这种简单的对象，也可以是对数据库查询或搜索引擎的复杂对象。

通常来说，一个URL可以分成很多部分。比如`http://www.example.com/docs/resource1.html`，协议是http，信息是在一个主机`www.example.com`上，主机上的资源名字是`/docs/resource1.html`。这个名字的意思是与协议相关，与主机相关。这些信息通常在一个文件之中，也可以动态生成。这个组件叫做路径组件。

URL可选择的指定端口。如果没指定则使用默认的。后面也可以跟一个分区，跟在`#`之后。分区技术上来说不是URL的一部分，它在资源已经获取到之后才标识，指定了文档中感兴趣的部分。

也可以指定相对URL，只包含了可以根据一个基础URL获取资源的足够的信息。相对URL不需要指定所有的组件，如果协议，主机，端口没有指定，可以从基础URL中继承，文件组件必须指定，分区不继承。

URL类自身对组件不进行编码于解码。编码功能是调用者的责任，先进行编码转义再调用URL，从URL返回是先解码转义功能。因此，URL不承认URL转义，所以不认为同一个URL编码解码前后的URL是相等的，比如`http://foo.com/hello world/` `http://foo.com/hello%20world`被认为是不想等的。

管理编码于解码功能的方式建议使用`java.net.URI`。

`URLEncoder` `URLDecoder`也可以使用，但是只支持HTML形式的编码，于RFC2396描述的模式编码不同。

## implements

```
implements java.io.Serializable
```

## fields

```
static final String BUILTIN_HANDLERS_PREFIX = "sun.net.www.protocol";
static final long serialVersionUID = -7627629688361524110L;

// 本属性是用来指定扫描协议管理器的包前缀列表。本类所有的协议管理器都会在一个<协议名>.Handler名字的类中，依次在列表中包中检测匹配的管理器。如果没有找到的（或这个属性没有指定），会使用默认的包前缀（sun.net.www.protocol）。搜索动作从第一个列表的第一个包到最后一个直到有匹配的被找到。
private static final String protocolPathProp = "java.protocol.handler.pkgs";

// 协议
private String protocol;

// 主机
private String host;

// 端口
private int port = -1;

// 主机上的文件名。定义为路径[?查询]
private String file;

// 查询
private transient String query;

// 权限
private String authority;

// 路径
private transient String path;

// 用户信息
private transient String userInfo;

// 引用
private String ref;

// 主机地址，在判断相等和哈希码中使用。
transient InetAddress hostAddress;

// 本URL的URL流管理器
transient URLStreamHandler handler;

// 哈希码
private int hashCode = -1;

private transient UrlDeserializedState tempState;
```

## methods

### 构造器

```
// 创建一个URL实例。host可以是一个主机名也可以是一个IP地址。如果使用IPv6地址，应该用中括号括起来。
// port为端口号，如果值为-1则表示需要使用协议默认的端口号。
// 如果这是用指定的协议创建的第一个实例，会为协议创建一个URLStreamHandler类的流协议管理器对象：如果应用前面已经建立了URLStreamHandlerFactory实例作为工厂方法，则会用协议字符串作为参数调用createURLStreamHandler方法来创建流协议管理器。如果没有创建URLStreamHandlerFactory，或工厂的createURLStreamHandler方法返回null，则构造器找到系统属性值：ava.protocol.handler.pkgs，构造器尝试加载类名为<包>.<协议>.Handler.
// 一些的协议管理器在路径中保证存在：http, https, file, and jar
//
public URL(String protocol, String host, int port, String file)
    throws MalformedURLException
{
    this(protocol, host, port, file, null);
}

public URL(String protocol, String host, String file)
        throws MalformedURLException {
    this(protocol, host, -1, file);
}

public URL(String protocol, String host, int port, String file,
           URLStreamHandler handler) throws MalformedURLException {
    if (handler != null) {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            // 是否有权限指定管理器
            checkSpecifyHandler(sm);
        }
    }

    protocol = protocol.toLowerCase();
    this.protocol = protocol;
    if (host != null) {

        // 如果是ipv6地址，则使它符合RFC2732 
        if (host.indexOf(':') >= 0 && !host.startsWith("[")) {
            host = "["+host+"]";
        }
        this.host = host;

        if (port < -1) {
            throw new MalformedURLException("Invalid port number :" +
                                                port);
        }
        this.port = port;
        authority = (port == -1) ? host : host + ":" + port;
    }

    Parts parts = new Parts(file);
    path = parts.getPath();
    query = parts.getQuery();

    if (query != null) {
        this.file = path + "?" + query;
    } else {
        this.file = path;
    }
    ref = parts.getRef();
 
    if (handler == null &&
        (handler = getURLStreamHandler(protocol)) == null) {
        throw new MalformedURLException("unknown protocol: " + protocol);
    }
    this.handler = handler;
}

public URL(String spec) throws MalformedURLException {
    this(null, spec);
}

// 用指定的上下文解析传入的字符串来创建URL实例。新的URL的建立方法如 <scheme>://<authority><path>?<query>#<fragment>
// 如果spec与context里的模式都定义了且不相同，则新的URL基于spec构建为绝对URL。否则，模式从context继承。
// 如果spec中有权限组件，则spec则作为绝对的，spec的权限和路径会替换context的权限和路径，如果没有权限组件，则从context继承。
// 如果spec的路径组件是/开头，则路径为绝对的，替换context路径。否则，为相对路径，添加到context路径后，在这个情况下，也会移除'.' '..'
//
public URL(URL context, String spec) throws MalformedURLException {
    this(context, spec, null);
}

public URL(URL context, String spec, URLStreamHandler handler)
    throws MalformedURLException
{
    String original = spec;
    int i, limit, c;
    int start = 0;
    String newProtocol = null;
    boolean aRef=false;
    boolean isRelative = false;

    // 检查权限
    if (handler != null) {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkSpecifyHandler(sm);
        }
    }

    try {
        limit = spec.length();
        while ((limit > 0) && (spec.charAt(limit - 1) <= ' ')) {
            limit--;        // 移除末尾的空格
        }
        while ((start < limit) && (spec.charAt(start) <= ' ')) {
            start++;        // 移除开头的空格
        }

        if (spec.regionMatches(true, start, "url:", 0, 4)) {
            start += 4; // 移除开头的“url:”
        }
        if (start < spec.length() && spec.charAt(start) == '#') {
            // 假定是相对于context的一个引用。这意味着协议不能用'#'打头，
            aRef=true;
        }
        // 模式
        for (i = start ; !aRef && (i < limit) &&
                 ((c = spec.charAt(i)) != '/') ; i++) {
            if (c == ':') {

                String s = spec.substring(start, i).toLowerCase();
                if (isValidProtocol(s)) {
                    newProtocol = s;
                    start = i + 1;
                }
                break;
            }
        }

        // spec context模式匹配或者spec没有模式
        protocol = newProtocol;
        if ((context != null) && ((newProtocol == null) ||
                        newProtocol.equalsIgnoreCase(context.protocol))) {
            // 没有指定管理器则从context继承
            if (handler == null) {
                handler = context.handler;
            }

            // 如果context的模式与spec的模式匹配，则为了方便起见，当作spec没有模式。
            if (context.path != null && context.path.startsWith("/"))
                newProtocol = null;

            if (newProtocol == null) {
                protocol = context.protocol;
                authority = context.authority;
                userInfo = context.userInfo;
                host = context.host;
                port = context.port;
                file = context.file;
                path = context.path;
                isRelative = true;
            }
        }

        if (protocol == null) {
            throw new MalformedURLException("no protocol: "+original);
        }

        // 获取协议管理器
        if (handler == null &&
            (handler = getURLStreamHandler(protocol)) == null) {
            throw new MalformedURLException("unknown protocol: "+protocol);
        }

        this.handler = handler;

        i = spec.indexOf('#', start);
        if (i >= 0) {
            ref = spec.substring(i + 1, limit);
            limit = i;
        }

        // 处理RFC2396 5.2.2部分暗示的继承查询和分区的特殊情况 
        if (isRelative && start == limit) {
            query = context.query;
            if (ref == null) {
                ref = context.ref;
            }
        }
		
		//处理模式与分区之间的部分
        handler.parseURL(this, spec, start, limit);

    } catch(MalformedURLException e) {
        throw e;
    } catch(Exception e) {
        MalformedURLException exception = new MalformedURLException(e.getMessage());
        exception.initCause(e);
        throw exception;
    }
}
```

### 字符串是否是有效的协议名

```
private boolean isValidProtocol(String protocol) {
    int len = protocol.length();
    if (len < 1)
        return false;
    char c = protocol.charAt(0);
    if (!Character.isLetter(c))
        return false;
    for (int i = 1; i < len; i++) {
        c = protocol.charAt(i);
        if (!Character.isLetterOrDigit(c) && c != '.' && c != '+' &&
            c != '-') {
            return false;
        }
    }
    return true;
}
```

### 检查权限

```
private void checkSpecifyHandler(SecurityManager sm) {
    sm.checkPermission(SecurityConstants.SPECIFY_HANDLER_PERMISSION);
}
```

### 设置URL的属性。这都是非public方法，因此只有`URLStreamHandlers`可以修改URL的属性。

```
void set(String protocol, String host, int port,
         String file, String ref) {
    synchronized (this) {
        this.protocol = protocol;
        this.host = host;
        authority = port == -1 ? host : host + ":" + port;
        this.port = port;
        this.file = file;
        this.ref = ref;
        //非常重要，修改过变量后要重新计算哈希值 
        hashCode = -1;
        hostAddress = null;
        int q = file.lastIndexOf('?');
        if (q != -1) {
            query = file.substring(q+1);
            path = file.substring(0, q);
        } else
            path = file;
    }
}

void set(String protocol, String host, int port,
         String authority, String userInfo, String path,
         String query, String ref) {
    synchronized (this) {
        this.protocol = protocol;
        this.host = host;
        this.port = port;
        this.file = query == null ? path : path + "?" + query;
        this.userInfo = userInfo;
        this.path = path;
        this.ref = ref;
        //非常重要，修改过变量后要重新计算哈希值 
        hashCode = -1;
        hostAddress = null;
        this.query = query;
        this.authority = authority;
    }
}
```

### 获取各属性值

```
public String getQuery() {
    return query;
}

public String getPath() {
    return path;
}

public String getUserInfo() {
    return userInfo;
}

public String getAuthority() {
    return authority;
}

public int getPort() {
    return port;
}

public int getDefaultPort() {
    return handler.getDefaultPort();
}

public String getProtocol() {
    return protocol;
}

public String getHost() {
    return host;
}

public String getFile() {
    return file;
}

public String getRef() {
    return ref;
}
```

### 判断相等与哈希计算

```
public boolean equals(Object obj) {
    if (!(obj instanceof URL))
        return false;
    URL u2 = (URL)obj;

    return handler.equals(this, u2);
}

public synchronized int hashCode() {
    if (hashCode != -1)
        return hashCode;

    hashCode = handler.hashCode(this);
    return hashCode;
}
```

### 比较两个URL（不包括分区部分）

```
public boolean sameFile(URL other) {
    return handler.sameFile(this, other);
}
```

### 转字符串

```
public String toString() {
    return toExternalForm();
}

public String toExternalForm() {
    return handler.toExternalForm(this);
}
```

### 转成URI

```
public URI toURI() throws URISyntaxException {
    return new URI (toString());
}
```

### 返回代表URL指向的远程对象的连接的`java.net.URLConnection`实例

```
// 每次调用本方法都会创建一个新的java.net.URLConnection实例
// java.net.URLConnection实例创建时不会建立实际的网络连接。只有在调用java.net.URLConnection.connect()时才会连接。
// 如果URL的协议（比如HTTP或者JAR）有共有的，特定的URLConnection子类且位于java.lang, java.io, java.util, java.net之下，则返回该子类，比如HTTP返回HttpURLConnection， JAR返回JarURLConnection
//
public URLConnection openConnection() throws java.io.IOException {
    return handler.openConnection(this);
}

// 与上个方法基本相同，只是用了代理服务器。不支持代理的协议管理器会忽略该代理并做一个普通的连接。
//
public URLConnection openConnection(Proxy proxy)
        throws java.io.IOException {
        if (proxy == null) {
            throw new IllegalArgumentException("proxy can not be null");
        }

        // 创建一个代理服务器的备份用作安全测试
        Proxy p = proxy == Proxy.NO_PROXY ? Proxy.NO_PROXY : sun.net.ApplicationProxy.create(proxy);
        SecurityManager sm = System.getSecurityManager();
        if (p.type() != Proxy.Type.DIRECT && sm != null) {
            InetSocketAddress epoint = (InetSocketAddress) p.address();
            if (epoint.isUnresolved())
                sm.checkConnect(epoint.getHostName(), epoint.getPort());
            else
                sm.checkConnect(epoint.getAddress().getHostAddress(),
                                epoint.getPort());
        }
        return handler.openConnection(this, p);
    }
```

### 打开本URL的连接并返回一个`InputStream`用做从通道读取信息。

```
public final InputStream openStream() throws java.io.IOException {
    return openConnection().getInputStream();
}
```

### 获取URL的内容

```
public final Object getContent() throws java.io.IOException {
    return openConnection().getContent();
}
```