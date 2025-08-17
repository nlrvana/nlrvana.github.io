# 第五章-Linux实战-挖矿

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```help  
应急响应工程师在内网服务器发现有台主机 cpu 占用过高，猜测可能是中了挖矿病毒，请溯源分析，提交对应的报告给应急小组  
虚拟机账号密码 root websecyjxy web 端口为 8081  
1、黑客的IP是？ flag格式：flag{黑客的ip地址}，如：flag{127.0.0.1}  
2、黑客攻陷网站的具体时间是？ flag格式：flag{年-月-日 时:分:秒}，如：flag{2023-12-24 22:23:24}  
3、黑客上传webshell的名称及密码是？ flag格式：flag{黑客上传的webshell名称-webshell密码}，如：flag{webshell.php-pass}  
4、黑客提权后设置的后门文件名称是？ flag格式：flag{后门文件绝对路径加上名称}，如：flag{/etc/passwd}  
5、对黑客上传的挖矿病毒进行分析，获取隐藏的Flag  
```  
## 0x01  
1、黑客的IP是？  
`find / -name &#34;*.php&#34;`查找一个web目录  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406070000794.png)
根目录`/www/admin/websec_80/wwwroot/`  
日志目录`/www/admin/websec_80/log`  
查看日志`nginx_access_2023-12-22.log`，大部分都是一个ip  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406070003581.png)
`flag{192.168.10.135}`  
2、黑客攻陷网站的具体时间是？  
dedecms 很多都是后台漏洞，所以我们需要登录进后台查看  
弱口令`admin:12345678`  
在用户管理处发现了黑客添加的用户  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406070014596.png)
`flag{2023-12-22 19:08:34}`  
3、黑客上传webshell的名称及密码是？  
压缩源码，放到 D 盾里面扫描  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406070020943.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406070021875.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406070022534.png)
`flag{404.php-cmd}`  
4、黑客提权后设置的后门文件名称是？  
看一下`root`的`.bash_history`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406070025060.png)
设置了`find`命令  
`flag{/usr/bin/find}`  
5、对黑客上传的挖矿病毒进行分析，获取隐藏的Flag  
一般挖矿任务会设置在定时任务里面  
`cat /etc/crontab`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406070036392.png)
`find / -name &#34;ldm&#34;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406070038815.png)
`cat /etc/.cache/ldm`  
找到一串关键代码  
`nohup python2 -c &#34;import base64;exec(base64.b64decode(&#39;aW1wb3J0IHRpbWUKd2hpbGUgMToKICAgIHByaW50KCJmbGFne3dlYnNlY19UcnVlQDg4OCF9IikKICAgIHRpbWUuc2xlZXAoMTAwMCk=&#39;))&#34; &gt;/dev/null 2&gt;&amp;1`  
解密看一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406070041232.png)
`flag{websec_True@888!}`  
  
## Tips  
时间文件排查  
`find / -newerct &#39;2024-01-24 08:10:00&#39; ! -newerct &#39;2024-01-24 09:10:00&#39; ! -path &#39;/proc/*&#39; ! -path /&#39;sys/*&#39; ! -path &#39;/run/*&#39; -type f -exec ls -lctr --full-time {} \&#43; 2&gt;/dev/null`  
敏感目录排查  
`find /tmp ! -type d -exec ls -lctr --full-time {} \&#43; 2&gt;/dev/null`  
HOME目录排查  
`find $HOME ! -type d -exec ls -lctr --full-time {} \&#43; 2&gt;/dev/null`  
特权文件排查  
`find / -perm -u=s 2 -type f -ls &gt;/dev/null`  
所有进程排查  
`pstree -as`  
隐藏进程排查  
`ps -ef | awk &#39;{print}&#39; | sort | uniq &gt; 1`  
`ps -ef | awk &#39;{print}&#39; | sort | uniq &gt; 2`  
`diff 1 2`  
计划任务排查  
`find /var/spool/cron/ -type f -exec ls -lctr --full-time {} \&#43; 2&gt;/dev/null`  
`find /etc/*cron* -type f -exec ls -lctr --full-time {} \&#43; 2&gt;/dev/null`  
启动项排查  
`find /etc/rc.d/ -type f -exec ls -lctr --full-time {} \&#43; 2&gt;/dev/null`  
自启服务排查  
`chkconfig --list`  
`service --status-all`  
命令别名排查  
`alias`  
`find / -name *bashrc* -type f -exec ls -lctr --full-time {} \&#43; 2&gt;/dev/null`  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%AC%AC%E4%BA%94%E7%AB%A0-linux%E5%AE%9E%E6%88%98-%E6%8C%96%E7%9F%BF/  

