# 第六章-流量特征分析-Waf上的截获的黑客攻击流量

  
  
&lt;!--more--&gt;  
## 0x00环境分析  
```question  
1.黑客成功登录系统的密码 flag{xxxxxxxxxxxxxxx}  
2.黑客发现的关键字符串 flag{xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx}  
3.黑客找到的数据库密码 flag{xxxxxxxxxxxxxxxx}  
```  
## 0x01  
1、黑客成功登录系统的密码  
直接过滤`http`请求，找到登陆请求，进行过滤  
`_ws.col.info == &#34;POST /admin/login.php?rec=login HTTP/1.1  (application/x-www-form-urlencoded)&#34; || http.response.code == 302`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406032212666.png)
`flag{admin!@#pass123}`  
2、黑客发现的关键字符串  
`/images/article/a.php`是webshell  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406032216612.png)
所以关键字符串大概率在服务器中，黑客可以通过`webshell`看到  
`http.request.uri==&#34;/images/article/a.php&#34; || http.response.code==200`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406032223283.png)
`flag{87b7cb79481f317bde90c116cf36084b}`  
3.黑客找到的数据库密码  
在`webshell`流量中找到了操作数据库的流量  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406032240565.png)
```php  
&lt;?php  
$conf = $m ? stripslashes($_POST[&#34;z1&#34;]) : $_POST[&#34;z1&#34;];  
$ar = explode(&#34;choraheiheihei&#34;, $conf);  
$dbn = $m ? stripslashes($_POST[&#34;z2&#34;]) : $_POST[&#34;z2&#34;];  
$sql = base64_decode($_POST[&#34;z3&#34;]);  
var_dump(&#34;host:&#34;.$ar[0],&#34;username:&#34;.$ar[1],&#34;password:&#34;.$ar[2]);  
?&gt;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406032246315.png)
得到密码  
`flag{e667jUPvJjXHvEUv}`  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E5%85%AD%E7%AB%A0-%E6%B5%81%E9%87%8F%E7%89%B9%E5%BE%81%E5%88%86%E6%9E%90-waf-%E4%B8%8A%E7%9A%84%E6%88%AA%E8%8E%B7%E7%9A%84%E9%BB%91%E5%AE%A2%E6%94%BB%E5%87%BB%E6%B5%81%E9%87%8F/  

