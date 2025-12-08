# Java内存马-基础内容学习

  
  
&lt;!--more--&gt;  
## 0x01 Java Web三大件  
Servlet、Filter、Listener  
当Tomcat接收到请求时候，依次会经过Listener-&gt;Filter-&gt;Servlet  
### Servlet  
#### 什么是Servlet  
Java Servlet是运行在Web服务器或应用服务器上的程序，它是作为来自Web浏览器或其他HTTP客户端的请求和HTTP服务器上的数据库或者应用程序之间的中间层。  
他在应用程序中一般在这个位置，可以理解成半个中间件，不同于其他业务性能很强的中间件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409122255006.png)
#### 请求的处理过程  
客户端发起一个http请求，比如get类型。  
Servlet容器接收到请求，根据请求信息，封装成HttpServletRequest和HttpServletResponse对象。这步就是我们的传参。  
Servlet容器调用HttpServlet的init()方法，init方法只在第一次请求的时候被调用。  
Servlet容器调用service()方法  
service()方法根据请求类型，这里是get类型，分别调用doGet或者doPost方法，这里调用doGet方法。  
doXXX方法中是我们自己写的业务逻辑  
业务逻辑处理完成之后，返回给Servlet容器，然后容器将结果返回给客户端  
容器关闭时候，会调用`destory`方法。  
一整个流程如果用代码来表示的话应该是这样的  
```java  
package tomcatShell.Servlet;    
    
import javax.servlet.*;    
import javax.servlet.annotation.WebServlet;    
import java.io.IOException;    
    
// 基础恶意类    
@WebServlet(&#34;/servlet&#34;)    
public class ServletTest implements Servlet {    
    @Override    
 public void init(ServletConfig config) throws ServletException {    
    
    }    
    
    @Override    
 public ServletConfig getServletConfig() {    
        return null;    
 }    
    
    @Override    
 public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {    
    }    
    
    @Override    
 public String getServletInfo() {    
        return null;    
 }    
    
    @Override    
 public void destroy() {    
    
    }    
}  
```  
#### servlet生命周期  
1)服务器启动时(web.xml中配置load-on-startup=1，默认为0)或者第一次请求该servlet时，就会初始化一个Servlet对象，也就是会执行初始化方法init(ServletConfig conf)  
2)servlet对象去处理所有客户端请求，在`service(ServletRequest req,ServletResponse res)`方法中执行  
3)服务器关闭时，销毁这个servlet对象，执行destory()方法  
4)由JVM进行垃圾回收  
### Filter  
#### Filter简介  
filter也称之为过滤器，是对Servlet技术的一个强补充，其主要功能是在`HttpServletRequest`到达`Servlet`之前，拦截客户的`HttpServletRequest`，根据需要检查`HttpServletRequest`，也可以修改`HttpServletRequest`头和数据；在`HttpServletResponse`到达客户端之前，拦截`HttpServletResponse`，也可以修改`HttpServletResponse`头和数据。  
工作原理如图所示  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409122333379.png)
其实这个地方，我们想办法在Filter前自己创建一个filter并且将其放在最前面，我们的filter就会最先执行，当我们在filter中添加恶意代码，就会进行命令执行，也就成为了一个内存Webshell。  
#### 基本工作原理  
1、Filter程序是一个实现了特殊接口的Java类，与Servlet类似，也是由Servlet容器进行调用和执行的。  
2、当在web.xml注册了一个Filter来对某个Servlet程序进行拦截处理时，它可以决定是否将请求继续传递给Servlet程序，以及对请求和相应消息是否进行修改。  
3、当Servlet容器开始调用某个Servlet程序时，如果发现已经注册了一个Filter程序来对该Servlet进行拦截，那么容器不再直接调用`Servlet`的`service`方法，而是调用Filter的doFilter方法，再由doFilter方法决定是否去激活service方法  
4、但在Filter.doFilter方法中不能直接调用`Servlet`的`service`方法，而是调用`FilterChain.doFilter`方法来激活目标`Servlet`的`service`方法，FilterChain对象时通过`Filter.doFilter`方法的参数传递进来的。  
5、只要在`Filter.doFilter`方法中调用`FilterChain.doFilter`方法的语句前后增加某些程序代码，这样就可以在`Servlet`进行相应前后实现某些特殊功能。  
6、如果在`Filter.doFilter`方法中没有调用`FilterChain.doFilter`方法，则目标`Servlet`的`service`方法不会被执行，这样通过Filter就可以阻止某些非法的访问请求。  
	简单理解就是  
