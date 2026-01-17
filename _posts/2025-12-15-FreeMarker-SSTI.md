---
layout: post
date: 2025-12-15
title: "JAVA-FreeMarker-SSTI"
author: "XinYiSleep"
category: Java
---
<h1 id="X26fk">一基础信息</h1>


```
这里FreeMarker还是有必要学习的，FreeMarker是不可以通过传参rce的，攻击面对比其他模板就不是很灵活了，比如：
	1.通过文件上传html
	2.模块编辑，如在后台可以编辑html等
可以看到都是很直接的，一般情况下还真不一定可以但也不少，下面我们用代码来了解一下FreeMarker,这里师傅们看个热闹就行，
因为我也不是特别会，首先肯定是引入组件了代码一,接着代码二就是我们的实例。
漏洞关键字：
	.process(
	Configuration
```

```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.freemarker</groupId>
    <artifactId>freemarker</artifactId>
    <version>2.3.30</version>
</dependency>
```

```java
//如文件123.ftl
<html>
<head>
    <title>User Info</title>
</head>
<body>
<h1>Hello, ${username}</h1>
<#assign value="freemarker.template.utility.Execute"?new()>${value("Calc")}
</body>
</html>

//Controller          
import freemarker.template.Configuration;
import freemarker.template.Template;
import freemarker.template.TemplateException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import java.io.IOException;
import java.io.StringWriter;
import java.util.HashMap;

@Controller
public class Freemarker {
    @Autowired
    private Configuration free;

    @RequestMapping("/index")
    @ResponseBody
    public String admin(String xss,String file) throws IOException, TemplateException {
        HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
        objectObjectHashMap.put("username",xss);
        Template template = free.getTemplate(file);
        StringWriter writer = new StringWriter();

//        free.setNewBuiltinClassResolver(TemplateClassResolver.SAFER_RESOLVER);
//        free.setNewBuiltinClassResolver(TemplateClassResolver.ALLOWS_NOTHING_RESOLVER);
        template.process(objectObjectHashMap,writer);

        return writer.toString();
    }
}

```

```
在上面代码中可以看到${username}和之前我们讲的el表达式一样的语法，其中可以看到${username}是从Controller中的HashMap中
获取到的当然也可以使用Model 的`addAttribute`，这里我们需要知道的是漏洞产生点就是当模块渲染的时候也就是执行到了.process(
方法的时候触发了漏洞，ok知道了这些之后我们看下图一的打印结果，发现执行了username的key，但是可以看到是执行了系统命令的，

<#assign value="freemarker.template.utility.Execute"?new()>${value("Calc")}

这也就是FreeMarker的 FTL 指令以 `#` 开头  ，那么在代码审计的时候主要查找观察的就是.process(方法就可以了。
```

![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/1.png)



```
这里可以看到的是我写了一个renderMergedTemplateModel关键字，这是什么呢？

1.AbstractTemplateView 是 Spring 里“模板视图”体系的顶层模板类，定义了所有渲染流程的抽象方法。

2.MVC 的渲染流程（最终输出html、xml等）都经过这个链路。

3.renderMergedTemplateModel 就是必须实现的抽象方法，被子类继承并实现后，用来“组合Model数据和模板，输出到浏览器”。

说白了就是说AbstractTemplateView这个抽象类就是关于springmvc模块渲染最顶类，其中关于renderMergedTemplateModel方法
就是这个类的主要功能来进行渲染输出的，那么在代码审计中如果在controller中没有看到process关键字就去找renderMergedTemplateModel
方法有可能重写了逻辑但是最终还是需要执行process的下面图一。
```
![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/14.png)
<h1 id="gwroR">二.代码分析</h1>
```
在我们简单了解了之后就需要知道解决三个疑问

1.漏洞是在哪产生

2.如何产生的

3.能否有其他方式

