# JavaStruts2 学习与环境搭建

  
  
&lt;!--more--&gt;  
## 0x00 Struts2 基础  
### Struts2 简介  
Apache Struts2 是一个非常优秀的 JavaWeb MVC 框架，2007 年 2 月第一个 full release 版本发布，直到今天，Struts发布 2.5.26 版本，而这些版本中，安全更新已至 S2-061，其中包含了非常多的 RCE 漏洞修复  
[基础资料](https://zhuanlan.zhihu.com/p/34402160)
### Struts2 执行流程  
Struts2 是一个基于 MVC设计模式的 Web 应用框架，它的本质就相当于一个 servlet，在 MVC 设计模式中，Struts2 作为控制器(Controller)来建立模型与视图的数据交互。Struts2是在 Struts 和 WebWork 的技术的基础上进行合并的全新的框架。Struts2 以WebWork 为核心，采用拦截器的机制来处理的请求。这样的设计使得业务逻辑控制器能够与 ServletAPi 完全脱离开  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410131645774.png)
Struts2 的执行流程  
Filter：首先经过核心的过滤器，即在`web.xml`中配置的`filter`及`filter-mapping`。这部分通常会配置`/*`全部的路由交给 Struts2来处理  
Interceptor-stack：执行拦截器，应用程序通常会在拦截器中实现一部分功能。也包括在`struts-core`包中`struts-default.xml`文件配置的默认的一些拦截器  
配置 Action：根据访问路径，找到处理这个请求对应的 Action 控制类，通常配置在`struts.xml`中的`package`中  
最后由Action 控制类执行请求的处理，执行结果可能是视图文件，可能是去访问另一个 Action，结果通过`HTTPServletResponse`响应  
### 如何实现 Action 控制类  
通常有以下的方式  
Action 写微一个 POJO类，并且包含`execute()`方法  
Action 类实现`Action`接口  
Action 类继承`ActionSupport`类  
## 0x02 环境搭建  
IDEA选中`web-app`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410131656542.png)
导入`Struts2`的核心依赖  
```xml  
&lt;dependency&gt;  
    &lt;groupId&gt;org.apache.struts&lt;/groupId&gt;  
    &lt;artifactId&gt;struts2-core&lt;/artifactId&gt;  
    &lt;version&gt;2.0.8&lt;/version&gt;  
&lt;/dependency&gt;  
```  
修改`web.xml`，在这里配置`struts2`的过滤器  
```xml  
&lt;web-app&gt;  
  &lt;display-name&gt;S2-001 Example&lt;/display-name&gt;  
  &lt;filter&gt;  
    &lt;filter-name&gt;struts2&lt;/filter-name&gt;  
    &lt;filter-class&gt;org.apache.struts2.dispatcher.FilterDispatcher&lt;/filter-class&gt;  
  &lt;/filter&gt;  
  &lt;filter-mapping&gt;  
    &lt;filter-name&gt;struts2&lt;/filter-name&gt;  
    &lt;url-pattern&gt;/*&lt;/url-pattern&gt;  
  &lt;/filter-mapping&gt;  
  &lt;welcome-file-list&gt;  
    &lt;welcome-file&gt;index.jsp&lt;/welcome-file&gt;  
  &lt;/welcome-file-list&gt;  
&lt;/web-app&gt;  
```  
main下添加 Java 目录  
创建类  
```java  
package com.test.s201.action;    
    
import com.opensymphony.xwork2.ActionSupport;    
    
public class LoginAction extends ActionSupport {    
    private String username;    
    private String password;    
    
    public String getUsername() {    
        return username;    
    }    
    
    public void setUsername(String username) {    
        this.username = username;    
    }    
    
    public String getPassword() {    
        return password;    
    }    
    
    public void setPassword(String password) {    
        this.password = password;    
    }    
    
    public String execute() throws Exception {    
        if((this.username.isEmpty()) || (this.password.isEmpty())){    
            return &#34;error&#34;;    
        }    
        if((this.username.equalsIgnoreCase(&#34;admin&#34;)) &amp;&amp; (this.password.equalsIgnoreCase(&#34;admin&#34;))){    
            return &#34;success&#34;;    
        }    
        return &#34;error&#34;;    
    }    
}  
```  
然后，在`webapp`目录下创建&amp;修改两个文件-`index.jsp`&amp;`welcome.jsp`，内容如下  
`index.jsp`  
```jsp  
&lt;%@ page language=&#34;java&#34; contentType=&#34;text/html; charset=UTF-8&#34;  
         pageEncoding=&#34;UTF-8&#34;%&gt;  
&lt;%@ taglib prefix=&#34;s&#34; uri=&#34;/struts-tags&#34; %&gt;  
  
&lt;html&gt;  
&lt;head&gt;  
    &lt;meta http-equiv=&#34;Content-Type&#34; content=&#34;text/html; charset=UTF-8&#34;&gt;  
    &lt;title&gt;S2-001&lt;/title&gt;  
&lt;/head&gt;  
&lt;body&gt;  
&lt;p&gt;Hello &lt;s:property value=&#34;username&#34;&gt;&lt;/s:property&gt;&lt;/p&gt;  
&lt;/body&gt;  
&lt;/html&gt;  
```  
在`main`文件夹下创建一个`resources`文件夹，内部添加一个`struts.xml`  
```xml  
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;  
  
&lt;!DOCTYPE struts PUBLIC  
        &#34;-//Apache Software Foundation//DTD Struts Configuration 2.0//EN&#34;  
        &#34;http://struts.apache.org/dtds/struts-2.0.dtd&#34;&gt;  
  
&lt;struts&gt;  
    &lt;package name=&#34;S2-001&#34; extends=&#34;struts-default&#34;&gt;  
        &lt;action name=&#34;login&#34; class=&#34;com.test.s2001.action.LoginAction&#34;&gt;  
            &lt;result name=&#34;success&#34;&gt;welcome.jsp&lt;/result&gt;  
            &lt;result name=&#34;error&#34;&gt;index.jsp&lt;/result&gt;  
        &lt;/action&gt;  
    &lt;/package&gt;  
&lt;/struts&gt;  
```  
在配置tomcat就能正常启动  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410131715653.png)
  