在Filter中的Filter访问需要在web.xml里面定义路径，这就非常人性化，因为有些接口需要加Filter，有些不用  
Filter有一条FilterChain，也就是由多个Filter组成的，会进行一个个的Filter操作，最后一个Filter最后会执行`Servlet.service()`  
#### Filter的生命周期  
与servlet一样，Filter的创建和销毁也由Web容器负责。Web应用程序启动时，Web服务器将会创建Filter的实例对象，并调用其init()方法，读取web.xml配置，完成对象的初始化功能，从而为后续的用户请求作好拦截的准备工作(filter对象只会创建一次，init方法也只会执行一次)。开发人员通过init方法的参数，可获得代表当前filter配置信息的FilterConfig对象。Filter对象创建后会驻留在内存，当Web应用移除或者服务器停止时候才销毁。在Web容器写在Filter对象之前被调用。该方法在Filter的生命周期中仅执行一次。在这个方法中，可以释放过滤器使用的资源。  
先去 web.xml 里面找接口 —&gt; `init()` —&gt; 执行 `doFilter()` —&gt; `destory()`  
```java  
package EvilFliter;    
    
import javax.servlet.*;    
import java.io.IOException;    
    
public class FilterTest implements Filter {    
    @Override    
    public void init(FilterConfig filterConfig) throws ServletException {    
    
    }    
    
    @Override    
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {    
        // 在这里面进行 doGet 和 doPost 这种类似的    
 }    
    
    @Override    
    public void destroy() {    
    
    }    
}  
```  
#### Filter链  
当多个Filter同时存在的时候，组成Filter链。Web服务器根据Filter在web.xml文件中的注册顺序，决定先调用哪个Filter。当第一个Filter的doFilter方法被调用时，web服务器会创建一个代表Filter链的FilterChain对象传递给该方法，通过判断FilterChain中是否还有Filter决定后面是否还调用Filter。  
如图  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409130019474.png)
### Listener  
#### 简介  
Java Web开发中的监听器(Listener)就是Application、Session和Request三大对象创建、销毁或者往其中添加、修改、删除属性时自动执行代码的功能组件。  
ServletContextListener：对Servlet上下文的创建和销毁进行监听；ServletContextAttributeListener：监听Servlet上下文属性的添加、删除和替换。  
HttpSessionListener：对Session的创建和销毁进行监听。Session的销毁有两种情况，一种Session超时，还有一种是调用Session对象的`invalidate()`方法使session失效。  
HttpSessionAttributeListener：对Session对象中属性的添加、删除和替换进行监听  
ServletRequestListener：对请求对象的初始化和销毁进行监听；  
ServletRequestAttributeListener:对请求对象属性的添加、删除和替换进行监听。  
#### 用途  
可以使用监听起监听客户端的请求、服务端的操作等。通过监听器，可以自动触发一些动作，比如监听在线的用户数量，统计网站访问量、网站访问监控等。  
## 0x02 Tomcat基础介绍  
### 什么是Tomcat  
大概可以通过对标Apache来看一下  
Apache是Web服务器(静态解析，如HTML)，Tomcat是Java应用服务器(动态解析，如JSP)  
Tomcat只是一个servlet(jsp也可以翻译成servlet)容器，可以认为是Apache的扩展，但是可以独立于Apache运行。  
一句话概括一下，就是Web服务器，比较不稳定，但是业务能力比较强。  
### Tomcat与Servlet的关系  
根据上面的基础知识可以知道Tomcat是Web应用服务器，是一个Servlet/JSP容器，而Servlet容器从上到下分别是`Engine`、`Host、`Context`、`Wrapper`。  
在Tomcat中Wrapper代表一个独立的`servlet`实例，StandardWarpper是Warpper接口的标准实现类(StandardWarpper的主要任务就是载入Servlet类并且进行实例化)，同时其从·ContainerBase类继承过来，表示他是一个容器，只是他是最底层的容器，不能再含有任何的子容器了，且其父容器只能是context。而我们在也就是需要这里去载入我们自定义的Servlet加载我们的内存马  
## 0x03 Tomcat架构  
Tomcat的框架如下图所示，主要有`server`、`service`、`connector`、`container`四个部分  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409131152543.png)
图中可以看出Tomcat的心脏是两个组件：Connector和Container：  
Connector主要负责对外交流，进行Socket通信(基于TCP/IP)，解析HTTP报文，对应下图中的http服务器；  
Container主要处理Connector接受的请求，主要处理内部事务，加载和管理Servlet，由Servlet具体负责处理Request请求，对应下图中的servlet容器。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409131155172.png)
### server  
即服务器，代表整个Tomcat服务器，它能够提供一个接口让其他程序能够访问到这个Service集合、同时要维护它所包含的所有`Service`的生命周期，包括如何初始化、如何结束服务、如何找到别人要访问的`Service`。还有其他的一些次要的任务。  
一个Tomcat只有一个Server，Server中包含至少一个`Service`组件，用于提供具体服务。  
### service  
Service主要是为了关联Connector和Container，同时会初始化它下面的其他组件，在Connector和Container外面多包一层，把它们组装在一起，向外面提供服务，一个Service可以设置多个Connector，但是只能有一个Container容器。  
Tomcat 中 Service 接口的标准实现类是 StandardService ，它不仅实现了 Service 借口同时还实现了 Lifecycle 接口，这样它就可以控制它下面的组件的生命周期了  
### conector  
Connector组件是Tomcat中两个核心组件之一，它的主要任务是负责接收浏览器发过来的tcp连接请求，创建一个Request和Response对象分别用于和请求短交换数据，然后会产生一个线程来处理这个请求并把产生的Request和Response对象传给处理这个请求的线程，处理这个请求的线程就是Container组件要做的事了。  
根据运行的逻辑图，我们也能看到连接器 connector 主要有三个功能：  
	socket 通信    
	解析处理应用层协议，如将 socket 连接封装成 request 和 response 对象，后续交给 Container 来处理    
	将 Request 转换为 ServletRequest，将 Response 转换为 ServletResponse  
#### Adapter组件  
由于协议的不同，Tomcat 定义了自己的 Request 类来存放请求信息，但是这个不是标准的 ServletRequest。于是需要使用 Adapter 将 Tomcat Request 对象转成 ServletRequest 对象，然后就能调用容器的 service 方法。  
简而言之，Endpoint 接收到 Socket 连接后，生成一个 SocketProcessor 任务提交到线程池进行处理，SocketProcessor 的 run 方法将调用 Processor 组件进行应用层协议的解析，Processor 解析后生成 Tomcat Request 对象，然后会调用 Adapter 的 Service 方法，方法内部通过如下代码将 Request 请求传递到容器中。  
```java  
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);  
```  
一个总的 Tomcat Connector 功能如图所示  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409131203729.png)
### Container  
Container（又名Catalina）用于处理Connector发过来的servlet连接请求，它是容器的父接口，所有子容器都必须实现这个接口，Container 容器的设计用的是典型的责任链的设计模式，它有四个子容器组件构成，分别是：Engine、Host、Context、Wrapper，这四个组件不是平行的，而是父子关系，Engine 包含 Host，Host 包含 Context，Context 包含 Wrapper。  
Tomcat 设计了 4 种容器: `Engine、Host、Context、Wrapper` ，这四种容器是父子关系  
- Engine: 最顶层容器组件，可以包含多个 Host。实现类为 `org.apache.catalina.core.StandardEngine`  
- Host: 代表一个虚拟主机，每个虚拟主机和某个域名 Domain Name 相匹配，可以包含多个 Context。实现类为 `org.apache.catalina.core.StandardHost`  
- Context: 一个 Context 对应于一个 Web 应用，可以包含多个 Wrapper。实现类为 `org.apache.catalina.core.StandardContext`  
- Wrapper: 一个 Wrapper 对应一个 Servlet。负责管理 Servlet ，包括 Servlet 的装载、初始化、执行以及资源回收。实现类为 `org.apache.catalina.core.StandardWrapper`  
通常一个 Servlet class 对应一个 Wrapper，如果有多个 Servlet 就可以定义多个 Wrapper，如果有多个 Wrapper 就要定义一个更高的 Container。  
a.com和b.com分别对应着两个Host  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409131205004.png)
每一个 Context 都有唯一的 path。这里的 path 不是指 servlet 绑定的 WebServlet 地址，而是指独立的一个 Web 应用地址。就好比 Tomat 默认的 / 地址和 /manager 地址就是两个不同的 web 应用，所以对应两个不同的 Context。要添加 Context 需要在 server.xml 中配置 docbase。  
如下图所示， 在一个 web 应用中创建了 2 个 servlet 服务，WebServlet 地址分别是 /Demo1 和 /Demo2 。 因为它们属于同一个 Web 应用所以 Context 一样，但访问地址不一样所以 Wrapper 不一样。 /manager 访问的 Web 应用是 Tomcat 默认的管理页面，是另外一个独立的 web 应用， 所以 Context 与前两个不一样。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409131207807.png)
## 0x04 Tomcat的类加载机制  
由于Tomcat中有多个WebApp同时要确保之间相互隔离，所以Tomcat的类加载机制也不是传统的双亲委派机制。  
Tomcta自定义的类加载器WebAppClassloader为了确保隔离多个WebApp之间相互隔离，所以打破了双亲委派机制。每个WebApp用一个独有的ClassLoader实例来优先处理加载。它首先尝试自己加载某个类，如果找不到再交给父类加载器，其目的是优先加载Web应用自己定义的类。  
同时为了防止 WEB 应用自己的类覆盖 JRE 的核心类，在本地 WEB 应用目录下查找之前，先使用 ExtClassLoader（使用双亲委托机制）去加载，这样既打破了双亲委托，同时也能安全加载类。  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/java%E5%86%85%E5%AD%98%E9%A9%AC-%E5%9F%BA%E7%A1%80%E5%86%85%E5%AE%B9%E5%AD%A6%E4%B9%A0/  

