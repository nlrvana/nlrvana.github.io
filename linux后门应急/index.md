# Linux后门应急

  
  
&lt;!--more--&gt;  
## 0x00 环境  
```bash  
SSH连接：ssh user@ip -p 222  
账号密码：user/Atmbctfer!  
```  
## 0x01 解题记录  
#### 1、主机后门用户名称：提交格式如：flag{backdoor}  
`cat /etc/passwd`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411261659370.png)
`flag{backdoor}`  
#### 2、主机排查项中可以发现到flag{}内以i开头的flag，如flag{ixxxxxxx}  
`ps -aux`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411261705058.png)
`flag{infoFl4g}`  
#### 3、主机排查发现9999端口是通过哪个配置文件如何开机启动的，如/etc/crontab则填写 /etc/crontab 的md5 ，提交方式示例：flag{md5}  
`/etc/systemd/system`查看自启动服务的相关配置文件  
打开`rc-local.service`，执行了`/etc/rc.d/rc.local`文件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411261722050.png)
解码得到  
`while true;do nohup nc -lvp 9999 -c &#34;flag{infoFl4g}&#34; 2&gt;&amp;1 ;sleep 1;done;`  
`flag{cf8a978fe83579e2e20ec158524d8c06}`  
#### 4、黑客3s做了记录所有用户的每次登陆的密码的手段，flag为黑客记录的登陆密码日志路径md5,提交方式示例:flag{md5(路径)}  
在`/tmp`目录下发现`.sshlog`文件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411261727717.png)
`flag{8997d5a1b8dcca5a4be75962250959f7}`  
#### 5、给出使用了/bin/bash 的RCE后门进程名称&#43;端口号 如进程名称为sshd，端口号为22，则flag{sshd22)  
在`docker-compose-app.service`文件，还启动了一个`/usr/lib/python3.7/site-packages/docker/startup.sh`脚本  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411261731066.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411261733832.png)
base64解密出来是一个Reverse Shell  
`flag{python38080}`  
#### 6、找出开机启动的后门服务名称MD5，提交flag(md5(服务名)}  
看一下开机自启动服务  
`systemctl list-unit-files --type=service --state=enabled`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411261736415.png)
`flag{5213e47de16522f1dc3f9e9ecc0ab8b0}`  
#### 7、渗透提权获得root目录下的flag  
利用`docker`进行提权  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411261741004.png)
`user`用户在`docker`组里面，而且`docker`有`root`权限  
```bash  
docker run -v /:/mnt -it alpine  
chroot /mnt bash  
```  
`flag{ATMB_root}`  
#### **8、黑客3s埋了一个flag在权限维持过程中的地方，可以发现flag()括号内的首字母是c开头，如flag{cxxxxxxx)**  
打开所有`crontab-jobs`  
```  
cat /etc/cron* /etc/at* /etc/anacrontab /var/spool/cron/crontabs/root 2&gt;/dev/null | grep -v &#34;^#&#34;  
```  
Orz但是并未发现 flag  
这里需要查看`crontab -e`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411272327964.png)
`flag{cr0nt4b_IRfind}`  
#### 9、黑客3s做了一个root用户执行cat命令就删除文件的操作，请发现删除了什么文件将文件名作为flag提交  
检测是否存在`LD_PRELOAD`劫持  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411261816367.png)
追踪一下`/bin/cat`命令  
`strace -f -e trace=file /bin/cat`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411261817671.png)
加载了恶意的`so`文件  
`strings`一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411261817522.png)
删除了`.bash_history`文件  
`flag{.bash_history}`  
#### 10、黑客3s很执着清理痕迹，并做了一个持续删痕迹的手段，请发现手段并给出删除的完整黑客删除命令的md5,如flag{md5(rm-f/var/log/ssh.log &gt;/dev/stdout)}  
还是上面的答案  
`flag{rm -rf ~/.bash_history &gt;/dev/null 2&gt;&amp;1}`  
`flag{b0f531b39d88d4f603fc89bd4dd2c0aa}`  
#### 11、黑客3s设置了一个万能密码后门使得这一个万能密码可以以所有用户身份登陆，也不影响原来密码使用。请发现这个万能密码，提交fag格式为flag{万能密码)  
万能密码首先想到的是`openssh`替换或者`pam`替换  
`pam_unix.so`负责与本地 Unix 系统认证进行交互，主要用于处理用户的身份验证，如密码验证  
```Shell  
find /lib /lib64 /usr/lib /usr/lib64 -name pam_unix.so  
```  
时间和其他文件不一样，很明显被修改过  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411261823557.png)
拖下来放入 IDA 中进行反编译  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411262018923.png)
看到密码是`ATMB6666`  
`flag{ATMB6666}`  

---

> Author: N1Rvana  
> URL: http://localhost:1313/linux%E5%90%8E%E9%97%A8%E5%BA%94%E6%80%A5/  

