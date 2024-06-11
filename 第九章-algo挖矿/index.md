# 第九章-Algo挖矿

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```help  
服务器场景操作系统 Linux  
服务器账号密码 root password  ssh端口222  
```  
## 0x01  
1、通过本地 PC SSH到服务器并且分析黑客的 IP 为多少,将黑客 IP 作为 FLAG 提交;  
`cat /var/log/auth.log.1`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406060023084.png)
`flag{172.20.10.3}`  
2、通过本地 PC SSH到服务器并且分析黑客挖矿程序反链 IP 为多少,将黑客挖矿程序反链的 IP 作为 FLAG 提交;  
一般在定时任务  
`cat /var/spool/cron/crontabs/root`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406060025164.png)
这个就是挖矿木马，运行一下看看  
`lsof /usr/bin/dhpcd`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406060028719.png)
`pid`是`765`  
`netstat -tunlap | grep 765`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406060030406.png)
这几个 IP 应该是上线的 c2和挖矿程序反链 IP，但都不是最终`flag`，  
挖矿程序一直在更新，所以反链ip也有变化，flag 仍然是之前的ip  
`flag{139.99.125.38}`  
3、通过本地 PC SSH到服务器并且分析黑客权限维持文件的md5,将文件的 MD5(md5sum /file) 作为 FLAG 提交;  
做了权限维持的应该就是上面那个定时任务  
`flag{7b9a3a8a9e47e5c9675278420e6e7fa0}`  
4、通过本地 PC SSH到服务器并且分析黑客的用户名为什么,将黑客的用户名作为 FLAG 提交;  
`cat /etc/passwd`里面并没有用户  
`cat /root/.ssh/authorized_keys`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406060039487.png)
`flag{otto}`  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%AC%AC%E4%B9%9D%E7%AB%A0-algo%E6%8C%96%E7%9F%BF/  

