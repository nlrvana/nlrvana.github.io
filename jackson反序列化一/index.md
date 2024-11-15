# Jackson反序列化（一）

  
  
&lt;!--more--&gt;  
## 0x01 Jackson基础使用  
### Jackson简介  
Jackson 是一个开源的 Java 反序列化和反序列化工具，可以将Java 对象序列化为 XML 或JSON 格式的字符串，以及将XML 或JSON 格式的字符串反序列化为 Java 对象。  
### 使用Jackson进行序列化与反序列化  
pom.xml  
```xml  
&lt;dependencies&gt;    
  &lt;dependency&gt;    
    &lt;groupId&gt;com.fasterxml.jackson.core&lt;/groupId&gt;    
    &lt;artifactId&gt;jackson-databind&lt;/artifactId&gt;    
    &lt;version&gt;2.7.9&lt;/version&gt;    
  &lt;/dependency&gt;    
  &lt;dependency&gt;    
    &lt;groupId&gt;com.fasterxml.jackson.core&lt;/groupId&gt;    
    &lt;artifactId&gt;jackson-core&lt;/artifactId&gt;    
    &lt;version&gt;2.7.9&lt;/version&gt;    
  &lt;/dependency&gt;    
  &lt;dependency&gt;    
    &lt;groupId&gt;com.fasterxml.jackson.core&lt;/groupId&gt;    
    &lt;artifactId&gt;jackson-annotations&lt;/artifactId&gt;    
    &lt;version&gt;2.7.9&lt;/version&gt;    
  &lt;/dependency&gt;    
&lt;/dependencies&gt;  
```  
自定义 Person 类  
```java  
public class Person {    
    public int age;    
    public String name;    
    
    @Override    
    public String toString() {    
        return &#34;Person{&#34; &#43;    
                &#34;age=&#34; &#43; age &#43;    
                &#34;, name=&#39;&#34; &#43; name &#43; &#39;\&#39;&#39; &#43;    
                &#39;}&#39;;    
    }    
}  
```  
接着编写 Jackson 的序列化与反序列化的代码  
```java  
    
import com.fasterxml.jackson.databind.ObjectMapper;    
    
public class JacksonTest {    
    public static void main(String[] args) throws Exception {    
        Person p = new Person();    
        p.age = 6;    
        p.name = &#34;huahua&#34;;    
        ObjectMapper mapper = new ObjectMapper();    
    
        String json = mapper.writeValueAsString(p);    
        System.out.println(json);    
    
        Person p2 = mapper.readValue(json, Person.class);    
        System.out.println(p2);    
    }    
}  
```  
## 0x02 Jaconson对于多态问题的解决  
Java 多态就是同一个接口使用不同的实例而执行不同的操作。  
那么问题来了，如果对多态类的某一个子类实例在序列化后进行反序列化时，如果能够保证反序列化出来的实例是我们想要的那个特定子类的实例而非多态类的其他子类实例呢？Jackson 实现了 JacksonPolymorphicDeserialization机制来解决这个问题。  
JacksonPolymorphicDeserialization即 Jackson 多态类型的反序列化：在反序列化某个类对象的过程中，如果类的成员变量不是具体类型，比如 Object、接口或抽象类，则可以在 JSON 字符串中指定其具体类型，Jackson 将生成具体类型的实例。  
简单来说，就是将具体的子类信息绑定在反序列化的内容中以便于后续反序列化的时候直接得到目标子类对象，其实现有两种`DefaultTyping`和`@JsonTypeInfo`注解。这里和fastjson相似  
### DefaultTyping  
Jackson 提供一个 enableDefaultTyping 设置，其包含 4 个值，查看`jackson-databind-2.7.9.jar!/com/fasterxml/jackson/databind/ObjectMapper.java`可以看到相关介绍信息  
```java  
public enum DefaultTyping {    
       /**    
        * This value means that only properties that have    
        * {@link java.lang.Object} as declared type (including    
        * generic types without explicit type) will use default    
        * typing.    
        */    
       JAVA_LANG_OBJECT,    
           
       /**    
        * Value that means that default typing will be used for    
        * properties with declared type of {@link java.lang.Object}    
        * or an abstract type (abstract class or interface).    
        * Note that this does &lt;b&gt;not&lt;/b&gt; include array types.    
        *&lt;p&gt;    
        * Since 2.4, this does NOT apply to {@link TreeNode} and its subtypes.    
        */    
       OBJECT_AND_NON_CONCRETE,    
    
       /**    
        * Value that means that default typing will be used for    
        * all types covered by {@link #OBJECT_AND_NON_CONCRETE}    
        * plus all array types for them.    
        *&lt;p&gt;    
        * Since 2.4, this does NOT apply to {@link TreeNode} and its subtypes.    
        */    
       NON_CONCRETE_AND_ARRAYS,    
           
       /**    
        * Value that means that default typing will be used for    
        * all non-final types, with exception of small number of    
        * &#34;natural&#34; types (String, Boolean, Integer, Double), which    
        * can be correctly inferred from JSON; as well as for    
        * all arrays of non-final types.    
        *&lt;p&gt;    
        * Since 2.4, this does NOT apply to {@link TreeNode} and its subtypes.    
        */    
       NON_FINAL    
   }  
```  
默认情况下，即无参数的`enableDefaultTyping`是第二个设置，OBJECT_AND_NON_CONCRETE。  
下面分别对这几个选项进行说明  
#### JAVA_LANG_OBJECT  
JAVA_LANG_OBJECT:当被序列化或反序列化的类里的属性被声明为一个 Object 类型时，会对该 Object 类型的属性进行序列化和反序列化，并且明确规定类名。（当然，这个 Object 本身也得是一个可被序列化的类）  
添加一个 Hacker 类  
```java  
package defaultTyping;    
    
public class Hacker {    
    public String skill = &#34;hiphop&#34;;    
}  
```  
修改 Person 类，添加 Object 类型属性  
新建`JAVA_LANG_OBJECTTest.java`，添加`enableDefaultTyping()`并设置为`JAVA_LANG_OBJECT`  
```java  
package defaultTyping;    
    
import com.fasterxml.jackson.databind.ObjectMapper;    
    
public class JAVA_LANG_OBJECTTest {    
    public static void main(String[] args) throws Exception {    
        Person p = new Person();    
        p.age = 6;    
        p.name = &#34;John Doe&#34;;    
        p.object = new Hacker();    
        ObjectMapper mapper = new ObjectMapper();    
        //设置 JAVA_LANG_OBJECT        mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.JAVA_LANG_OBJECT);    
        String json = mapper.writeValueAsString(p);    
        System.out.println(json);    
        Person p2 = mapper.readValue(json, Person.class);    
        System.out.println(p2);    
    }    
}  
```  
这里我们同样写一个类，是没有添加`enableDefaultTyping()`的  
```java  
package defaultTyping;    
    
import com.fasterxml.jackson.databind.ObjectMapper;    
public class No_JAVA_LANG_OBJECTTest {    
    public static void main(String[] args) throws Exception {    
        Person p = new Person();    
        p.age = 6;    
        p.name = &#34;John Doe&#34;;    
        p.object = new Hacker();    
        ObjectMapper mapper = new ObjectMapper();    
        String json = mapper.writeValueAsString(p);    
        System.out.println(json);    
        Person p2 = mapper.readValue(json, Person.class);    
        System.out.println(p2);    
    }    
}  
```  
  
