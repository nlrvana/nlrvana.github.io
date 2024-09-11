# FastJson-前置基础

  
Fastjson
&lt;!--more--&gt;  
## 0x01Fastjson简介  
Fastjson是Alibaba开发的Java语言编写的高性能JSON库，用于将数据在JSON和Java Object之间相互转换。  
提供两个主要接口来分别实现序列化和反序列化操作  
`JSON.toJSONString`将Java对象转换为`json`对象，序列化的过程。  
`JSON.parseObject/JSON.parse`将json对象重新变回Java对象:反序列化的过程  
所以可以简单的把json理解成是一个字符串。  
## 0x02代码demo  
### 1、序列化代码实现  
这里通过Demo了解下如何使用Fastjson进行序列化和反序列化，以及其中的一些特性之间的区别等等。  
在pom.xml里面导入Fastjson的依赖，这里先倒入`1.2.24`的  
```  
&lt;dependency&gt;    
 &lt;groupId&gt;com.alibaba&lt;/groupId&gt;    
 &lt;artifactId&gt;fastjson&lt;/artifactId&gt;    
 &lt;version&gt;1.2.24&lt;/version&gt;    
&lt;/dependency&gt;  
```  
`Student`类  
```java  
public class Student {    
    private String name;    
    private int age;    
    
    public Student() {    
        System.out.println(&#34;构造函数&#34;);    
    }    
    
    public String getName() {    
        System.out.println(&#34;getName&#34;);    
        return name;    
    }    
    
    public void setName(String name) {    
        System.out.println(&#34;setName&#34;);    
        this.name = name;    
    }    
    
    public void setAge(int age) {    
        System.out.println(&#34;setAge&#34;);    
        this.age = age;    
    }    
    
    public int getAge() {    
        System.out.println(&#34;getAge&#34;);    
        return age;    
    }    
}  
```  
然后写序列化的代码，调用`JSON.toJsonString()`来序列化`Student`类对象  
`StudentSerialize.java`  
```java  
import com.alibaba.fastjson.JSON;    
import com.alibaba.fastjson.serializer.SerializerFeature;    
    
public class StudentSerialize {    
    public static void main(String[] args) {    
        Student student = new Student();    
        student.setName(&#34;huahua&#34;);    
        String jsonString = JSON.toJSONString(student,SerializerFeature.WriteClassName);    
        System.out.println(jsonString);    
    }    
}  
```  
这个地方，序列化的逻辑我们可以调试一下  
首先会进到`JSON`这个类，然后进到它的`toJSONString()`的函数里面，new了一个`SerializeWriter`对象。我们的序列化在这一步就已经是完成了。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408011554011.png)
再进到`JSON`这个类里面的时候多出了个`static`的变量，写着`members of JSON`，这里特别注意一个值`DEFAULT_TYPE_KEY`为`@type`，这个很重要。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408011555494.png)
里面定义了一些初值，赋值给out变量，这个out变量后续作为`JSONSerializer`类构造的参数。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408011557238.png)
继续往下，就是显示部分了，`toString()`方法，最后的运行结果。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408011559111.png)
很明显这句语句是关键的  
`String jsonString=JSON.toJSONString(student,SerializerFeature.WriteClassName);`  
我们关注它的参数  
第一个参数是`student`，是一个对象。  
第二个参数是`SerializerFeature.WriteClassName`，是`JSON.toJSONString()`中的一个设置属性值，设置之后在序列化的时候会多写入一个`@tyoe`，即协商被序列化的类名，`type`可以指定反序列化的类，并且调用其`getter/setter/is`方法。  
`Fastjson`接受的`JSON`可以通过`@type`字段来指定该JSON应当还原成何种类型的对象，在反序列化的时候方便操作。  
```  
// 设置了SerializerFeature.WriteClassName  
构造函数  
setName  
getAge  
getName  
{&#34;@type&#34;:&#34;Student&#34;,&#34;age&#34;:0,&#34;name&#34;:&#34;huahua&#34;}  
// 未设置SerializerFeature.WriteClassName  
构造函数  
setName  
getAge  
getName  
{&#34;age&#34;:0,&#34;name&#34;:&#34;huahua&#34;}  
```  
### 2、反序列化代码实现  
调用`JSON.parseObject()`，代码如下  
`StudentUnserialize01.java`  
```java  
import com.alibaba.fastjson.JSON;    
import com.alibaba.fastjson.parser.Feature;    
    
public class StudentUnserialize01 {    
    public static void main(String[] args) {    
        String jsonString = &#34;{\&#34;@type\&#34;:\&#34;Student\&#34;,\&#34;age\&#34;:0,\&#34;name\&#34;:\&#34;huahua\&#34;}&#34;;    
        Student student = JSON.parseObject(jsonString, Student.class, Feature.SupportNonPublicField);    
        System.out.println(student);    
        System.out.println(student.getClass().getName());    
    }    
}  
```  
运行结果如图  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202408011610088.png)
## 0x03 fastjson反序列化漏洞原理  
fastjson在反序列化的时候会找我们在`@type`中规定的类是哪个类，然后在反序列化的时候会自动调用这些`setter`和`getter`方法的调用，注意！并不是所有的`setter`和`getter`方法。  
直接引用结论，Fastjson会对满足下列要求的`setter/getter`方法进行调用：  
满足条件的`setter`:  
非静态函数  
返回类型为void或当前类  
参数个数为1个  
满足条件的getter  
非静态方法  
无参数  
返回值类型继承自`Collection`或`Map`或`AtomicBoolean`或`Atomiclnteger`或`AtomicLong`  
### 漏洞原理  
由前面知道，Fastjson是自己实现的一套序列化和反序列化机制，不过时用的Java原生的序列化和反序列化机制。无论是哪个版本，Fastjson反序列化漏洞的原理都是一样的，只不过不同版本是针对不同的黑名单或者利用不同利用链来进行绕过利用  
通过Fastjson反序列化漏洞，攻击者可以传入一个恶意构造的JSON内容，程序对其进行反序列化后得到恶意类并执行了恶意类中的恶意函数，进而导致代码执行。  
### 那么如何才能够反序列化恶意类呢？  
由前面demo指导，Fastjson使用`parseObject()/parse()`进行反序列化的时候可以指定类型。如果指定的类型太大，包含太多子类，就有利用空间了。例如，如果指定类型为`Object`和`JSONObject`，则可以反序列化出来任意类。例如代码写`Object o = JSON.parseObject(poc,Object.class)`就可以反序列化出Object类或其任意子类，而Object又是任意类的父类，所以就可以反序列化出所有类。  
### 接着，如何才能反序列化得到恶意类中的恶意函数呢？  
由前面知道，在某些情况下进行反序列化时会将反序列化得到的类的构造函数、`getter`方法、`setter`方法执行一遍，如果这三种方法中存在危险操作，则可能导致反序列化漏洞的存在。换句话说，就是攻击者传入要进行反序列化的类中的构造函数、`getter`方法、`setter`方法中要存在漏洞才能触发。  
我们到`DefaultJSONParser.parseObject(Map object, Object fieldName)`中看下，JSON中以@type形式传入的类的时候，调用`deserializer.deserialize()`处理该类，并去调用这个类的`setter`和`getter`方法：  
```java  
public final Object parseObject(final Map object, Object fieldName) {  
    ...  
    // JSON.DEFAULT_TYPE_KEY即@type  
    if (key == JSON.DEFAULT_TYPE_KEY &amp;&amp; !lexer.isEnabled(Feature.DisableSpecialKeyDetect)) {  
        ...  
        ObjectDeserializer deserializer = config.getDeserializer(clazz);  
        return deserializer.deserialze(this, clazz, fieldName);  
```  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/fastjson-%E5%89%8D%E7%BD%AE%E5%9F%BA%E7%A1%80/  

