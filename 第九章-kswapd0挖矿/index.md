# 第九章-Kswapd0挖矿

  
  
&lt;!--more--&gt;  
1、通过本地 PC SSH到服务器并且分析黑客的 IP 为多少,将黑客 IP 作为 FLAG 提交;  
`cat /var/log/auth.log.1`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406062309145.png)
`flag{183.164.3.252}`  
2、通过本地 PC SSH到服务器并且分析黑客的用户名为什么,将黑客的用户名作为 FLAG 提交;  
`cat /etc/passwd`中并无可疑用户  
`cat /root/.ssh/authorized_keys`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406062311020.png)
`flag{mdrfckr}`  
3、通过本地 PC SSH到服务器并且分析黑客权限维持文件的md5,将文件的 MD5(md5sum /file) 作为 FLAG 提交;  
权限维持一般是定时任务  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406062312009.png)
看一下这几个文件是否是恶意文件， 
利用`stat`看一下时间  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406062314833.png)
只有`/root/.X291-unix/.rsync/c/aptitude`符合 Linux 主机被入侵的时间。  
可以确定定时任务做了权限维持  
`md5sum /var/spool/cron/crontabs/root`  
`flag{cc84a750dcfb988861c8bf90aa15c039}`  
  
## 定时任务  
```  
我们经常使用的是crontab命令是cron table的简写，它是cron的配置文件，也可以叫它作业列表，我们可以在以下文件夹内找到相关配置文件。    
/var/spool/cron/ 目录下存放的是每个用户包括root的crontab任务，每个任务以创建者的名字命名    
/etc/crontab 这个文件负责调度各种管理和维护任务。    
/etc/cron.d/ 这个目录用来存放任何要执行的crontab文件或脚本。    
我们还可以把脚本放在/etc/cron.hourly、/etc/cron.daily、/etc/cron.weekly、/etc/cron.monthly目录中，让它每小时/天/星期、月执行一次  
```  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%AC%AC%E4%B9%9D%E7%AB%A0-kswapd0%E6%8C%96%E7%9F%BF/  

