# 第九章-Blueteam的小心思

  
  
&lt;!--more--&gt;  
## 0x00 环境分析   
```help  
服务器场景操作系统 Linux  
服务器账号密码 root qi5qaz  
  
任务环境说明  
    注：进去后执行  sed -i &#39;s/Listen 80/Listen 9999/&#39; /etc/apache2/ports.conf &amp;&amp; service apache2 restart  
开放题目  
    漏洞修复  
```  
## 0x01  
1、攻击者通过什么密码成功登录了网站的后台？提交密码字符串的小写md5值，格式flag{md5}。  
查找一下流量包  
`find / -name &#34;*.pcap&#34;`  
`http contains &#34;login.php&#34;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081814941.png)
`Aa12345^`  
`flag{d63edb0e9df4cf411398e3658c0237e0}`  
2、攻击者在哪个PHP页面中成功上传了后门文件？例如upload.php页面，上传字符串&#34;upload.php&#34;的小写md5值，格式flag{md5}。  
`http contains &#34;boundary&#34;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081816546.png)
`pluginmgr.php`  
`flag{b05c0be368ffa72e6cb2df7e1e1b27be}`  
3、找到攻击者上传的webshell文件，提交该文件的小写md5值，格式flag{md5}。  
D盾扫描  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081822302.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081825198.png)
`flag{a097b773ced57bb7d51c6719fe8fe5f5}`  
4、攻击者后续又下载了一个可执行的后门程序，提交该文件的小写md5值，格式flag{md5}。  
看一下webshell的时间  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081831419.png)
```  
find / -newerct &#39;2023-11-18 07:30:00&#39; ! -newerct &#39;2023-11-19 07:30:00&#39; ! -path &#39;/proc/*&#39; ! -path /&#39;sys/*&#39; ! -path &#39;/run/*&#39; -type f -exec ls -lctr --full-time {} \&#43; 2&gt;/dev/null | grep www-data  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081838262.png)
`md5sum &#34;/var/www/html/plugins/.       /is.world&#34;`  
`flag{ee279c39bf3dcb225093bdbafeb9a439}`  
5、攻击者创建了后门用户的名称是？例如attack恶意用户，上传字符串&#34;attack&#34;的小写md5值，格式flag{md5}。  
`cat /etc/passwd`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081842523.png)
`md5 -s knowledgegraphd`  
`flag{4cda3461543c9a770a3349760594facd}`  
6、攻击者创建了一个持久化的配置项，导致任意用户登录就会触发后门的连接。提交该配置项对应配置文件的小写md5值，格式flag{md5}。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081848678.png)
`cat /etc/profile`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081848969.png)
发送了其中恶意文件  
`md5sum /etc/profile`  
`flag{65bf3e4a9ac90d75ec28be0317775618}`  
7、攻击者创建了一个持久化的配置项，导致只有root用户登录才会触发后门的连接。提交该配置项对应配置文件的小写md5值，格式flag{md5}。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081851134.png)
`cat /root/.bashrc`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081851492.png)
`md5sum /root/.bashrc`  
`flag{4acc9c465eeeb139c194893ec0a8bcbc}`  
8、攻击者加密了哪个数据库？提交数据库的文件夹名，例如user数据库对应存放位置为user文件夹，上传字符串&#34;user&#34;的小写md5值，格式flag{md5}。  
连接一下数据库  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081855595.png)
`root/mysql123`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081900035.png)
其中`JPMorgan Chase`数据库无法打开  
在`/var/lib/mysql`中存储着数据库文件  
`md5 -s JPMorgan@0020Chase`  
`flag{0928a5424aa6126e5923980ca103560e}`  
9、解密数据库，提交Harper用户对应Areer的值。提交Areer值的小写md5值，格式flag{md5}。  
`find / -type f -newer /var/www/html/plugins/cpg.php ! -newer /var/lib/mysql/JPMorgan@0020Chase/Balance.frm 2&gt;/dev/null`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081908838.png)
其中`clockup.php`文件正是加密文件  
写出一个解密脚本  
```php  
&lt;?php  
$currentDate = &#34;2023-11-18&#34;;  
$key = md5($currentDate);  
$iv = substr(hash(&#39;sha256&#39;, &#34;DeepMountainsGD&#34;), 0, 16);  
$filePath = &#34;/var/lib/mysql/JPMorgan@0020Chase&#34;;  
$files = scandir($filePath);  
foreach ($files as $file) {  
    if ($file != &#34;.&#34; &amp;&amp; $file != &#34;..&#34;) {  
        $fullPath = $filePath . &#39;/&#39; . $file;  
        $content = file_get_contents($fullPath);  
        $encryptedContent = openssl_decrypt($content, &#39;aes-256-cbc&#39;, $key, 0, $iv);  
        file_put_contents($fullPath, $encryptedContent);  
    }  
}  
?&gt;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081915740.png)
`flag{8fd82b8864d71ed7fa12b59e6e34cd1c}`  
10、因为什么文件中的漏洞配置，导致了攻击者成功执行命令并提权。提交该文件的小写md5值，格式flag{md5}。  
`find / -user root -perm -4000 2&gt;/dev/null`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081920944.png)
`cat /etc/sudoers`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081922305.png)
`sudo`提权  
`md5sum /etc/sudoers`  
`flag{6585817513b0ea96707ebb0d04d6aeff}`  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%AC%AC%E4%B9%9D%E7%AB%A0-blueteam-%E7%9A%84%E5%B0%8F%E5%BF%83%E6%80%9D/  