第一个我们已经知道了是通过process产生的，还有个问题就是在我们上面的案例中可以发现文件路径是我们可以控制的file参数,
那么肯定就有师傅想到了是不是可以执行任意的文件通过跳转目录，其实并不能，Template template = free.getTemplate(file);
在这个Configuration#getTemplate方法里面是存在过滤TemplateNameFormat#normalizeRootBasedName图一，其中我们可以看到180
行使用了indexof这个方法中是否有/../(存在就是0不存在就是-1)，存在就直接抛出异常了，那么师傅又想到了../呢？只跳转一次目录，
187行中也已经告诉我们了不可能(不要灰心哦，**安全无绝对我能这么讲嘿嘿师傅们自行分析**),所以该漏洞如果想通过文件上传在进行渲
染必须上传的路径可以控制，或者直接上传到渲染目录了(这种可能性非常小)。
```

![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/2.png)

```
下面我们debug进行调试解决如何产生的，Environment#process图一，其中getTemplate().getRootTreeNode()就是获取到了我们的内容跟
进去visit方法图二，其中会走到accept方法会获取我们的内容并且把我们的内容变成图三，之后for循环内容挨个进入到accept方法师傅问不应该是visit方法吗？
你注意看visit方法不就是又执行回来了吗哈哈所以这里最关键的应该是accept方法，这里debug直接循环到我们插入的
<#assign value="freemarker.template.utility.Execute"?new()>${value("Calc")}跟进去。
```

![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/3.png)

![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/4.png)

![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/5.png)

```
跟进去就是一路的eval方法一直跟到下面图一，会判断我们的插入的freemarker.template.utility.Execute是不是实现了TemplateNumberModel或者TemplateDateModel
不是走到coerceModelToTextualCommon方法进去之后也是判断是不是实现了TemplateScalarModel方法是就会进入到modelToString方法，
这里是实现了TemplateScalarModel接口那么modelToString其实就是返回类完整全限类名的字符串，目的是什么呢？没错其实就是动态加载类会返回到NewBI#_eval方法中图二，
其中会实例化ConstructorFunction这个内部类，其中进入到resolve方法里面我就不截图了就是动态类加载Class.forName,图三返回回来之后开始判断我们的
freemarker.template.utility.Execute是不是实现了TemplateModel了确实是的后面就走出去了,到这里我们就知道了一个最关键的条件就是实现TemplateModel方法。
```

![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/6.png)

![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/8.png)

```
回到MethodCall#_eval图一，这里并不会直接执行exec而是再次循环到${value("Calc")时候执行了
Execute#exec方法，图二就是朴实无华的Runtime操作执行了命令到这里我们就已经搞清楚如何产生的。
```

![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/9.png)

![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/10.png)

<h1 id="gwro">三.关于指定目录跳转绕过</h1>
```
在前面我们知道了指定了目录通过../是没办法绕过的也就是代码分析的图一我们是可以看到有防护措施的，但是并没有过滤..\这就引发了安全问题，
如果在存在return来返回页面内容是不是就存在了任意文件读取呢?目前这么认为是没有任何问题的，因为渲染在返回结果这个流程是非常常见的组合，
ok分析看看呢？这里师傅们可以直接定位到TemplateCache#lookupWithLocalizedThenAcquisitionStrategy方法图一
这个方法就是查找zh_cn文件，如果自己控制的文件名会在920行获取第一个.,就会如123.html变成123_zh_CN.html文件.进行循环获取到我们的内容，
进入到lookupWithAcquisitionStrategy方法后面的没什么重要操作分别的方法是from-findTemplateSource到这里就是最关键的了。
```

![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/11.png)

![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/12.png)

```
在上面图二中我们看到了一个协议：
	`classpath:/`是一种特殊的资源定位协议，表示从应用程序的类路径(classpath)中查找资源。 

简单了解这个协议其实就是只能访问到classes下面的资源，指定之后那么我们实战试试呢？下面图一
payload: %2e%2e%5capplication.properties
成功访问到资源文件，更细有点慢最近和女朋友去了成都 重庆 桂林 阳朔 看看祖国的大好河山。
```

![](https://xinyisleep.github.io/img/2025/SSTI/FreeMarker/13.png)

