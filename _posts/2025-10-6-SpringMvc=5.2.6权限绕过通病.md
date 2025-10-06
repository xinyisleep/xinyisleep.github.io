---
layout: post
date: 2025-10-6
title: "SpringMvc<=5.2.6(<=2.3.0)权限绕过通病"
author: "XinYiSleep"
category: Java
---
<h1 id="UQ1CO">一：基础信息</h1>

```
这两天国庆节打算好好和朋友畅饮一番(实际基本天天畅饮)，昨天突然看到关于权限验证的一个问题，
发现我也没写过相关文章，其实这个问题是java进行权限验证的通病，在Spring Boot<=2.3.0.RELEASE spring-webmvc<=5.2.6,
在没有严格的验证请求的话如单纯使用getRequestURI()获取url，就会造成权限验证../绕过，在shiro中就出现过cve其中本质问题就是springmvc处理的问题
，昨天在看文章 ‘Spring 中 ../的处理机制为何时灵时不灵’讲到了在高版本通过上下文路径进行绕过的方法server.servlet.context-path，
今天我们棕头到尾的详细分析一下。
```
```java
低版本
<properties>
    <java.version>1.8</java.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <spring-boot.version>2.3.0.RELEASE</spring-boot.version>
</properties>

高版本
<properties>
    <java.version>1.8</java.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <spring-boot.version>2.3.1.RELEASE</spring-boot.version>
</properties>
<properties>
    <java.version>1.8</java.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <spring-boot.version>2.7.0</spring-boot.version>
</properties>
```

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/1.jpg)


<h1 id="EafED">二：spring-webmvc<=5.2.6(2.3.0)绕过</h1>
```
首先我们在org.springframework.web.servlet.handler.AbstractHandlerMethodMapping类中找到getHandlerInternal方法进行断点分析，
为什么是这里因为再往上就是doGet了下面图一,下面我们跟进去getLookupPathForRequest，下面我们直接走进去看看getPathWithinServletMapping
方法图二，直接我就直接讲关键点了，一些对请求路由一些处理我就不细看了，接着我们看看getServletPath方法是怎么处理我们的路由的图三，
这里其实没必要再跟进去了，因为底层其实就是Request类的getServletPath方法，那么他就是获取我们的请求返回调用servlet的URL部分如当然是
不包含请求上下文路径的，那么这里就会造成权限验证的一个问题当然如在filter过滤器中进行了严格的验证就不会出现问题，
如果只是使用getRequestURI()那么就是权限验证问题了。
```

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/2.png)

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/3.png)

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/4.png)

```
这里我们做一个实验代码一图一，本质问题就是getServletPath方法造成的，其实师傅们可以自己走一遍，
其实也有不少细节之类的如在请求中会把/admin;aaaaa/;aaaaa变成空之类的，下面我们看一下在2.3.1版本中是如何修复的呢？
其实很简单在上面图getLookupPathForRequest方法alwaysUseFullPath属性在老版本中是false，在2.3.1中之后在启动项目的时候
设置成了true导致if走的分支不通这里我们可以看下图二是新版本的修复，因为没有走getPathWithinServletMapping方法而是走到了
getPathWithinApplication。
```

```java
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/admin")
public class Admin {
    @RequestMapping("/demo")
    public String index() {
        return "admin";
    }
}
```

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/5.png)

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/6.png)


<h1 id="siA2Z">三：spring-webmvc>5.2.6(Spring Boot>2.3.0)绕过</h1>

<h2 id="Kd5Gb">3.1：2.3.1绕过</h2>
```
上面我们得知了因为修复导致我们并没有走到else分支，那么我们进去看看getPathWithinApplication方法
图一下面我们一个一个看，先看看getContextPath，那么其实就是Request类getContextPath方法，
其中会先去判断我们有没有配置上下文路径server.servlet.context-path=/aaaa，来获取反斜杠的数量图二，
之后在1194行获取我们的上下文路径和1195获取我们完整的路由(getRequestURI())并且会把我们的上下文路径进行一次url解码之类的，
接着查看图三其中会匹配我们请求路由中存在不存在上下文路径之后就返回了如：

/aaaaa/../demo/admin/demo

/aaaaa/../demo

```

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/7.png)

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/8.png)

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/9.png)

```
这里我们直接看getRemainingPath方法代码一,其中会返回我们的请求路径，但是并不包含上下文路径
如

/aaaaa/../demo/admin/demo

/admin/demo

这就导致我们可以使用../进行了绕过，那么如果不配置上下文呢路径呢server.servlet.context-path？还记得
我们上面查看了getContextPath第一行代码就是获取上下文路径的反斜杠，都没配置所有拿来的放斜杠？
没理解的可以看下面图一因为没有反斜杠返回空，所以进入getRemainingPath方法是空截取不到导致的绕过问题，
```

```java
private String getRemainingPath(String requestUri, String mapping, boolean ignoreCase) {
    int index1 = 0;
    int index2 = 0;

    while(true) {
        if (index1 < requestUri.length() && index2 < mapping.length()) {
            char c1 = requestUri.charAt(index1);
            char c2 = mapping.charAt(index2);
            if (c1 == ';') {
                index1 = requestUri.indexOf(47, index1);
                if (index1 == -1) {
                    return null;
                }

                c1 = requestUri.charAt(index1);
            }

            if (c1 == c2 || ignoreCase && Character.toLowerCase(c1) == Character.toLowerCase(c2)) {
                ++index1;
                ++index2;
                continue;
            }

            return null;
        }

        if (index2 != mapping.length()) {
            return null;
        }

        if (index1 == requestUri.length()) {
            return "";
        }

        if (requestUri.charAt(index1) == ';') {
            index1 = requestUri.indexOf(47, index1);
        }

        return index1 != -1 ? requestUri.substring(index1) : "";
    }
}
```

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/10.png)



<h2 id="GHNQt">3.2：2.7.0绕过</h2>
```
上面2.3.1绕过之后，其实后面的高版本变化也不是很大，我们也来看看还是老规矩
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#getHandlerInternal
为什么是这里，可以仔细看看上面我已经说过了，图一进入initLookupPath，进去之后我们可以看到图二，
其中已经发生了改变那么我们看看到底什么地方执行了我们跳转到类型源图三跟进去initContextPath但是我们其实可以看到，
contextPath目前是存在/ssssss/../aaaa，没配置上下文路径是不存在的，那么我们在网上走发现其实就是通过getContextPath()获取的图四。

```

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/11.png)

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/12.png)

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/13.png)

![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/14.png)

```
再上图中其实就是我们分析2.3.1是一样的，我就不进去看了就是获取上下文的路径的反斜杠来确定你是否配置上下文路径，
接着获取上下文路径，进行url解码然后在进行匹配请求路径是不是上下文的路径，最终匹配到了
/aaaa,那么返回的就是/ssssss/../aaaa,那么我们走进extractPathWithinApplication，其实和2.3.1差不多就是获取长度然后
获取上下文之后的路径可以看下图一，好了到这里基本2.3.1和2.7.0就明白了，其实就是获取了上下文路径之后会把上下文路径截断来获取真实的请求，
这里也就导致了可以使用../进行跳目录。
```
![](https://xinyisleep.github.io/img/2025/Spring权限绕过通病/15.png)


<h1 id="hv0dU">四：接着国庆接着喝</h1>
```
都别卷了好好过下国庆吧。
```



