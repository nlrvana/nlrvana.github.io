# 第二章-日志分析-Redis应急响应

  
  
&lt;!--more--&gt;  
## 0x00 靶机信息  
```bash  
服务器场景操作系统 Linux  
服务器账号密码 root xjredis  
```  
## 0x01  
1、通过本地 PC SSH到服务器并且分析黑客攻击成功的 IP 为多少,将黑客 IP 作为 FLAG 提交;  
查看日志  
在`/var/log`下存在`redis.log`文件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282010325.png)
`192.168.100.20`进行了主从复制加载了`exp.so`文件  
2、通过本地 PC SSH到服务器并且分析黑客第一次上传的恶意文件,将黑客上传的恶意文件里面的 FLAG 提交;  
通过上面得到了恶意文件名`exp.so`，利用`find`进行查找，  
`find / -name &#34;exp.so&#34;`，查找到在根目录下载下来  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282014388.png)
`flag{XJ_78f012d7-42fc-49a8-8a8c-e74c87ea109b}`  
3、通过本地 PC SSH到服务器并且分析黑客反弹 shell 的IP 为多少,将反弹 shell 的IP 作为 FLAG 提交;  
`crontab -l`查看定时任务  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282016636.png)
4、通过本地 PC SSH到服务器并且溯源分析黑客的用户名，并且找到黑客使用的工具里的关键字符串(flag{黑客的用户-关键字符串} 注关键字符串 xxx-xxx-xxx)。将用户名和关键字符串作为 FLAG提交  
`find / -name &#34;authorized_keys&#34;`  
在`ssh`里面发现用户  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282018005.png)
`xj-test-user`  
到`github`上面搜索一下这个用户  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282022818.png)
在里面发现了`redis-rogue-getshell`工具  
看一下历史版本，发现关键字  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282023212.png)
`flag{xj-test-user-wow-you-find-flag}`  
5、通过本地 PC SSH到服务器并且分析黑客篡改的命令,将黑客篡改的命令里面的关键字符串作为 FLAG 提交;  
命令替换，查看一下`/usr/bin`  
`ls -al /usr/bin`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282026849.png)
发现了`ps`和`ps_`，直接打开查看一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282027519.png)
`flag{c195i2923381905517d818e313792d196}`  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E4%BA%8C%E7%AB%A0%E6%97%A5%E5%BF%97%E5%88%86%E6%9E%90-redis%E5%BA%94%E6%80%A5%E5%93%8D%E5%BA%94/  

