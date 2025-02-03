# 实战-行业攻防应急响应

  
  
&lt;!--more--&gt;  
####  1. 根据流量包分析首个进行扫描攻击的IP是  
被攻击的主机 ip 是`192.168.0.211`  
这里过滤一下  
`ip.dst == 192.168.0.211`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291404839.png)
这是正常的`ping`包，继续往下看  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291407172.png)
疑似存在端口探测行为，这里继续进行过滤一下  
`ip.dst == 192.168.0.211 &amp;&amp; ip.src== 192.168.0.223`  
可以确定为端口探测&#43;目录扫描  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291409616.png)
攻击ip是`192.168.0.223`  
`flag{192.168.0.223}`  
#### 2. 根据流量包分析第二个扫描攻击的IP和漏扫工具，以flag{x.x.x.x&amp;工具名}  
`ip.dst == 192.168.0.211 &amp;&amp; ip.src != 192.168.0.223 &amp;&amp; http`  
其中`192.168.0.200`一直在做漏洞扫描  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291413098.png)
  
查找一下漏洞扫描器特征，从UA、字典特征，行为特征、DNSlog地址、url中的某些地址查看  
其中 url 地址中部分为`bxss.me`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291416608.png)
这是`awvs(acunetix)`扫描器的特征  
`flag{192.168.0.200&amp;acunetix}`  
#### 3、提交频繁爆破密钥的IP及爆破次数，以flag{ip&amp;次数}提交  
服务器内存在ruoyi框架，存在`shiro`，这里爆破的是`shiro`密钥  
这里一直在`/login`路由爆破  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291423964.png)
过滤一下  
`ip.dst == 192.168.0.211 &amp;&amp; ip.src == 192.168.0.226 &amp;&amp;http.request.uri==&#34;/login&#34;`  
去除开头三次  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291426863.png)
一共爆破了`1068`次  
`flag{192.168.0.226&amp;1068}`  
#### 4、提交攻击者利用成功的密钥，以flag{xxxxx}提交  
`ip.dst == 192.168.0.211 &amp;&amp; ip.src != 192.168.0.223 &amp;&amp; http.request.uri contains login`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291429124.png)
其中`242`在`226`之后，并且一开始就在爆破密钥，这里过滤一下  
`ip.dst == 192.168.0.211&amp;&amp;ip.src==192.168.0.242`  
这里并没有爆破成功，而是下载了`/actuator/heapdump`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291433136.png)
这里下载下来，进行分析  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291439678.png)
`flag{c&#43;3hFGPjbgzGdrC&#43;MHgoRQ==}`  
#### 5、提交攻击者获取到的所有服务的弱口令，多个以&amp;提交  
分析`heapdump`即可，得到  
`flag{ruoyi123&amp;admin123&amp;123456}`  
  
#### 6、根据应急响应方法，提交利用漏洞成功的端口，多个以&amp;连接  
根据上面的分析，得知漏洞利用成功端口是  
分别是泄露了`heapdump`和`ruoyi`框架漏洞利用  
`flag{9988&amp;12333}`  
#### 7、根据流量包分析，提交攻击者利用密钥探测成功的dnslog地址  
筛选 DNS 协议和主机  
`dns&amp;&amp;ip.src==192.168.0.211`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291442558.png)
`flag{1dvrle.dnslog.cn}`  
#### 8、根据流量包分析，提交攻击者反弹shell的地址和端口  
漏洞利用是在`ruoyi`进行的，直接过滤http流量即可  
`ip.dst==192.168.0.211&amp;&amp;ip.src==192.168.0.242&amp;&amp;http`  
从`heapdump`往后看即可  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291446929.png)
直接找到最后一个  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291459902.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291500197.png)
`flag{192.168.0.251:8888}`  
#### 9、攻击者在主机放置了fscan(已改名)，经扫描拿下一台永恒之蓝漏洞主机，以此为线索进行提交fscan绝对路径  
因为是反弹 shell，在TCP/IP协议中，会话需要经过三次握手和四次挥手，SYN，ACK包中存在明文流量  
`ip.dst==192.168.0.211&amp;&amp;ip.src==192.168.0.251&amp;&amp;tcp.port==8888 &amp;&amp;tcp.flags.syn == 1 &amp;&amp; tcp.flags.ack == 1`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291511445.png)
`flag{/opt/.f/.s/.c/.a/.n}`  
#### 10、另类方法：提交此fscan工具的MD5值  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291514175.png)
`flag{b8053bcd04ce9d7d19c7f36830a9f26b}`  
#### 11、攻击者为了权限维持，在主机放置了仿真远控工具，需提交此远控工具的下载地址  
权限位置，一般是定时任务  
`cat /etc/cron* /etc/at* /etc/anacrontab /var/spool/cron/crontabs/root 2&gt;/dev/null | grep -v &#34;^#&#34;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291517462.png)
每隔十分钟执行一次`.qxwc.sh`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291517157.png)
`flag{http://zhoudinb.com:12345/qxwc.sh}`  
#### 12、攻击者就知道你会这样找到，所以又创建了一条相关的脚本，使用其他方法进行下载，提交脚本的绝对路径  
Linux 中还可以通过开机自启动，来做权限维持，通过`systemctl`查看  
`/etc/systemd/system/`创建任务是通过此目录下进行定位文件名创建任务名  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291520333.png)
`happy.service`服务可疑，打开查看  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/20241129152107.png)
`flag{/home/security/upload/.CCC/.happy.sh}`  
#### 13、攻击者创建了一个隐藏用户，提交此用户的用户名  
 在`/root/.ssh/id_rsa.pub`下，发现可疑用户  
 ![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291524052.png)
`flag{xj1zhoudi@kali}`  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E5%AE%9E%E6%88%98-%E8%A1%8C%E4%B8%9A%E6%94%BB%E9%98%B2%E5%BA%94%E6%80%A5%E5%93%8D%E5%BA%94/  

