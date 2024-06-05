# 第四章-Windows实战-Wordpress

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```  
rdp 端口 3389  
administrator xj@123456  
phpstudy  
```  
## 0x01  
1、请提交攻击者攻击成功的第一时间  
日志存储位置`C:\phpstudy_pro\Extensions\Nginx1.15.11\logs\access.log`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406050009828.png)
黑客一直在爆破后台，从最后一个`POST`登陆请求之后，访问`/manage/welcome.php`就返回`200`了  
所以时间是`29/Apr/2023:22:45:23`  
`flag{2023:04:29 22:45:23}`  
2、请提交攻击者的浏览器版本  
`flag{Firefox/110.0}`  
3、请提交攻击者目录扫描所使用的工具名称  
在日志中，发现扫描器特征  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406050013020.png)
`flag{Fuzz Faster U Fool}`  
4、找到攻击者写入的恶意后门文件，提交文件名  
使用`D盾`扫描一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406050020057.png)
`flag{C:\phpstudy_pro\WWW\.x.php}`  
5、找到攻击者隐藏在正常web应用代码中的恶意代码，提交该文件名  
同上，已经在`D盾`中显示出来  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406050021990.png)
`flag{C:\phpstudy_pro\WWW\usr\themes\default\post.php}`  
6、请指出可疑进程采用的自动启动的方式，启动的脚本的名字  
查看自启动文件目录  
`C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp`  
`C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup`  
并无可疑文件，接下来查看`Temp`目录和`Windows`目录  
在`Windows`目录下，发现可疑文件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406050043177.png)
`x.bat`启动了`360.exe`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406050043541.png)
将`360.exe`上传微步看看，发现是木马  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406050045022.png)
`flag{x.bat}`  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%AC%AC%E5%9B%9B%E7%AB%A0-windows%E5%AE%9E%E6%88%98-wordpress/  