## 0x03 OGNL表达式  
OGNL 是 Object-Grap Navagation Language 的缩写，它是一种功能强大的表达式语言，通过它简单一致的表达式语法，可以存取对象的任意属性，调用对象的方法，遍历整个对象的结构图，实现字段类型转化等功能。它使用相同的表达式去读取对象的属性。  
### OGNL三要素  
表达式：  
表达式是整个 OGNL 的核心内容，所有的 OGNL 操作都是针对表达式解析后进行的。通过表达式来告诉 OGNL 操作到底要干些什么。因此，表达式其实是一个带有语法含义的字符串，整个字符串将规定操作的类型和内容。OGNL 表达式支持大量的表达式，如“链式访问对象”、表达式计算、甚至还支持 lambda 表达式。  
Root 对象：  
OGNL 的 Root 对象可以理解为 OGNL 的操作对象。当我们指定了一个表达式的时候，我们需要指定这个表达式针对是哪个具体的对象。而这个具体的对象就是 Root 对象，这就意味着，如果有一个 OGNL 表达式，那么我们就需要针对 Root 对象来进行 OGNL 表达式的计算并且返回结果  
上下文环境：  
有个 Root 对象和表达式，我们就可以使用 OGNL 进行简单的操作，如对 Root 对象的赋值与取值操作。但是，实际上在 OGNL 的内部，所有的操作都会在一个特定的数据环境中运行。这个数据环境就是上下文环境(Context)。OGNL 的上下文环境是一个 Map 结构，称之为`OrgnlContext`。Root 对象也会被添加到上下文环境当中去。  
说白了上下文就是一个 MAP 结构，它实现了`java.utils.Map`的接口。  
### OGNL的基础使用  
导入 pom.xml  
```xml  
&lt;dependency&gt;  
	&lt;groupId&gt;ognl&lt;/groupId&gt;  
	&lt;artifactId&gt;ognl&lt;/artifactId&gt;  
	&lt;version&gt;3.1.19&lt;/version&gt;  
&lt;/dependency&gt;  
```  
先创建两个实体类  
Address.java  
```java  
package com.test.s201.pojo;    
    
public class Address {    
    private String port;    
    private String address;    
	public Address(String port, String address) {    
	    this.port = port;    
	    this.address = address;    
	}    
    
	public Address() {    
	}  
    public String getPort() {    
        return port;    
    }    
    
    public void setPort(String port) {    
        this.port = port;    
    }    
    
    public String getAddress() {    
        return address;    
    }    
    
    public void setAddress(String address) {    
        this.address = address;    
    }    
}  
```  
User.java  
```java  
package com.test.s201.pojo;    
    
public class User {    
    private String name;    
    private int age;    
    private Address address;    
    
    public User(int age, String name) {    
        this.age = age;    
        this.name = name;    
    }    
    
    public User() {    
    }    
    
    public String getName() {    
        return name;    
    }    
    
    public void setName(String name) {    
        this.name = name;    
    }    
    
    public int getAge() {    
        return age;    
    }    
    
    public void setAge(int age) {    
        this.age = age;    
    }    
    
    public Address getAddress() {    
        return address;    
    }    
    
    public void setAddress(Address address) {    
        this.address = address;    
    }    
}  
```  
OGNL使用`getValue()`方法来获取对象，并且访问对象当中的值。  
#### 对Root对象的访问  
OGNL 使用的是一种链式风格进行对象的访问  
所谓的链式风格，则是累死 StringBuffer 的 append 方法的写法  
```java  
StringBuffer buffer = new StringBuffer();    
buffer.append(&#39;a&#39;).append(&#39;b&#39;).append(&#39;c&#39;).append(&#39;d&#39;);  
```  
VisitRoot.java  
```java  
package com.test.s201;    
    
import com.test.s201.pojo.Address;    
import com.test.s201.pojo.User;    
import ognl.Ognl;    
    
public class VisitRoot {    
    public static void main(String[] args) throws Exception {    
        User user = new User(&#34;Huahua&#34;,18);    
        Address address = new Address(&#34;1001&#34;,&#34;翻斗花园&#34;);    
        user.setAddress(address);    
        System.out.println(Ognl.getValue(&#34;name&#34;,user));    
        System.out.println(Ognl.getValue(&#34;name.length()&#34;,user));    
        System.out.println(Ognl.getValue(&#34;address&#34;,user));    
        System.out.println(Ognl.getValue(&#34;address.port&#34;,user));    
    }    
}  
```  
#### 对上下文对象的访问  
使用 OGNL 的时候如果不设置上下文对象，系统会自动创建一个上下文对象，如果传入的参数当中包含了上下文对象则会使用传入的上下文对象  
当访问上下文环境当中的参数时候，需要在表达式前面加上`&#34;#&#34;`，表示了与访问 Root 对象的区别  
VisitContext.java  
```java  
package com.test.s201;    
    
import com.test.s201.pojo.Address;    
import com.test.s201.pojo.User;    
import ognl.Ognl;    
    
import java.util.HashMap;    
import java.util.Map;    
    
public class VisitContext {    
    public static void main(String[] args) throws Exception {    
        User user = new User(&#34;Huahua&#34;,18);    
        Address address = new Address(&#34;1001&#34;,&#34;翻斗花园&#34;);    
        user.setAddress(address);    
        Map&lt;String,Object&gt; context = new HashMap&lt;String,Object&gt;();    
        context.put(&#34;init&#34;,&#34;hello&#34;);    
        context.put(&#34;user&#34;,user);    
        System.out.println(Ognl.getValue(&#34;init&#34;,context,user));    
        System.out.println(Ognl.getValue(&#34;#user.name&#34;,context,user));    
        System.out.println(Ognl.getValue(&#34;name&#34;,context,user));    
    }    
}  
```  
#### 对静态变量与静态方法的访问  
在 OGNL 表达式当中也可以访问静态变量或者调用静态方法，格式如`@[class]@[field/method()]`  
#### 创建对象  
OGNL 支持直接使用表达式来创建对象。主要有三种情况  
构造 List 对象  
构造 Map 对象  
构造任意对象  
```java  
public class CreateClass {    
    public static void main(String[] args) throws Exception{    
        System.out.println(Ognl.getValue(&#34;#{&#39;key1&#39;:&#39;value1&#39;}&#34;, null));   
        System.out.println(Ognl.getValue(&#34;{&#39;key1&#39;,&#39;value1&#39;}&#34;, null));    
        System.out.println(Ognl.getValue(&#34;new com.test.s201.pojo.User()&#34;, null));    
    }    
}  
```  
弹计算器的 EXP  
```java  
package com.test.s201;    
    
import ognl.Ognl;    
import ognl.OgnlException;    
    
public class EvilCalc {    
    public static void main(String[] args) throws OgnlException {    
        Ognl.getValue(&#34;new java.lang.ProcessBuilder(new java.lang.String[]{\&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator\&#34;}).start()&#34;, null);    
    }    
}  
```  
## 0x04 OGNL表达式总结  
表达式功能操作清单  
```notepad  
1. 基本对象树的访问  
对象树的访问就是通过使用点号将对象的引用串联起来进行。  
例如：xxxx，xxxx.xxxx，xxxx. xxxx. xxxx. xxxx. xxxx  
  
2. 对容器变量的访问  
对容器变量的访问，通过#符号加上表达式进行。  
例如：#xxxx，#xxxx. xxxx，#xxxx.xxxxx. xxxx. xxxx. xxxx  
  
3. 使用操作符号  
OGNL表达式中能使用的操作符基本跟Java里的操作符一样，除了能使用 &#43;, -, *, /, &#43;&#43;, --, ==, !=, = 等操作符之外，还能使用 mod, in, not in等。  
  
4. 容器、数组、对象  
OGNL支持对数组和ArrayList等容器的顺序访问：例如：group.users[0]  
同时，OGNL支持对Map的按键值查找：  
例如：#session[&#39;mySessionPropKey&#39;]  
不仅如此，OGNL还支持容器的构造的表达式：  
例如：{&#34;green&#34;, &#34;red&#34;, &#34;blue&#34;}构造一个List，#{&#34;key1&#34; : &#34;value1&#34;, &#34;key2&#34; : &#34;value2&#34;, &#34;key3&#34; : &#34;value3&#34;}构造一个Map  
你也可以通过任意类对象的构造函数进行对象新建  
例如：new Java.net.URL(&#34;xxxxxx/&#34;)  
  
5. 对静态方法或变量的访问  
要引用类的静态方法和字段，他们的表达方式是一样的@class@member或者@class@method(args)：  
  
6. 方法调用  
直接通过类似Java的方法调用方式进行，你甚至可以传递参数：  
例如：user.getName()，group.users.size()，group.containsUser(#requestUser)  
  
7. 投影和选择  
OGNL支持类似数据库中的投影（projection） 和选择（selection）。  
投影就是选出集合中每个元素的相同属性组成新的集合，类似于关系数据库的字段操作。投影操作语法为 collection.{XXX}，其中XXX 是这个集合中每个元素的公共属性。  
例如：group.userList.{username}将获得某个group中的所有user的name的列表。  
选择就是过滤满足selection 条件的集合元素，类似于关系数据库的纪录操作。选择操作的语法为：collection.{X YYY}，其中X 是一个选择操作符，后面则是选择用的逻辑表达式。而选择操作符有三种：  
? 选择满足条件的所有元素  
^ 选择满足条件的第一个元素  
$ 选择满足条件的最后一个元素  
例如：group.userList.{? #txxx.xxx != null}将获得某个group中user的name不为空的user的  
```  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/javastruts2-%E5%AD%A6%E4%B9%A0%E4%B8%8E%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/  

