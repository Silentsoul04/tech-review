---
title: "（CVE-2017-12616）Tomcat 信息泄露"
id: zhfly3199
---

# （CVE-2017-12616）Tomcat 信息泄露

## 一、漏洞简介

CVE-2017-12616(信息泄露):允许未经身份验证的远程攻击者查看敏感信息。如果tomcat开启VirtualDirContext有可能绕过安全限制访问服务器上的JSP文件源码。漏洞触发的先决条件是需要在conf/server.xml配置VirtualDirContex参数，默认情况下tomcat7并不会对该参数进行配置。

那么为什么要配置一个这样的虚拟目录呢？

通过`VirtualDirContext`,允许在单独的一个webapp应用下对外暴露出多个文件系统的目录。在实际开发中，为了避免拷贝静态资源文件如(images等)至webapp目录下，tomcat推荐的做法是在server.xml配置文件中建立虚拟子目录。

在开启了这个配置之后，可以通过windows目录下的文件的解析问题，从而暴露在在目录中的源码。

## 二、漏洞影响

Apache Tomcat 7.0.0 - 7.0.80

## 三、复现过程

### 漏洞调试

在本地搭建一个系统环境，目录结果如下：
在`D:\testtomcat2`的目录下面的文件结构如下:

```
D:.
├─img
│      1.jpg
│
├─src
│      HelloWorld.java
│
├─target
└─web
    │  index.jsp
    │
    └─WEB-INF
            web.xml 
```

在tomcat中的`conf/server.xml`下的``标签下面增加如下的配置:

```
<Context path='/site' docBase="D:\testtomcat2\web" reloadable="true">

```
&lt;Resources className="org.apache.naming.resources.VirtualDirContext" extraResourcePaths="/WEB-INF/classes=F:/testtomcat2/target/classe../img=D:/testtomcat2/img,/tmp=D:/testtomcat2/web" /&gt;

&lt;Loader className="org.apache.catalina.loader.VirtualWebappLoader" virtualClasspath="/WEB-INF/classes=F:/testtomcat2/target/classes" /&gt;

&lt;JarScanner scanAllDirectories="true" /&gt; 
``` `</Context>` 
```

从配置可以发现，我创建了2个虚拟目录。分别为`images`和`tmp`，分别映射到本地的`D:/testtomcat2/img`和`D:/testtomcat2/web`。大家在进行测试的时候，可以根据自己的目录自行参照修改。

部署完毕之后，在浏览器中访问`localhost:8080/si../img/1.jpg`

![image](../img/33cdf860ff8151818e5e853ac006ca16.png)

顺利地出现了`1.jpg`，说明部署正确。

这个漏洞的触发，同样会使用到tomcat中因为文件后缀的解析的问题。和12615是一样的，只有后缀是`jsp`和`jspx`由`JSPservlet`处理，其他都是由`DefaultServlet`处理。在`12615`中配合`PUT`方法，可以通过上传`test.jsp%20`、`test.jsp/`、`test.jsp::$DATA`的方式上传任意的问价，包括webshell。但是在本例中，只能通过`test.jsp%20`和`test.jsp::$DATA`获得源代码，无法通过`test.jsp/`获取源代码。以下就是演示的结果:

![image](../img/2ed42d74fb2adec1f59fed43f1d53295.png)

而访问`http://localhost:8080/site/tmp/index.jsp/`会显示404,

![image](../img/7a653083f83e797c80a28a117481bd1a.png)

整个漏洞的分析过程和12615是一样的，下面就为什么无法使用`test.jsp/`无法获取源代码进行说明。
当访问`http://localhost:8080/site/tmp/index.jsp/`时，是由`Tomcat`中的`DefaultServelt::doGet`来处理。

```
@Override
protected void doGet(HttpServletRequest request,HttpServletResponse response) throws IOException, ServletException {
    // Serve the requested resource, including the data content
    serveResource(request, response, true);
} 
```

追踪进入到`serveResource`，其中的关键代码如下：

```
protected void serveResource(HttpServletRequest request,HttpServletResponse response,boolean content) throws IOException, ServletException {
    boolean serveContent = content;
    // Identify the requested resource path
    String path = getRelativePath(request, true);
    CacheEntry cacheEntry = resources.lookupCache(path);
    // If the resource is not a collection, and the resource path
    // ends with "/" or "\", return NOT FOUND
    if (cacheEntry.context == null) {
        if (path.endsWith("/") || (path.endsWith("\\"))) {
            // Check if we're included so we can return the appropriate
            // missing resource name in the error
            String requestUri = (String) request.getAttribute(
                    RequestDispatcher.INCLUDE_REQUEST_URI);
            if (requestUri == null) {
                requestUri = request.getRequestURI();
            }
            response.sendError(HttpServletResponse.SC_NOT_FOUND,
                                requestUri);
            return;
        }
    }
} 
```

所以当`cacheEntry.context == null`而且`path.endsWith("/") || (path.endsWith("\\"))`，则直接向客户端返回404,所以采用`test.jsp/`方式并不能够成功触发获取服务端漏洞的JSP代码。

所以这也就是为什么通过`test.jsp/`获取源代码的原因了。

## 参考链接

> https://blog.spoock.com/2017/09/25/tomcat-cve-2017-12615-12616/