```java  
//无设置  
{&#34;age&#34;:6,&#34;name&#34;:&#34;John Doe&#34;,&#34;object&#34;:{&#34;skill&#34;:&#34;hiphop&#34;}}  
defaultTyping.Person.age=6, defaultTyping.Person.name=John Doe, {skill=hiphop}  
//设置  
{&#34;age&#34;:6,&#34;name&#34;:&#34;John Doe&#34;,&#34;object&#34;:[&#34;defaultTyping.Hacker&#34;,{&#34;skill&#34;:&#34;hiphop&#34;}]}  
defaultTyping.Person.age=6, defaultTyping.Person.name=John Doe, defaultTyping.Hacker@7c16905e  
```  
输出对比看到， 通过`enableDefaultTyping()`设置设置`JAVA_LANG_OBJECT`后，会多输出`Hacker`类名，且在输出的`Object`属性时直接输出的是`Hacker`类对象，也就是说同时对`Object`属性对象进行了序列化和反序列化操作：  
#### OBJECT_AND_NON_CONCRETE  
OBJECT_AND_NON_CONCRETE：除了前面提到的特征，当类里有Interface、AbstractClass 类时，对其进行序列化和反序列化（当然这些类本身需要时合法的，可被序列化的对象）  
此外，enableDefaultTyping()默认的无参数的设置就是此选项  
添加一个 Sex 接口  
```java  
package defaultTyping.object_and_nonconcrete;    
    
public interface Sex {    
    public void setSex(int sex);    
    public int getSex();    
}  
```  
添加 MySex 类实现 Sex 接口类  
```java  
package defaultTyping.object_and_nonconcrete;    
    
public class MySex implements Sex{    
    
    int sex;    
    @Override    
    public void setSex(int sex) {    
        this.sex = sex    
    }    
    
    @Override    
    public int getSex() {    
        return sex;    
    }    
}  
```  
修改 Person 类  
```java  
package defaultTyping.object_and_nonconcrete;    
    
import com.fasterxml.jackson.databind.ObjectMapper;    
import defaultTyping.Hacker;    
import defaultTyping.Person;    
    
public class OBJECT_AND_NON_CONCRETE_Test {    
    public static void main(String[] args) throws Exception {    
        Person p = new Person();    
        p.age = 6;    
        p.name = &#34;huahua&#34;;    
        p.object = new Hacker();    
        p.sex = new MySex();    
        ObjectMapper mapper = new ObjectMapper();    
        //设置OBJECT_AND_NON_CONCRETE    
        mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.OBJECT_AND_NON_CONCRETE);    
        //或直接无参调用，输出一样    
        //mapper.enableDefaultTyping();    
        String json = mapper.writeValueAsString(p);    
        System.out.println(json);    
    
        Person p2 = mapper.readValue(json,Person.class);    
        System.out.println(p2);    
    }    
}  
```  
输出，可以看到该 interface 类属性被成功序列化和反序列化  
```json  
{&#34;age&#34;:6,&#34;name&#34;:&#34;huahua&#34;,&#34;object&#34;:[&#34;defaultTyping.Hacker&#34;,{&#34;skill&#34;:&#34;hiphop&#34;}],&#34;sex&#34;:[&#34;defaultTyping.object_and_nonconcrete.MySex&#34;,{&#34;sex&#34;:0}]}  
Person.age=6, Person.name=huahua, defaultTyping.Hacker@2a5ca609, defaultTyping.object_and_nonconcrete.MySex@20e2cbe0  
```  
#### NON_CONCRETE_AND_ARRAYS  
NON_CONCRETE_AND_ARRAYS：除了前面提到的特征外，还支持 Array 类型。  
编写序列化与反序列化的代码，在 Object 属性中存在的是数组：  
```java  
package defaultTyping;    
    
import com.fasterxml.jackson.databind.ObjectMapper;    
import defaultTyping.object_and_nonconcrete.MySex;    
    
public class NON_CONCRETE_AND_ARRAYS_Test {    
    public static void main(String[] args) throws Exception {    
        Person p = new Person();    
        p.age = 6;    
        p.name = &#34;huahua&#34;;    
        Hacker[] hackers = new Hacker[2];    
        hackers[0] = new Hacker();    
        hackers[1] = new Hacker();    
        p.object = hackers;    
        p.sex = new MySex();    
        ObjectMapper mapper = new ObjectMapper();    
        //设置 NON_CONCRETE_AND_ARRAYS        mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_CONCRETE_AND_ARRAYS);    
    
        String json = mapper.writeValueAsString(p);    
        System.out.println(json);    
    
        Person p2 = mapper.readValue(json, Person.class);    
        System.out.println(p2);    
    }    
}  
```  
  
