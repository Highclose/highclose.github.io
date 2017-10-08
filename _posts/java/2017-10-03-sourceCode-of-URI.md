---
layout: post
category: java
title:  "Source code of URI in java."
comments: true
description: "Analyze source code of URI."
tags: [java,source]
---

## 说明

统一资源标识符（URI）

除了以下注明的一些小的偏离，本类的实例表示了一个URI。URI在[RFC 2396: Uniform Resource Identifiers (URI): Generic Syntax](http://www.ietf.org/rfc/rfc2396.txt)中定义，在[RFC&nbsp;2732: Format for  Literal IPv6 Addresses in URLs](http://www.ietf.org/rfc/rfc2732.txt)修补，后者也支持`scope_ids`。`scope_ids`的语法和使用在[这里]("Inet6Address.html#scoped")描述。本类提供了由他们的要素或者分析字符串创造URI实例的构造器，获取实例各种要素的方法。本类的实例是不可变的。

### URI的语法和要素

* [模式:]模式具体部分[#片段]

一个绝对URI指定一个模式；非绝对URI则为相对URI。URI也可以依据是否是分层来区分。

不分层的URI是一个绝对的URI,模式具体部分不是`/`字符开头。不分层的URI不用来做进一步的解析。
| 例子                           |
| :--------------------------- |
| mailto:java-net@java.sun.com |
| news:comp.lang.java          |
| urn:isbn:096139210x          |

一个分层的URI可以是绝对URI或是相对URI，相对URI不指明模式
| 例子                                       |
| :--------------------------------------- |
| http://java.sun.com/j2se/1.3/            |
| docs/guide/collections/designfaq.html#28 |
| ../../../demo/jfc/SwingSet2/src/SwingSet2.java |
| file:///~/calendar                       |

一个分层URI根据语法可以做进一步的解析。

* \[模式:\]\[//权限\]\[路径\]\[?查询\]\[#分区\]

模式和路径中的组件组成了分层URI的模式具体部分。

权限组件是基于服务器或者基于注册的，一个基于服务器的权限解析依据类似的语法

\[用户信息@\]主机\[:端口\]

现在在使用的几乎所URI都是基于服务器的。不是按这种方式解析的权限部分都是基于注册的。

路径组件如果是`/`开头则是绝对路径，否则是相对的。如果指定权限则一定是绝对路径。

一个URI实例包含了以下九个组件：

| 组件     | 组件类型   |
| ------ | ------ |
| 模式     | String |
| 模式具体部分 | String |
| 权限     | String |
| 用户信息   | String |
| 主机     | String |
| 端口     | int    |
| 路径     | String |
| 查询     | String |
| 分区     | String |

一个实例中，任意一个组件都是未定义或者明确定义的。未定义的字符串组件由`null`表示，数字类型则由`-1`表示。字符串组件可以定义成空字符串。组件是否未定义取决于实例表示的URI类型。一个非分层URI只有模式，模式具体部分，可能有分区，但是没有其他组件。一个分层URI一定有路径（可能未空字符串）或者可能有其他组件。如果权限组件存在并且是基于服务器的，则主机组件会被定义，用户信息和端口组件有可能会被定义。

### URI实例的操作

本类支持的关键操作包括标准化，解析，相对化。

标准化的过程包括从分层URI的路径组件中移除不需要的`.`,`..`部分，每一个`.`部分被简单的移除，`..`部分只有在不是`..`部分的后面时才会移除。标准化对非分层URI没有影响。

解析的过程包括使用另一个基础URI来解析一个URI。生成的URI由RFC 2396指定的组件构成，哪些原始URI中没有的组件则从基础URI中提取。对分层URI来说，原始URI的路径组件会用基础URI的路径来解析并标准化。比如要解析`docs/guide/collections/designfaq.html#28`，基础URI为`http://java.sun.com/j2se/1.3/`，则生成的URI为`http://java.sun.com/j2se/1.3/docs/guide/collections/designfaq.html#28`。

解析相对URI

解析`../../../demo/jfc/SwingSet2/src/SwingSet2.java`,基础路径为上一个生成URI的路径，则生成的URI为`http://java.sun.com/j2se/1.3/demo/jfc/SwingSet2/src/SwingSet2.java`

用其他URI来解析`file:///~calendar`路径，则会返回原路径，因为原路径时绝对路径。用前倒数第二个相对路径(已经标准化)来解析前倒数第一个相对路径。得到的也是相对路径：`demo/jfc/SwingSet2/src/SwingSet2.java`

相对化，与解析相反。对任意两个标准化的URI `u`和`v`:

`u.relative(u.resolvize(v)).equals(v)` `u.resolve(u.relativize(v)).equals(v)`

当构建一个包含URI的文档，这些URI不管文档存放的位置相对与文档的基础URI必须是相对的，这些操作就经常使用。

比如，使用`http://java.sun.com/j2se/1.3`相对化`https://docs.oracle.com/javase/1.3/docs/guide/index.html`,得到的URI为`docs/guide/index.html`

### 标识

对于任意的URI，以下情况一定成立：`new URI(u.toString()).equals(u)`

对于任意空的权限前面有两个斜线(`file:///tmp/`)或主机主机后面跟着冒号缺没有端口(`http://java.sun.com:`)的URI，除非被引用否则这些字符不会被编码。

`new URI(u.getScheme(),u.getSchemeSpecificPart(),u.getFragment()).equals(u)`在所有情况下成立

`new URI(u.getScheme(),u.getUserInfo(),u.getAuthority(),u.getPath(),u.getQuery().u.getFragment()).equals(u)`在分层URI成立

`new URI(u.getScheme(),u.getUserInfo(),u.getHost(),u.getPort(),u.getPath(),u.getQuery(),u.getFragment()).equals(u)`在分层URI中，没有权限或者基于服务器。

### URI, URL, URN

URI是统一资源标识符， URL是统一资源定位符，因此每个URL都是URI，不是每一个URI都是URL。这是因为URI的另一个子集，统一资源名称URN，URN命名资源但不指定如何定位资源。上面提到的`mailto` `news` `isbn`都是URN。

概念上URI与URL的不同体现在本类和`URL`类的不同。

URI实例既可以是绝对的也可以是相对的。一个URI字符串根据通用的语法来解析，与模式无关。不查找主机，不建立模式相关的流管理器。相等，哈希，比较根据实例的字符内容来操作。换句话说，一个URI实例只是一个比字符串多了一些支持语法，模式无关的比较，标准化，解析，相对化操作功能的实例。

URL实例一定是绝对的，所以一定会指定一个模式。一个URL字符串根据模式进行解析。URL会建立一个流处理器，事实上，如果处理器不可用是不可能创建URL实例的。相等与哈希取决于模式和主机地址，比较未定义。换句话说，一个URL是一个支持语法的解析操作，查找主机并打开连接到指定资源的网络操作的字符串。

## implements

```
implements Comparable<URI>, Serializable
```

## fields

### 实例的组件和属性

```
private transient String scheme;            // 相对URI为null
private transient String fragment;

private transient String authority;         // 服务器或注册

// 基于服务器的权限
private transient String userInfo;
private transient String host;              // 基于注册为null
private transient int port = -1;            // -1则为未定义

// 分层URI剩余组件
private transient String path;              // 未分层未null
private transient String query;

//剩余需要计算的变量
private volatile transient String schemeSpecificPart;
private volatile transient int hash;        // 0为未定义

private volatile transient String decodedUserInfo = null;
private volatile transient String decodedAuthority = null;
private volatile transient String decodedPath = null;
private volatile transient String decodedQuery = null;
private volatile transient String decodedFragment = null;
private volatile transient String decodedSchemeSpecificPart = null;
```

### 本URI的字符串形式

```
private volatile String string;             // 序列化时唯一的变量
```

## methods

### 私有构造器

```
private URI() { }                           // 内部使用
```

### 用给定的字符串构建URI实例，调用了[RFC 2396](http://www.ietf.org/rfc/rfc2396.txt)中的语法，除了以下的变动：

一个空的权限组件允许后面跟一个非空的路径，查询，分区组件。这允许解析类似于`file:///foo/bar`的字符串。如果权限为空，则用户信息，主机，端口为未定义。

允许空的相对路径。这个变动最大的结果就是单独的分区诸如`#foo`可以解析为相对路径，可以将`<a href="#resolve-frag">Resolve</a>`用基础URI解析。

```
public URI(String str) throws URISyntaxException {
    new Parser(str).parse(false);
}
```

### 构造器，构造一个分层URI

初始时，结果字符串为空。

如果模式指定，则添加到字符串并添加一个`:`

如果有用户信息或主机或端口，则添加一个`//`

如果有用户信息，添加到字符串，并添加一个`@`

如果主机给定，添加到字符串，如果时IPV6地址，且中括号没有闭合，则闭合中括号

如果端口给定，添加`:`后跟端口

如果路径给定，添加到字符串

如果查询给定，添加`?`后跟查询

最后，如果分区给定，添加`#`后跟分区

```
public URI(String scheme,
           String userInfo, String host, int port,
           String path, String query, String fragment)
    throws URISyntaxException
{
    String s = toString(scheme, null,
                        null, userInfo, host, port,
                        path, query, fragment);
    checkPath(s, scheme, path);
    new Parser(s).parse(true);
}
```

### 构造器

```
public URI(String scheme,
           String authority,
           String path, String query, String fragment)
    throws URISyntaxException
{
    String s = toString(scheme, null,
                        authority, null, null, -1,
                        path, query, fragment);
    checkPath(s, scheme, path);
    new Parser(s).parse(false);
}
```

### 构造器

```
public URI(String scheme, String host, String path, String fragment)
    throws URISyntaxException
{
    this(scheme, null, host, -1, path, null, fragment);
}
```

### 构造器，`ssp`为模式具体部分

```
public URI(String scheme, String ssp, String fragment)
    throws URISyntaxException
{
    new Parser(toString(scheme, ssp,
                        null, null, null, -1,
                        null, null, fragment))
        .parse(false);
}
```

### 创建URI实例，这是要给便捷的工厂方法。就好像直接调用了构造器。这个方法在确定字符串为合法的URI时使用，比如在程序中申明的一个字符串常量，如果没有像预期解析可能是程序问题。而直接抛出`URISyntaxException`的构造器，应该在从用户输入或其他来源获取的字符串构造时使用。

```
public static URI create(String str) {
    try {
        return new URI(str);
    } catch (URISyntaxException x) {
        throw new IllegalArgumentException(x.getMessage(), x);
    }
}
```

### 试图将URI的权限组件解析成用户信息，主机，端口。

提供这个方法是因为RFC 2396指定的语法无法区分出是基于服务器还是基于注册的，因此可能会造成混淆。比如`//foo:bar`这个权限组件，是基于注册的。在当前普遍的场合看来，使用的URI或者是URL或者是URN，使用中的URI都是基于服务器的。调用这个方法是确定字符串可以解析成基于服务器的，否则抛出异常。这些时候可以如下调用：`URI u = new URI(str).parseServerAuthority()`

```
public URI parseServerAuthority()
    throws URISyntaxException
{
    if ((host != null) || (authority == null))
        return this;
    defineString();
    new Parser(string).parse(true);
    return this;
}
```

### 标准化URI的路径组件

如果URI是非分层的或者路径已经标准化，则返回该URI，否则构造一个新的URI,其路径被标准化，标准化方法如下：

所有的`.`部分被移除

如果一个非`..`的部分后跟一个`..`部分，则这两个部分都被移除，这个部分会循环直到没有匹配的。

如果路径是相对的，且该组件第一个部分包含`:`，则在最前面添加一个`.`避免一个诸如`a:b/c/d`的相对URI解析成一个`a`为模式`b/c/d`为模式具体部分的非分区URI

一个标准化的URI路径组件如果前面没有足够多的非`..`部分，则是从一个或多个`..`开始的，如果经过前一个步骤，则是从`.`开始的，其他情况，一个标准化的路径不包含`.` `..`

```
public URI normalize() {
    return normalize(this);
}
```

### 用本URI解析传入的URI

如果传入的URI已经是绝对的，或者如果是非分层的，则返回传入的URI

如果传入的URI只有一个分区组件，则返回的URI的除分区其他组件与本组件相同

其他情况，本方法遵守RFC 2396的要求构造一个新的分层URI，步骤如下：

* 新的URI使用本URI的模式组件与传入的URI的查询和分区组件
* 如果传入的URI有权限组件，新的URI使用传入的URI的权限和路径组件
* 否则使用本URI的权限组件，路径组件如下计算：
  * 如果传入的URI的路径组件是绝对的，则使用传入的
  * 否则使用本URI的路径来解析传入的URI的路径

本方法的结果只有在本URI或传入的URI是绝对的时候才是绝对的

```
public URI resolve(URI uri) {
    return resolve(this, uri);
}
```

### 与上相同

```
public URI resolve(String str) {
    return resolve(URI.create(str));
}
```

### 用本URI相对化传入的URI

相对化过程：

* 如果本URI或传入的URI其中一个是非分层的，或者两个URI的模式和权限组件没有标识，或者本URI的路径不是传入URI路径的前缀，则返回传入的URI
* 否则，生成的URI使用传入URI的查询与分区组件并使用从传入URI路径开头移除本URI路径的结果路径。

```
public URI relativize(URI uri) {
    return relativize(this, uri);
}
```

### 转成URL

```
public URL toURL()
    throws MalformedURLException {
    if (!isAbsolute())
        throw new IllegalArgumentException("URI is not absolute");
    return new URL(toString());
}
```

### 组件操作

```
//获取模式组件
public String getScheme() {
    return scheme;
}

//是否绝对
public boolean isAbsolute() {
    return scheme != null;
}

//是否分层
public boolean isOpaque() {
    return path == null;
}

//返回原始模式指定部分，该部分不会是未定义的，不过可能为空 
public String getRawSchemeSpecificPart() {
    defineSchemeSpecificPart();
    return schemeSpecificPart;
}

//返回已解码的模式指定部分 
public String getSchemeSpecificPart() {
    if (decodedSchemeSpecificPart == null)
        decodedSchemeSpecificPart = decode(getRawSchemeSpecificPart());
    return decodedSchemeSpecificPart;
}

//获取原始权限组件
public String getRawAuthority() {
    return authority;
}

//获取已解码的权限组件 
public String getAuthority() {
    if (decodedAuthority == null)
        decodedAuthority = decode(authority);
    return decodedAuthority;
}

public String getRawUserInfo() {
    return userInfo;
}

public String getUserInfo() {
    if ((decodedUserInfo == null) && (userInfo != null))
        decodedUserInfo = decode(userInfo);
    return decodedUserInfo;
}

public String getHost() {
    return host;
}

public int getPort() {
    return port;
}

public String getRawPath() {
    return path;
}

public String getPath() {
    if ((decodedPath == null) && (path != null))
        decodedPath = decode(path);
    return decodedPath;
}

public String getRawQuery() {
    return query;
}

public String getQuery() {
    if ((decodedQuery == null) && (query != null))
        decodedQuery = decode(query);
    return decodedQuery;
}

public String getRawFragment() {
    return fragment;
}

public String getFragment() {
    if ((decodedFragment == null) && (fragment != null))
        decodedFragment = decode(fragment);
    return decodedFragment;
}
```

### 相等，比较，哈希，转字符串，序列化

```
public boolean equals(Object ob) {
    if (ob == this)
        return true;
    if (!(ob instanceof URI))
        return false;
    URI that = (URI)ob;
    if (this.isOpaque() != that.isOpaque()) return false;
    if (!equalIgnoringCase(this.scheme, that.scheme)) return false;
    if (!equal(this.fragment, that.fragment)) return false;

    // 非分层
    if (this.isOpaque())
        return equal(this.schemeSpecificPart, that.schemeSpecificPart);

    // 分层
    if (!equal(this.path, that.path)) return false;
    if (!equal(this.query, that.query)) return false;

    // 权限
    if (this.authority == that.authority) return true;
    if (this.host != null) {
        // Server-based
        if (!equal(this.userInfo, that.userInfo)) return false;
        if (!equalIgnoringCase(this.host, that.host)) return false;
        if (this.port != that.port) return false;
    } else if (this.authority != null) {
        // 基于注册
        if (!equal(this.authority, that.authority)) return false;
    } else if (this.authority != that.authority) {
        return false;
    }

    return true;
}

public int hashCode() {
    if (hash != 0)
        return hash;
    int h = hashIgnoringCase(0, scheme);
    h = hash(h, fragment);
    if (isOpaque()) {
        h = hash(h, schemeSpecificPart);
    } else {
        h = hash(h, path);
        h = hash(h, query);
        if (host != null) {
            h = hash(h, userInfo);
            h = hashIgnoringCase(h, host);
            h += 1949 * port;
        } else {
            h = hash(h, authority);
        }
    }
    hash = h;
    return h;
}

public int compareTo(URI that) {
    int c;

    if ((c = compareIgnoringCase(this.scheme, that.scheme)) != 0)
        return c;

    if (this.isOpaque()) {
        if (that.isOpaque()) {
            // Both opaque
            if ((c = compare(this.schemeSpecificPart,
                             that.schemeSpecificPart)) != 0)
                return c;
            return compare(this.fragment, that.fragment);
        }
        return +1;                  // 非分层大于分层
    } else if (that.isOpaque()) {
        return -1;                  // 分层小于非分层
    }

    //分层
    if ((this.host != null) && (that.host != null)) {
        // 都是基于服务器
        if ((c = compare(this.userInfo, that.userInfo)) != 0)
            return c;
        if ((c = compareIgnoringCase(this.host, that.host)) != 0)
            return c;
        if ((c = this.port - that.port) != 0)
            return c;
    } else {
        if ((c = compare(this.authority, that.authority)) != 0) return c;
    }

    if ((c = compare(this.path, that.path)) != 0) return c;
    if ((c = compare(this.query, that.query)) != 0) return c;
    return compare(this.fragment, that.fragment);
}

public String toString() {
    defineString();
    return string;
}

public String toASCIIString() {
    defineString();
    return encode(string);
}


//保存URI的内容到给定的序列流中 
private void writeObject(ObjectOutputStream os)
    throws IOException
{
    defineString();
    os.defaultWriteObject();        // Writes the string field only
}

//从序列流中重建URI 
private void readObject(ObjectInputStream is)
    throws ClassNotFoundException, IOException
{
    port = -1;                      // Argh
    is.defaultReadObject();
    try {
        new Parser(string).parse(false);
    } catch (URISyntaxException x) {
        IOException y = new InvalidObjectException("Invalid URI");
        y.initCause(x);
        throw y;
    }
}
```

### 字符串变量进行计算和哈希的工具方法

```
private static int toLower(char c) {
    if ((c >= 'A') && (c <= 'Z'))
        return c + ('a' - 'A');
    return c;
}

// US-ASCII only
private static int toUpper(char c) {
    if ((c >= 'a') && (c <= 'z'))
        return c - ('a' - 'A');
    return c;
}

private static boolean equal(String s, String t) {
    if (s == t) return true;
    if ((s != null) && (t != null)) {
        if (s.length() != t.length())
            return false;
        if (s.indexOf('%') < 0)
            return s.equals(t);
        int n = s.length();
        for (int i = 0; i < n;) {
            char c = s.charAt(i);
            char d = t.charAt(i);
            if (c != '%') {
                if (c != d)
                    return false;
                i++;
                continue;
            }
            if (d != '%')
                return false;
            i++;
            if (toLower(s.charAt(i)) != toLower(t.charAt(i)))
                return false;
            i++;
            if (toLower(s.charAt(i)) != toLower(t.charAt(i)))
                return false;
            i++;
        }
        return true;
    }
    return false;
}

private static boolean equalIgnoringCase(String s, String t) {
    if (s == t) return true;
    if ((s != null) && (t != null)) {
        int n = s.length();
        if (t.length() != n)
            return false;
        for (int i = 0; i < n; i++) {
            if (toLower(s.charAt(i)) != toLower(t.charAt(i)))
                return false;
        }
        return true;
    }
    return false;
}

private static int hash(int hash, String s) {
    if (s == null) return hash;
    return s.indexOf('%') < 0 ? hash * 127 + s.hashCode()
                              : normalizedHash(hash, s);
}


private static int normalizedHash(int hash, String s) {
    int h = 0;
    for (int index = 0; index < s.length(); index++) {
        char ch = s.charAt(index);
        h = 31 * h + ch;
        if (ch == '%') {
            /*
             * Process the next two encoded characters
             */
            for (int i = index + 1; i < index + 3; i++)
                h = 31 * h + toUpper(s.charAt(i));
            index += 2;
        }
    }
    return hash * 127 + h;
}

private static int hashIgnoringCase(int hash, String s) {
    if (s == null) return hash;
    int h = hash;
    int n = s.length();
    for (int i = 0; i < n; i++)
        h = 31 * h + toLower(s.charAt(i));
    return h;
}

private static int compare(String s, String t) {
    if (s == t) return 0;
    if (s != null) {
        if (t != null)
            return s.compareTo(t);
        else
            return +1;
    } else {
        return -1;
    }
}

private static int compareIgnoringCase(String s, String t) {
    if (s == t) return 0;
    if (s != null) {
        if (t != null) {
            int sn = s.length();
            int tn = t.length();
            int n = sn < tn ? sn : tn;
            for (int i = 0; i < n; i++) {
                int c = toLower(s.charAt(i)) - toLower(t.charAt(i));
                if (c != 0)
                    return c;
            }
            return sn - tn;
        }
        return +1;
    } else {
        return -1;
    }
}
```

### 字符串组合                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 

```
//如果模式指定，则路径如果指定必须是绝对的
private static void checkPath(String s, String scheme, String path)
    throws URISyntaxException
{
    if (scheme != null) {
        if ((path != null)
            && ((path.length() > 0) && (path.charAt(0) != '/')))
            throw new URISyntaxException(s,
                                         "Relative path in absolute URI");
    }
}

private void appendAuthority(StringBuffer sb,
                             String authority,
                             String userInfo,
                             String host,
                             int port)
{
    if (host != null) {
        sb.append("//");
        if (userInfo != null) {
            sb.append(quote(userInfo, L_USERINFO, H_USERINFO));
            sb.append('@');
        }
        boolean needBrackets = ((host.indexOf(':') >= 0)
                                && !host.startsWith("[")
                                && !host.endsWith("]"));
        if (needBrackets) sb.append('[');
        sb.append(host);
        if (needBrackets) sb.append(']');
        if (port != -1) {
            sb.append(':');
            sb.append(port);
        }
    } else if (authority != null) {
        sb.append("//");
        if (authority.startsWith("[")) {
            //可能有IPV6地址
            int end = authority.indexOf("]");
            String doquote = authority, dontquote = "";
            if (end != -1 && authority.indexOf(":") != -1) {
                //有ipv6地址
                if (end == authority.length()) {
                    dontquote = authority;
                    doquote = "";
                } else {
                    dontquote = authority.substring(0 , end + 1);
                    doquote = authority.substring(end + 1);
                }
            }
            sb.append(dontquote);
            sb.append(quote(doquote,
                        L_REG_NAME | L_SERVER,
                        H_REG_NAME | H_SERVER));
        } else {
            sb.append(quote(authority,
                        L_REG_NAME | L_SERVER,
                        H_REG_NAME | H_SERVER));
        }
    }
}

private void appendSchemeSpecificPart(StringBuffer sb,
                                      String opaquePart,
                                      String authority,
                                      String userInfo,
                                      String host,
                                      int port,
                                      String path,
                                      String query)
{
    if (opaquePart != null) {
        if (opaquePart.startsWith("//[")) {
            int end =  opaquePart.indexOf("]");
            if (end != -1 && opaquePart.indexOf(":")!=-1) {
                String doquote, dontquote;
                if (end == opaquePart.length()) {
                    dontquote = opaquePart;
                    doquote = "";
                } else {
                    dontquote = opaquePart.substring(0,end+1);
                    doquote = opaquePart.substring(end+1);
                }
                sb.append (dontquote);
                sb.append(quote(doquote, L_URIC, H_URIC));
            }
        } else {
            sb.append(quote(opaquePart, L_URIC, H_URIC));
        }
    } else {
        appendAuthority(sb, authority, userInfo, host, port);
        if (path != null)
            sb.append(quote(path, L_PATH, H_PATH));
        if (query != null) {
            sb.append('?');
            sb.append(quote(query, L_URIC, H_URIC));
        }
    }
}

private void appendFragment(StringBuffer sb, String fragment) {
    if (fragment != null) {
        sb.append('#');
        sb.append(quote(fragment, L_URIC, H_URIC));
    }
}

private String toString(String scheme,
                        String opaquePart,
                        String authority,
                        String userInfo,
                        String host,
                        int port,
                        String path,
                        String query,
                        String fragment)
{
    StringBuffer sb = new StringBuffer();
    if (scheme != null) {
        sb.append(scheme);
        sb.append(':');
    }
    appendSchemeSpecificPart(sb, opaquePart,
                             authority, userInfo, host, port,
                             path, query);
    appendFragment(sb, fragment);
    return sb.toString();
}

private void defineSchemeSpecificPart() {
    if (schemeSpecificPart != null) return;
    StringBuffer sb = new StringBuffer();
    appendSchemeSpecificPart(sb, null, getAuthority(), getUserInfo(),
                             host, port, getPath(), getQuery());
    if (sb.length() == 0) return;
    schemeSpecificPart = sb.toString();
}

private void defineString() {
    if (string != null) return;

    StringBuffer sb = new StringBuffer();
    if (scheme != null) {
        sb.append(scheme);
        sb.append(':');
    }
    if (isOpaque()) {
        sb.append(schemeSpecificPart);
    } else {
        if (host != null) {
            sb.append("//");
            if (userInfo != null) {
                sb.append(userInfo);
                sb.append('@');
            }
            boolean needBrackets = ((host.indexOf(':') >= 0)
                                && !host.startsWith("[")
                                && !host.endsWith("]"));
            if (needBrackets) sb.append('[');
            sb.append(host);
            if (needBrackets) sb.append(']');
            if (port != -1) {
                sb.append(':');
                sb.append(port);
            }
        } else if (authority != null) {
            sb.append("//");
            sb.append(authority);
        }
        if (path != null)
            sb.append(path);
        if (query != null) {
            sb.append('?');
            sb.append(query);
        }
    }
    if (fragment != null) {
        sb.append('#');
        sb.append(fragment);
    }
    string = sb.toString();
}
```

### 标准化，解析，相对化

```
// RFC2396 5.2 (6)
private static String resolvePath(String base, String child,
                                  boolean absolute)
{
    int i = base.lastIndexOf('/');
    int cn = child.length();
    String path = "";

    if (cn == 0) {
        if (i >= 0)
            path = base.substring(0, i + 1);
    } else {
        StringBuffer sb = new StringBuffer(base.length() + cn);
        // 5.2 (6a)
        if (i >= 0)
            sb.append(base.substring(0, i + 1));
        // 5.2 (6b)
        sb.append(child);
        path = sb.toString();
    }

    // 5.2 (6c-f)
    String np = normalize(path);

    // 5.2 (6g): 如果结果是绝对的，但是路径以../开头，则保持路径原样

    return np;
}

// RFC2396 5.2
private static URI resolve(URI base, URI child) {
    //检查child是否是非分层
    if (child.isOpaque() || base.isOpaque())
        return child;

    if ((child.scheme == null) && (child.authority == null)
        && child.path.equals("") && (child.fragment != null)
        && (child.query == null)) {
        if ((base.fragment != null)
            && child.fragment.equals(base.fragment)) {
            return base;
        }
        URI ru = new URI();
        ru.scheme = base.scheme;
        ru.authority = base.authority;
        ru.userInfo = base.userInfo;
        ru.host = base.host;
        ru.port = base.port;
        ru.path = base.path;
        ru.fragment = child.fragment;
        ru.query = base.query;
        return ru;
    }

    // 5.2 (3): child是绝对的
    if (child.scheme != null)
        return child;

    URI ru = new URI();             // 解析URI
    ru.scheme = base.scheme;
    ru.query = child.query;
    ru.fragment = child.fragment;

    // 5.2 (4): 权限
    if (child.authority == null) {
        ru.authority = base.authority;
        ru.host = base.host;
        ru.userInfo = base.userInfo;
        ru.port = base.port;

        String cp = (child.path == null) ? "" : child.path;
        if ((cp.length() > 0) && (cp.charAt(0) == '/')) {
            // 5.2 (5): Child路径是绝对的
            ru.path = child.path;
        } else {
            // 5.2 (6): 解析相对路径
            ru.path = resolvePath(base.path, cp, base.isAbsolute());
        }
    } else {
        ru.authority = child.authority;
        ru.host = child.host;
        ru.userInfo = child.userInfo;
        ru.host = child.host;
        ru.port = child.port;
        ru.path = child.path;
    }

    // 5.2 (7): Recombine (nothing to do here)
    return ru;
}

//如果传入的URI路径是标准化的，则返回URI,否则，返回一个新的包含标准化路径的URI
private static URI normalize(URI u) {
    if (u.isOpaque() || (u.path == null) || (u.path.length() == 0))
        return u;

    String np = normalize(u.path);
    if (np == u.path)
        return u;

    URI v = new URI();
    v.scheme = u.scheme;
    v.fragment = u.fragment;
    v.authority = u.authority;
    v.userInfo = u.userInfo;
    v.host = u.host;
    v.port = u.port;
    v.path = np;
    v.query = u.query;
    return v;
}

private static URI relativize(URI base, URI child) {
    if (child.isOpaque() || base.isOpaque())
        return child;
    if (!equalIgnoringCase(base.scheme, child.scheme)
        || !equal(base.authority, child.authority))
        return child;

    String bp = normalize(base.path);
    String cp = normalize(child.path);
    if (!bp.equals(cp)) {
        if (!bp.endsWith("/"))
            bp = bp + "/";
        if (!cp.startsWith(bp))
            return child;
    }

    URI v = new URI();
    v.path = cp.substring(bp.length());
    v.query = child.query;
    v.fragment = child.fragment;
    return v;
}
```

### 路径标准化

```
//以下的算法避免为每一个分区创建一个字符串对象以及最后用String Buffer计算最后结果，采用了编辑一个字符数组的方式。数组首先分成个部分，每个斜杠替换成'\0'，然后创建一个保存每部分索引的数组，其中每个元素是每个对应部分的第一个字符的索引。然后我们操作两个数组，移除'.' '..' 和其他需要移除的部分然后将所以数组的入口设为-1，最后，使用两个数组组合部分并计算最终结果。

//检查传入的路径是否需要标准化，本方法使用字符串参数而不是用字符数组这样就不需要调用path.toCharArray()
static private int needsNormalization(String path) {
    boolean normal = true;
    int ns = 0;                     // 每部分的编号
    int end = path.length() - 1;    // 路径中最后一个字符的索引
    int p = 0;                      // 路径中下一个字符的索引

    //跳过初始斜杠
    while (p <= end) {
        if (path.charAt(p) != '/') break;
        p++;
    }
    if (p > 1) normal = false;

    // 扫描部分
    while (p <= end) {

        //查找'.' '..'
        if ((path.charAt(p) == '.')
            && ((p == end)
                || ((path.charAt(p + 1) == '/')
                    || ((path.charAt(p + 1) == '.')
                        && ((p + 1 == end)
                            || (path.charAt(p + 2) == '/')))))) {
            normal = false;
        }
        ns++;

        //查找下一部分的起点
        while (p <= end) {
            if (path.charAt(p++) != '/')
                continue;

            //跳过重复斜杠
            while (p <= end) {
                if (path.charAt(p) != '/') break;
                normal = false;
                p++;
            }

            break;
        }
    }

    return normal ? -1 : ns;
}


//将传入的路径分成各部分，用空替换斜杠并且填充到给定的部分索引数组
//先验条件：segs.length == 路径中部分的数量
//后验条件：
//   所有斜杠替换成'\0'
//   segs[i] == 部分中第一个字符的索引值 i (0 <= i < segs.length)
static private void split(char[] path, int[] segs) {
    int end = path.length - 1;      // 路径中最后一个字符的索引
    int p = 0;                      // 下一个字符的索引
    int i = 0;                      // 当前部分的索引

    // 跳过第一个斜杠
    while (p <= end) {
        if (path[p] != '/') break;
        path[p] = '\0';
        p++;
    }

    while (p <= end) {

        // 标识部分的开始
        segs[i++] = p++;

        // 查找下一部分的开始
        while (p <= end) {
            if (path[p++] != '/')
                continue;
            path[p - 1] = '\0';

            // 跳过重复的斜杠
            while (p <= end) {
                if (path[p] != '/') break;
                path[p++] = '\0';
            }
            break;
        }
    }

    if (i != segs.length)
        throw new InternalError();  // ASSERT
}

//用传入的部分索引数组整合传入的路径数组，索引入口为-1的部分被忽略，在需要时插入斜杠，返回结果路径的长度
//
// 先验条件:
//   segs[i] == -1 标识i部分会被忽略
//
// 后验条件:
//   path[0] .. path[返回值] == 结果路径
//
static private int join(char[] path, int[] segs) {
    int ns = segs.length;           // 部分的数量
    int end = path.length - 1;      // 最后一个字符的索引
    int p = 0;                      // 写一个写入字符的索引

    if (path[p] == '\0') {
        // 还原绝对路径的第一个斜杠
        path[p++] = '/';
    }

    for (int i = 0; i < ns; i++) {
        int q = segs[i];            // 当前部分
        if (q == -1)
            // 忽略
            continue;

        if (p == q) {
            //在部分中，跳到最后
            while ((p <= end) && (path[p] != '\0'))
                p++;
            if (p <= end) {
                // 保留反斜杠
                path[p++] = '/';
            }
        } else if (p < q) {
            // 保存q到p
            while ((q <= end) && (path[q] != '\0'))
                path[p++] = path[q++];
            if (q <= end) {
                //保留反斜杠
                path[p++] = '/';
            }
        } else
            throw new InternalError(); // ASSERT false
    }

    return p;
}


//移除'.' '..'
private static void removeDots(char[] path, int[] segs) {
    int ns = segs.length;
    int end = path.length - 1;

    for (int i = 0; i < ns; i++) {
        int dots = 0;               // 找到的.数量（0，1，2）
        // 查找下个'.' '..'
        do {
            int p = segs[i];
            if (path[p] == '.') {
                if (p == end) {
                    dots = 1;
                    break;
                } else if (path[p + 1] == '\0') {
                    dots = 1;
                    break;
                } else if ((path[p + 1] == '.')
                           && ((p + 1 == end)
                               || (path[p + 2] == '\0'))) {
                    dots = 2;
                    break;
                }
            }
            i++;
        } while (i < ns);
        if ((i > ns) || (dots == 0))
            break;

        if (dots == 1) {
            // 移除本'.'
            segs[i] = -1;
        } else {
            //如果'..'前有非'..'部分，移除两者，否则保留原样
            int j;
            for (j = i - 1; j >= 0; j--) {
                if (segs[j] != -1) break;
            }
            if (j >= 0) {
                int q = segs[j];
                if (!((path[q] == '.')
                      && (path[q + 1] == '.')
                      && (path[q + 2] == '\0'))) {
                    segs[i] = -1;
                    segs[j] = -1;
                }
            }
        }
    }
}

//如果标准化的路径是相对的，第一个部分可能会被解析成模式，在最前面加一个'.'部分
private static void maybeAddLeadingDot(char[] path, int[] segs) {

    if (path[0] == '\0')
        // 路径是绝对的
        return;

    int ns = segs.length;
    int f = 0;                      // 第一个部分的索引
    while (f < ns) {
        if (segs[f] >= 0)
            break;
        f++;
    }
    if ((f >= ns) || (f == 0))
        //路径是空的，或者原始的第一个部分被保留，则不需要最前面的'.'
        return;

    int p = segs[f];
    while ((p < path.length) && (path[p] != ':') && (path[p] != '\0')) p++;
    if (p >= path.length || path[p] == '\0')
        //第一部分没有':',所以不需要'.'
        return;

    //这个位置第一部分没有使用，因此插入'.'
    path[0] = '.';
    path[1] = '\0';
    segs[0] = 0;
}


//标准化传入的路径字符串，一个标准的路径字符串没有（会有'//'），没有部分等于'.'，没有'..'在非'..'之后，相对与UNIX形式的路径标准，URI路径保留了反斜杠。
private static String normalize(String ps) {

    //是否需要标准化
    int ns = needsNormalization(ps);        //部分的数量
    if (ns < 0)
        // 不需要，则返回
        return ps;

    char[] path = ps.toCharArray();         // 字符数组形式的路径

    // 分割路径到部分
    int[] segs = new int[ns];               // 部分索引数组
    split(path, segs);

    // 移除'.' '..'
    removeDots(path, segs);

    // 避免模式名字混淆
    maybeAddLeadingDot(path, segs);

    //连接剩余的部分并返回结果
    String s = new String(path, 0, join(path, segs));
    if (s.equals(ps)) {
        // 字符串已经被标准化
        return ps;
    }
    return s;
}
```

### 用于解析的字符类

```
// RFC2396 详细描述了在一个URI的组件中，允许使用哪些US-ASCII字符集中的字符。这里定义了一系列掩码对实现那些限制。每一对掩码对包含两个long型，一个低位字节，一个高位字节。放一起是他们代表了一个128为的掩码，当字符的i值被允许，则i位会被设置。
//这个方法比持续在数组中搜索允许的字符更加的有效。预编译这个掩码信息，一个传入的掩码可以通过一次单独的表格查询来决定字符是否合适，以此可以更加有效

// 计算传入的字符串的低位字节
private static long lowMask(String chars) {
    int n = chars.length();
    long m = 0;
    for (int i = 0; i < n; i++) {
        char c = chars.charAt(i);
        if (c < 64)
            m |= (1L << c);
    }
    return m;
}

// 计算传入的字符串的高位字节
private static long highMask(String chars) {
    int n = chars.length();
    long m = 0;
    for (int i = 0; i < n; i++) {
        char c = chars.charAt(i);
        if ((c >= 64) && (c < 128))
            m |= (1L << (c - 64));
    }
    return m;
}

//计算first,last之间的低位字节
private static long lowMask(char first, char last) {
    long m = 0;
    int f = Math.max(Math.min(first, 63), 0);
    int l = Math.max(Math.min(last, 63), 0);
    for (int i = f; i <= l; i++)
        m |= 1L << i;
    return m;
}

//计算first,last之间的高位字节
private static long highMask(char first, char last) {
    long m = 0;
    int f = Math.max(Math.min(first, 127), 64) - 64;
    int l = Math.max(Math.min(last, 127), 64) - 64;
    for (int i = f; i <= l; i++)
        m |= 1L << i;
    return m;
}

// Tell whether the given character is permitted by the given mask pair
//检查传入的掩码对是否允许传入的字符
private static boolean match(char c, long lowMask, long highMask) {
    if (c == 0) // 在掩码中0没有位置，所以永远不会匹配
        return false;
    if (c < 64)
        return ((1L << c) & lowMask) != 0;
    if (c < 128)
        return ((1L << (c - 64)) & highMask) != 0;
    return false;
}

//数字
private static final long L_DIGIT = lowMask('0', '9');
private static final long H_DIGIT = 0L;

//大写字母
private static final long L_UPALPHA = 0L;
private static final long H_UPALPHA = highMask('A', 'Z');

//小写字母
private static final long L_LOWALPHA = 0L;
private static final long H_LOWALPHA = highMask('a', 'z');

//字母
private static final long L_ALPHA = L_LOWALPHA | L_UPALPHA;
private static final long H_ALPHA = H_LOWALPHA | H_UPALPHA;

//字母加数字
private static final long L_ALPHANUM = L_DIGIT | L_ALPHA;
private static final long H_ALPHANUM = H_DIGIT | H_ALPHA;

//十六进制
private static final long L_HEX = L_DIGIT;
private static final long H_HEX = highMask('A', 'F') | highMask('a', 'f');

//符号
private static final long L_MARK = lowMask("-_.!~*'()");
private static final long H_MARK = highMask("-_.!~*'()");

//未保留的：字符+数字+符号
private static final long L_UNRESERVED = L_ALPHANUM | L_MARK;
private static final long H_UNRESERVED = H_ALPHANUM | H_MARK;

//保留的符号
private static final long L_RESERVED = lowMask(";/?:@&=+$,[]");
private static final long H_RESERVED = highMask(";/?:@&=+$,[]");

//第0位用来标识允许转义对和非US-ASCII字符。sanEscape方法会操作这个变量
private static final long L_ESCAPED = 1L;
private static final long H_ESCAPED = 0L;

// uric          = reserved | unreserved | escaped
private static final long L_URIC = L_RESERVED | L_UNRESERVED | L_ESCAPED;
private static final long H_URIC = H_RESERVED | H_UNRESERVED | H_ESCAPED;

// pchar         = unreserved | escaped |
//                 ":" | "@" | "&" | "=" | "+" | "$" | ","
private static final long L_PCHAR
    = L_UNRESERVED | L_ESCAPED | lowMask(":@&=+$,");
private static final long H_PCHAR
    = H_UNRESERVED | H_ESCAPED | highMask(":@&=+$,");

// 所有有效的路径字符
private static final long L_PATH = L_PCHAR | lowMask(";/");
private static final long H_PATH = H_PCHAR | highMask(";/");

//破折号，在域名标签和顶级标签里使用
private static final long L_DASH = lowMask("-");
private static final long H_DASH = highMask("-");

// 主机名用的'.'
private static final long L_DOT = lowMask(".");
private static final long H_DOT = highMask(".");

// 用户信息      = *( unreserved | escaped |
//                    ";" | ":" | "&" | "=" | "+" | "$" | "," )
private static final long L_USERINFO
    = L_UNRESERVED | L_ESCAPED | lowMask(";:&=+$,");
private static final long H_USERINFO
    = H_UNRESERVED | H_ESCAPED | highMask(";:&=+$,");

// 注册名字      = 1*( unreserved | escaped | "$" | "," |
//                     ";" | ":" | "@" | "&" | "=" | "+" )
private static final long L_REG_NAME
    = L_UNRESERVED | L_ESCAPED | lowMask("$,;:@&=+");
private static final long H_REG_NAME
    = H_UNRESERVED | H_ESCAPED | highMask("$,;:@&=+");

//基于服务器的权限中所有有效的字符
private static final long L_SERVER
    = L_USERINFO | L_ALPHANUM | L_DASH | lowMask(".:@[]");
private static final long H_SERVER
    = H_USERINFO | H_ALPHANUM | H_DASH | highMask(".:@[]");

//基于服务器的权限中包含IPV6地址的特殊情况，这个情况下，`%`没有转义功能
private static final long L_SERVER_PERCENT
    = L_SERVER | lowMask("%");
private static final long H_SERVER_PERCENT
    = H_SERVER | highMask("%");
private static final long L_LEFT_BRACKET = lowMask("[");
private static final long H_LEFT_BRACKET = highMask("[");

// 模式       = alpha *( alpha | digit | "+" | "-" | "." )
private static final long L_SCHEME = L_ALPHA | L_DIGIT | lowMask("+-.");
private static final long H_SCHEME = H_ALPHA | H_DIGIT | highMask("+-.");

// uric_no_slash = unreserved | escaped | ";" | "?" | ":" | "@" |
//                 "&" | "=" | "+" | "$" | ","
private static final long L_URIC_NO_SLASH
    = L_UNRESERVED | L_ESCAPED | lowMask(";?:@&=+$,");
private static final long H_URIC_NO_SLASH
    = H_UNRESERVED | H_ESCAPED | highMask(";?:@&=+$,");


// -- 转义和编码--

private final static char[] hexDigits = {
    '0', '1', '2', '3', '4', '5', '6', '7',
    '8', '9', 'A', 'B', 'C', 'D', 'E', 'F'
};

private static void appendEscape(StringBuffer sb, byte b) {
    sb.append('%');
    sb.append(hexDigits[(b >> 4) & 0x0f]);
    sb.append(hexDigits[(b >> 0) & 0x0f]);
}

private static void appendEncoded(StringBuffer sb, char c) {
    ByteBuffer bb = null;
    try {
        bb = ThreadLocalCoders.encoderFor("UTF-8")
            .encode(CharBuffer.wrap("" + c));
    } catch (CharacterCodingException x) {
        assert false;
    }
    while (bb.hasRemaining()) {
        int b = bb.get() & 0xff;
        if (b >= 0x80)
            appendEscape(sb, (byte)b);
        else
            sb.append((char)b);
    }
}

//s字符串中不被掩码对允许的字符会被引号包含
private static String quote(String s, long lowMask, long highMask) {
    int n = s.length();
    StringBuffer sb = null;
    boolean allowNonASCII = ((lowMask & L_ESCAPED) != 0);
    for (int i = 0; i < s.length(); i++) {
        char c = s.charAt(i);
        if (c < '\u0080') {
            if (!match(c, lowMask, highMask)) {
                if (sb == null) {
                    sb = new StringBuffer();
                    sb.append(s.substring(0, i));
                }
                appendEscape(sb, (byte)c);
            } else {
                if (sb != null)
                    sb.append(c);
            }
        } else if (allowNonASCII
                   && (Character.isSpaceChar(c)
                       || Character.isISOControl(c))) {
            if (sb == null) {
                sb = new StringBuffer();
                sb.append(s.substring(0, i));
            }
            appendEncoded(sb, c);
        } else {
            if (sb != null)
                sb.append(c);
        }
    }
    return (sb == null) ? s : sb.toString();
}

//对大于等于\u0080的字符编码
private static String encode(String s) {
    int n = s.length();
    if (n == 0)
        return s;

    //先检查是否需要编码
    for (int i = 0;;) {
        if (s.charAt(i) >= '\u0080')
            break;
        if (++i >= n)
            return s;
    }

    String ns = Normalizer.normalize(s, Normalizer.Form.NFC);
    ByteBuffer bb = null;
    try {
        bb = ThreadLocalCoders.encoderFor("UTF-8")
            .encode(CharBuffer.wrap(ns));
    } catch (CharacterCodingException x) {
        assert false;
    }

    StringBuffer sb = new StringBuffer();
    while (bb.hasRemaining()) {
        int b = bb.get() & 0xff;
        if (b >= 0x80)
            appendEscape(sb, (byte)b);
        else
            sb.append((char)b);
    }
    return sb.toString();
}

private static int decode(char c) {
    if ((c >= '0') && (c <= '9'))
        return c - '0';
    if ((c >= 'a') && (c <= 'f'))
        return c - 'a' + 10;
    if ((c >= 'A') && (c <= 'F'))
        return c - 'A' + 10;
    assert false;
    return -1;
}

private static byte decode(char c1, char c2) {
    return (byte)(  ((decode(c1) & 0xf) << 4)
                  | ((decode(c2) & 0xf) << 0));
}

// Evaluates all escapes in s, applying UTF-8 decoding if needed.  Assumes
// that escapes are well-formed syntactically, i.e., of the form %XX.  If a
// sequence of escaped octets is not valid UTF-8 then the erroneous octets
// are replaced with '\uFFFD'.
// Exception: any "%" found between "[]" is left alone. It is an IPv6 literal
//            with a scope_id
//
//
private static String decode(String s) {
    if (s == null)
        return s;
    int n = s.length();
    if (n == 0)
        return s;
    if (s.indexOf('%') < 0)
        return s;

    StringBuffer sb = new StringBuffer(n);
    ByteBuffer bb = ByteBuffer.allocate(n);
    CharBuffer cb = CharBuffer.allocate(n);
    CharsetDecoder dec = ThreadLocalCoders.decoderFor("UTF-8")
        .onMalformedInput(CodingErrorAction.REPLACE)
        .onUnmappableCharacter(CodingErrorAction.REPLACE);

    // This is not horribly efficient, but it will do for now
    char c = s.charAt(0);
    boolean betweenBrackets = false;

    for (int i = 0; i < n;) {
        assert c == s.charAt(i);    // Loop invariant
        if (c == '[') {
            betweenBrackets = true;
        } else if (betweenBrackets && c == ']') {
            betweenBrackets = false;
        }
        if (c != '%' || betweenBrackets) {
            sb.append(c);
            if (++i >= n)
                break;
            c = s.charAt(i);
            continue;
        }
        bb.clear();
        int ui = i;
        for (;;) {
            assert (n - i >= 2);
            bb.put(decode(s.charAt(++i), s.charAt(++i)));
            if (++i >= n)
                break;
            c = s.charAt(i);
            if (c != '%')
                break;
        }
        bb.flip();
        cb.clear();
        dec.reset();
        CoderResult cr = dec.decode(bb, cb, true);
        assert cr.isUnderflow();
        cr = dec.flush(cb);
        assert cr.isUnderflow();
        sb.append(cb.flip().toString());
    }

    return sb.toString();
}


// -- Parsing --

// For convenience we wrap the input URI string in a new instance of the
// following internal class.  This saves always having to pass the input
// string as an argument to each internal scan/parse method.

private class Parser {

    private String input;           // URI input string
    private boolean requireServerAuthority = false;

    Parser(String s) {
        input = s;
        string = s;
    }

    // -- Methods for throwing URISyntaxException in various ways --

    private void fail(String reason) throws URISyntaxException {
        throw new URISyntaxException(input, reason);
    }

    private void fail(String reason, int p) throws URISyntaxException {
        throw new URISyntaxException(input, reason, p);
    }

    private void failExpecting(String expected, int p)
        throws URISyntaxException
    {
        fail("Expected " + expected, p);
    }

    private void failExpecting(String expected, String prior, int p)
        throws URISyntaxException
    {
        fail("Expected " + expected + " following " + prior, p);
    }


    // -- Simple access to the input string --

    // Return a substring of the input string
    //
    private String substring(int start, int end) {
        return input.substring(start, end);
    }

    // Return the char at position p,
    // assuming that p < input.length()
    //
    private char charAt(int p) {
        return input.charAt(p);
    }

    // Tells whether start < end and, if so, whether charAt(start) == c
    //
    private boolean at(int start, int end, char c) {
        return (start < end) && (charAt(start) == c);
    }

    // Tells whether start + s.length() < end and, if so,
    // whether the chars at the start position match s exactly
    //
    private boolean at(int start, int end, String s) {
        int p = start;
        int sn = s.length();
        if (sn > end - p)
            return false;
        int i = 0;
        while (i < sn) {
            if (charAt(p++) != s.charAt(i)) {
                break;
            }
            i++;
        }
        return (i == sn);
    }


    // -- Scanning --

    // The various scan and parse methods that follow use a uniform
    // convention of taking the current start position and end index as
    // their first two arguments.  The start is inclusive while the end is
    // exclusive, just as in the String class, i.e., a start/end pair
    // denotes the left-open interval [start, end) of the input string.
    //
    // These methods never proceed past the end position.  They may return
    // -1 to indicate outright failure, but more often they simply return
    // the position of the first char after the last char scanned.  Thus
    // a typical idiom is
    //
    //     int p = start;
    //     int q = scan(p, end, ...);
    //     if (q > p)
    //         // We scanned something
    //         ...;
    //     else if (q == p)
    //         // We scanned nothing
    //         ...;
    //     else if (q == -1)
    //         // Something went wrong
    //         ...;


    // Scan a specific char: If the char at the given start position is
    // equal to c, return the index of the next char; otherwise, return the
    // start position.
    //
    private int scan(int start, int end, char c) {
        if ((start < end) && (charAt(start) == c))
            return start + 1;
        return start;
    }

    // Scan forward from the given start position.  Stop at the first char
    // in the err string (in which case -1 is returned), or the first char
    // in the stop string (in which case the index of the preceding char is
    // returned), or the end of the input string (in which case the length
    // of the input string is returned).  May return the start position if
    // nothing matches.
    //
    private int scan(int start, int end, String err, String stop) {
        int p = start;
        while (p < end) {
            char c = charAt(p);
            if (err.indexOf(c) >= 0)
                return -1;
            if (stop.indexOf(c) >= 0)
                break;
            p++;
        }
        return p;
    }

    // Scan a potential escape sequence, starting at the given position,
    // with the given first char (i.e., charAt(start) == c).
    //
    // This method assumes that if escapes are allowed then visible
    // non-US-ASCII chars are also allowed.
    //
    private int scanEscape(int start, int n, char first)
        throws URISyntaxException
    {
        int p = start;
        char c = first;
        if (c == '%') {
            // Process escape pair
            if ((p + 3 <= n)
                && match(charAt(p + 1), L_HEX, H_HEX)
                && match(charAt(p + 2), L_HEX, H_HEX)) {
                return p + 3;
            }
            fail("Malformed escape pair", p);
        } else if ((c > 128)
                   && !Character.isSpaceChar(c)
                   && !Character.isISOControl(c)) {
            // Allow unescaped but visible non-US-ASCII chars
            return p + 1;
        }
        return p;
    }

    // Scan chars that match the given mask pair
    //
    private int scan(int start, int n, long lowMask, long highMask)
        throws URISyntaxException
    {
        int p = start;
        while (p < n) {
            char c = charAt(p);
            if (match(c, lowMask, highMask)) {
                p++;
                continue;
            }
            if ((lowMask & L_ESCAPED) != 0) {
                int q = scanEscape(p, n, c);
                if (q > p) {
                    p = q;
                    continue;
                }
            }
            break;
        }
        return p;
    }

    // Check that each of the chars in [start, end) matches the given mask
    //
    private void checkChars(int start, int end,
                            long lowMask, long highMask,
                            String what)
        throws URISyntaxException
    {
        int p = scan(start, end, lowMask, highMask);
        if (p < end)
            fail("Illegal character in " + what, p);
    }

    // Check that the char at position p matches the given mask
    //
    private void checkChar(int p,
                           long lowMask, long highMask,
                           String what)
        throws URISyntaxException
    {
        checkChars(p, p + 1, lowMask, highMask, what);
    }


    // -- Parsing --

    // [<scheme>:]<scheme-specific-part>[#<fragment>]
    //
    void parse(boolean rsa) throws URISyntaxException {
        requireServerAuthority = rsa;
        int ssp;                    // Start of scheme-specific part
        int n = input.length();
        int p = scan(0, n, "/?#", ":");
        if ((p >= 0) && at(p, n, ':')) {
            if (p == 0)
                failExpecting("scheme name", 0);
            checkChar(0, L_ALPHA, H_ALPHA, "scheme name");
            checkChars(1, p, L_SCHEME, H_SCHEME, "scheme name");
            scheme = substring(0, p);
            p++;                    // Skip ':'
            ssp = p;
            if (at(p, n, '/')) {
                p = parseHierarchical(p, n);
            } else {
                int q = scan(p, n, "", "#");
                if (q <= p)
                    failExpecting("scheme-specific part", p);
                checkChars(p, q, L_URIC, H_URIC, "opaque part");
                p = q;
            }
        } else {
            ssp = 0;
            p = parseHierarchical(0, n);
        }
        schemeSpecificPart = substring(ssp, p);
        if (at(p, n, '#')) {
            checkChars(p + 1, n, L_URIC, H_URIC, "fragment");
            fragment = substring(p + 1, n);
            p = n;
        }
        if (p < n)
            fail("end of URI", p);
    }

    // [//authority]<path>[?<query>]
    //
    // DEVIATION from RFC2396: We allow an empty authority component as
    // long as it's followed by a non-empty path, query component, or
    // fragment component.  This is so that URIs such as "file:///foo/bar"
    // will parse.  This seems to be the intent of RFC2396, though the
    // grammar does not permit it.  If the authority is empty then the
    // userInfo, host, and port components are undefined.
    //
    // DEVIATION from RFC2396: We allow empty relative paths.  This seems
    // to be the intent of RFC2396, but the grammar does not permit it.
    // The primary consequence of this deviation is that "#f" parses as a
    // relative URI with an empty path.
    //
    private int parseHierarchical(int start, int n)
        throws URISyntaxException
    {
        int p = start;
        if (at(p, n, '/') && at(p + 1, n, '/')) {
            p += 2;
            int q = scan(p, n, "", "/?#");
            if (q > p) {
                p = parseAuthority(p, q);
            } else if (q < n) {
                // DEVIATION: Allow empty authority prior to non-empty
                // path, query component or fragment identifier
            } else
                failExpecting("authority", p);
        }
        int q = scan(p, n, "", "?#"); // DEVIATION: May be empty
        checkChars(p, q, L_PATH, H_PATH, "path");
        path = substring(p, q);
        p = q;
        if (at(p, n, '?')) {
            p++;
            q = scan(p, n, "", "#");
            checkChars(p, q, L_URIC, H_URIC, "query");
            query = substring(p, q);
            p = q;
        }
        return p;
    }

    // authority     = server | reg_name
    //
    // Ambiguity: An authority that is a registry name rather than a server
    // might have a prefix that parses as a server.  We use the fact that
    // the authority component is always followed by '/' or the end of the
    // input string to resolve this: If the complete authority did not
    // parse as a server then we try to parse it as a registry name.
    //
    private int parseAuthority(int start, int n)
        throws URISyntaxException
    {
        int p = start;
        int q = p;
        URISyntaxException ex = null;

        boolean serverChars;
        boolean regChars;

        if (scan(p, n, "", "]") > p) {
            // contains a literal IPv6 address, therefore % is allowed
            serverChars = (scan(p, n, L_SERVER_PERCENT, H_SERVER_PERCENT) == n);
        } else {
            serverChars = (scan(p, n, L_SERVER, H_SERVER) == n);
        }
        regChars = (scan(p, n, L_REG_NAME, H_REG_NAME) == n);

        if (regChars && !serverChars) {
            // Must be a registry-based authority
            authority = substring(p, n);
            return n;
        }

        if (serverChars) {
            // Might be (probably is) a server-based authority, so attempt
            // to parse it as such.  If the attempt fails, try to treat it
            // as a registry-based authority.
            try {
                q = parseServer(p, n);
                if (q < n)
                    failExpecting("end of authority", q);
                authority = substring(p, n);
            } catch (URISyntaxException x) {
                // Undo results of failed parse
                userInfo = null;
                host = null;
                port = -1;
                if (requireServerAuthority) {
                    // If we're insisting upon a server-based authority,
                    // then just re-throw the exception
                    throw x;
                } else {
                    // Save the exception in case it doesn't parse as a
                    // registry either
                    ex = x;
                    q = p;
                }
            }
        }

        if (q < n) {
            if (regChars) {
                // Registry-based authority
                authority = substring(p, n);
            } else if (ex != null) {
                // Re-throw exception; it was probably due to
                // a malformed IPv6 address
                throw ex;
            } else {
                fail("Illegal character in authority", q);
            }
        }

        return n;
    }


    // [<userinfo>@]<host>[:<port>]
    //
    private int parseServer(int start, int n)
        throws URISyntaxException
    {
        int p = start;
        int q;

        // userinfo
        q = scan(p, n, "/?#", "@");
        if ((q >= p) && at(q, n, '@')) {
            checkChars(p, q, L_USERINFO, H_USERINFO, "user info");
            userInfo = substring(p, q);
            p = q + 1;              // Skip '@'
        }

        // hostname, IPv4 address, or IPv6 address
        if (at(p, n, '[')) {
            // DEVIATION from RFC2396: Support IPv6 addresses, per RFC2732
            p++;
            q = scan(p, n, "/?#", "]");
            if ((q > p) && at(q, n, ']')) {
                // look for a "%" scope id
                int r = scan (p, q, "", "%");
                if (r > p) {
                    parseIPv6Reference(p, r);
                    if (r+1 == q) {
                        fail ("scope id expected");
                    }
                    checkChars (r+1, q, L_ALPHANUM, H_ALPHANUM,
                                            "scope id");
                } else {
                    parseIPv6Reference(p, q);
                }
                host = substring(p-1, q+1);
                p = q + 1;
            } else {
                failExpecting("closing bracket for IPv6 address", q);
            }
        } else {
            q = parseIPv4Address(p, n);
            if (q <= p)
                q = parseHostname(p, n);
            p = q;
        }

        // port
        if (at(p, n, ':')) {
            p++;
            q = scan(p, n, "", "/");
            if (q > p) {
                checkChars(p, q, L_DIGIT, H_DIGIT, "port number");
                try {
                    port = Integer.parseInt(substring(p, q));
                } catch (NumberFormatException x) {
                    fail("Malformed port number", p);
                }
                p = q;
            }
        }
        if (p < n)
            failExpecting("port number", p);

        return p;
    }

    // Scan a string of decimal digits whose value fits in a byte
    //
    private int scanByte(int start, int n)
        throws URISyntaxException
    {
        int p = start;
        int q = scan(p, n, L_DIGIT, H_DIGIT);
        if (q <= p) return q;
        if (Integer.parseInt(substring(p, q)) > 255) return p;
        return q;
    }

    // Scan an IPv4 address.
    //
    // If the strict argument is true then we require that the given
    // interval contain nothing besides an IPv4 address; if it is false
    // then we only require that it start with an IPv4 address.
    //
    // If the interval does not contain or start with (depending upon the
    // strict argument) a legal IPv4 address characters then we return -1
    // immediately; otherwise we insist that these characters parse as a
    // legal IPv4 address and throw an exception on failure.
    //
    // We assume that any string of decimal digits and dots must be an IPv4
    // address.  It won't parse as a hostname anyway, so making that
    // assumption here allows more meaningful exceptions to be thrown.
    //
    private int scanIPv4Address(int start, int n, boolean strict)
        throws URISyntaxException
    {
        int p = start;
        int q;
        int m = scan(p, n, L_DIGIT | L_DOT, H_DIGIT | H_DOT);
        if ((m <= p) || (strict && (m != n)))
            return -1;
        for (;;) {
            // Per RFC2732: At most three digits per byte
            // Further constraint: Each element fits in a byte
            if ((q = scanByte(p, m)) <= p) break;   p = q;
            if ((q = scan(p, m, '.')) <= p) break;  p = q;
            if ((q = scanByte(p, m)) <= p) break;   p = q;
            if ((q = scan(p, m, '.')) <= p) break;  p = q;
            if ((q = scanByte(p, m)) <= p) break;   p = q;
            if ((q = scan(p, m, '.')) <= p) break;  p = q;
            if ((q = scanByte(p, m)) <= p) break;   p = q;
            if (q < m) break;
            return q;
        }
        fail("Malformed IPv4 address", q);
        return -1;
    }

    // Take an IPv4 address: Throw an exception if the given interval
    // contains anything except an IPv4 address
    //
    private int takeIPv4Address(int start, int n, String expected)
        throws URISyntaxException
    {
        int p = scanIPv4Address(start, n, true);
        if (p <= start)
            failExpecting(expected, start);
        return p;
    }

    // Attempt to parse an IPv4 address, returning -1 on failure but
    // allowing the given interval to contain [:<characters>] after
    // the IPv4 address.
    //
    private int parseIPv4Address(int start, int n) {
        int p;

        try {
            p = scanIPv4Address(start, n, false);
        } catch (URISyntaxException x) {
            return -1;
        } catch (NumberFormatException nfe) {
            return -1;
        }

        if (p > start && p < n) {
            // IPv4 address is followed by something - check that
            // it's a ":" as this is the only valid character to
            // follow an address.
            if (charAt(p) != ':') {
                p = -1;
            }
        }

        if (p > start)
            host = substring(start, p);

        return p;
    }

    // hostname      = domainlabel [ "." ] | 1*( domainlabel "." ) toplabel [ "." ]
    // domainlabel   = alphanum | alphanum *( alphanum | "-" ) alphanum
    // toplabel      = alpha | alpha *( alphanum | "-" ) alphanum
    //
    private int parseHostname(int start, int n)
        throws URISyntaxException
    {
        int p = start;
        int q;
        int l = -1;                 // Start of last parsed label

        do {
            // domainlabel = alphanum [ *( alphanum | "-" ) alphanum ]
            q = scan(p, n, L_ALPHANUM, H_ALPHANUM);
            if (q <= p)
                break;
            l = p;
            if (q > p) {
                p = q;
                q = scan(p, n, L_ALPHANUM | L_DASH, H_ALPHANUM | H_DASH);
                if (q > p) {
                    if (charAt(q - 1) == '-')
                        fail("Illegal character in hostname", q - 1);
                    p = q;
                }
            }
            q = scan(p, n, '.');
            if (q <= p)
                break;
            p = q;
        } while (p < n);

        if ((p < n) && !at(p, n, ':'))
            fail("Illegal character in hostname", p);

        if (l < 0)
            failExpecting("hostname", start);

        // for a fully qualified hostname check that the rightmost
        // label starts with an alpha character.
        if (l > start && !match(charAt(l), L_ALPHA, H_ALPHA)) {
            fail("Illegal character in hostname", l);
        }

        host = substring(start, p);
        return p;
    }


    // IPv6 address parsing, from RFC2373: IPv6 Addressing Architecture
    //
    // Bug: The grammar in RFC2373 Appendix B does not allow addresses of
    // the form ::12.34.56.78, which are clearly shown in the examples
    // earlier in the document.  Here is the original grammar:
    //
    //   IPv6address = hexpart [ ":" IPv4address ]
    //   hexpart     = hexseq | hexseq "::" [ hexseq ] | "::" [ hexseq ]
    //   hexseq      = hex4 *( ":" hex4)
    //   hex4        = 1*4HEXDIG
    //
    // We therefore use the following revised grammar:
    //
    //   IPv6address = hexseq [ ":" IPv4address ]
    //                 | hexseq [ "::" [ hexpost ] ]
    //                 | "::" [ hexpost ]
    //   hexpost     = hexseq | hexseq ":" IPv4address | IPv4address
    //   hexseq      = hex4 *( ":" hex4)
    //   hex4        = 1*4HEXDIG
    //
    // This covers all and only the following cases:
    //
    //   hexseq
    //   hexseq : IPv4address
    //   hexseq ::
    //   hexseq :: hexseq
    //   hexseq :: hexseq : IPv4address
    //   hexseq :: IPv4address
    //   :: hexseq
    //   :: hexseq : IPv4address
    //   :: IPv4address
    //   ::
    //
    // Additionally we constrain the IPv6 address as follows :-
    //
    //  i.  IPv6 addresses without compressed zeros should contain
    //      exactly 16 bytes.
    //
    //  ii. IPv6 addresses with compressed zeros should contain
    //      less than 16 bytes.

    private int ipv6byteCount = 0;

    private int parseIPv6Reference(int start, int n)
        throws URISyntaxException
    {
        int p = start;
        int q;
        boolean compressedZeros = false;

        q = scanHexSeq(p, n);

        if (q > p) {
            p = q;
            if (at(p, n, "::")) {
                compressedZeros = true;
                p = scanHexPost(p + 2, n);
            } else if (at(p, n, ':')) {
                p = takeIPv4Address(p + 1,  n, "IPv4 address");
                ipv6byteCount += 4;
            }
        } else if (at(p, n, "::")) {
            compressedZeros = true;
            p = scanHexPost(p + 2, n);
        }
        if (p < n)
            fail("Malformed IPv6 address", start);
        if (ipv6byteCount > 16)
            fail("IPv6 address too long", start);
        if (!compressedZeros && ipv6byteCount < 16)
            fail("IPv6 address too short", start);
        if (compressedZeros && ipv6byteCount == 16)
            fail("Malformed IPv6 address", start);

        return p;
    }

    private int scanHexPost(int start, int n)
        throws URISyntaxException
    {
        int p = start;
        int q;

        if (p == n)
            return p;

        q = scanHexSeq(p, n);
        if (q > p) {
            p = q;
            if (at(p, n, ':')) {
                p++;
                p = takeIPv4Address(p, n, "hex digits or IPv4 address");
                ipv6byteCount += 4;
            }
        } else {
            p = takeIPv4Address(p, n, "hex digits or IPv4 address");
            ipv6byteCount += 4;
        }
        return p;
    }

    // Scan a hex sequence; return -1 if one could not be scanned
    //
    private int scanHexSeq(int start, int n)
        throws URISyntaxException
    {
        int p = start;
        int q;

        q = scan(p, n, L_HEX, H_HEX);
        if (q <= p)
            return -1;
        if (at(q, n, '.'))          // Beginning of IPv4 address
            return -1;
        if (q > p + 4)
            fail("IPv6 hexadecimal digit sequence too long", p);
        ipv6byteCount += 2;
        p = q;
        while (p < n) {
            if (!at(p, n, ':'))
                break;
            if (at(p + 1, n, ':'))
                break;              // "::"
            p++;
            q = scan(p, n, L_HEX, H_HEX);
            if (q <= p)
                failExpecting("digits for an IPv6 address", p);
            if (at(q, n, '.')) {    // Beginning of IPv4 address
                p--;
                break;
            }
            if (q > p + 4)
                fail("IPv6 hexadecimal digit sequence too long", p);
            ipv6byteCount += 2;
            p = q;
        }

        return p;
    }

}
```