```json  
{&#34;age&#34;:6,&#34;name&#34;:&#34;huahua&#34;,&#34;object&#34;:[&#34;[LdefaultTyping.Hacker;&#34;,[{&#34;skill&#34;:&#34;hiphop&#34;},{&#34;skill&#34;:&#34;hiphop&#34;}]],&#34;sex&#34;:[&#34;defaultTyping.object_and_nonconcrete.MySex&#34;,{&#34;sex&#34;:0}]}  
Person.age=6, Person.name=huahua, [LdefaultTyping.Hacker;@475530b9, defaultTyping.object_and_nonconcrete.MySex@1d057a39  
```  
输出看到，类名变成了`&#34;[L&#34;&#43;类名&#43;&#34;;&#34;`，序列化 Object 之后为数组形式，反序列化之后得到`[LdefaultTyping.Hacker;`类对象，说明对 Array 类型成功进行了序列化和反序列化；  
#### NON_FINAL  
NON_FINAL：除了前面的所有特征外，包含即将被序列化的类里的全部、非 final 的属性，也就是相当于整个类，除了 final 歪的属性信息都需要被序列化和反序列化。  
修改 Person 类，添加 Hacker 属性；  
```java  
package defaultTyping;    
import defaultTyping.object_and_nonconcrete.Sex;    
    
public class Person {    
    public int age;    
    public String name;    
    public Object object;    
    public Sex sex;    
    public Hacker hacker;    
    
    @Override    
    public String toString() {    
        return String.format(&#34;Person.age=%d, Person.name=%s, %s, %s, %s&#34;, age, name, object == null ? &#34;null&#34; : object, sex == null ? &#34;null&#34; : sex, hacker == null ? &#34;null&#34; : hacker);    
    }    
}  
```  
编写序列化和反序列化类  
```java  
package defaultTyping;    
    
import com.fasterxml.jackson.databind.ObjectMapper;    
import defaultTyping.object_and_nonconcrete.MySex;    
    
public class NON_FINAL_Test {    
    public static void main(String[] args) throws Exception{    
        Person p = new Person();    
        p.age = 6;    
        p.name = &#34;huahua&#34;;    
        p.object = new Hacker();    
        p.sex = new MySex();    
        p.hacker = new Hacker();    
        ObjectMapper mapper = new ObjectMapper();    
        //设置 NON_FINAL        mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);    
    
        String json = mapper.writeValueAsString(p);    
        System.out.println(json);    
    
        Person p2 = mapper.readValue(json,Person.class);    
        System.out.println(p2);    
    }    
}  
```  
```json  
[&#34;defaultTyping.Person&#34;,{&#34;age&#34;:6,&#34;name&#34;:&#34;huahua&#34;,&#34;object&#34;:[&#34;defaultTyping.Hacker&#34;,{&#34;skill&#34;:&#34;hiphop&#34;}],&#34;sex&#34;:[&#34;defaultTyping.object_and_nonconcrete.MySex&#34;,{&#34;sex&#34;:0}],&#34;hacker&#34;:[&#34;defaultTyping.Hacker&#34;,{&#34;skill&#34;:&#34;hiphop&#34;}]}]  
Person.age=6, Person.name=huahua, defaultTyping.Hacker@2a5ca609, defaultTyping.object_and_nonconcrete.MySex@20e2cbe0, defaultTyping.Hacker@68be2bc2  
```  
输出看到，成功对非 final 的 hacker 属性进行序列化和反序列化  
#### 小结  
从前面的分析指导，DefaultTyping 的几个设置选项是逐渐扩大使用范围的，如下表  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410172057753.png)
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/jackson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B8%80/  

