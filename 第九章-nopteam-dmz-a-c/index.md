# 第九章-NOPTeamDmz-a-C

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```help  
第九章-NOP Team-A  
靶机来源 NOP Team 鸣谢 NOP Team  
可以采用本地搭建或者是云端调度  
搭建链接 https://mp.weixin.qq.com/s/p1KNDU84PXOv-fqo8TfHNQ  
账户 root  密码 nopteama  
请启动禅道服务  /opt/zbox/zbox -ap 8081 &amp;&amp; /opt/zbox/zbox start  
  
通过dmz-A攻击者遗留下来的攻击结果，分析结果使用里面的用户名和密码可以对此环境进行登陆，自行保存id_rsa密钥文件  
ssh -p 222 xxx@xxx.xxxx.xxx  
  
可以通过dmz-B泄露的密钥将其保存到本地，然后进行登陆ssh  
ssh -i xxx_xxx xxxx@xxxx.xxxx.xxxx  
```  
## 0x01  
1、请提交禅道的版本号，格式: flag{xx.xxx.xxxx}  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101438634.png)
`flag{18.0.beta1}`  
2、分析攻击者使用了什么工具对内网环境进行了信息收集，将该工具名称提交 flag{xxxx}  
查看`apt.pcap`  
利用了`icmp echo`探测了存活  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101453469.png)
发送了一次完整的`TCP`请求  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101456969.png)
并且还有漏洞扫描的流量，特征符合`fscan`  
`flag{fscan}`  
3、攻击者攻击服务器得到shell以后，是处于哪个用户下做的操作，将该用户名提交 flag{xxxx}  
18.0.beta1下存在一个RCE漏洞  
https://blog.csdn.net/qq_41904294/article/details/128838423  
搜索一下特征`http contains &#34;SCM=Subversion&#34;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101502814.png)
`flag{nobody}`  
4、攻击者扫描到内网 DMZ-B机器的账户密码为多少格式 flag{root:root}  
搜索一下是否有`fscan`  
`find / -name &#34;*fscan*&#34;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101503462.png)
直接看扫描结果  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101504148.png)
`flag{admin:123456}`  
1、攻击者通过DMZ-A登陆到DMZ-B机器里，在上面发现了DMZ-C机器里的一个密钥，通过某服务直接进行了登陆，请将服务名与登陆的用户名提交 &amp;lt;格式：flag{ftp:anonymous}  
通过上面拿到的密码`admin:123456`  
通过查看历史命令  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101540248.png)
得到通过`ssh`连接到了`dmz-C`  
`flag{ssh:deploy}`  
将私钥拿下来  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101542227.png)
1、攻击者上传了一个挖矿程序，请将该挖矿程序的名称提交，格式 &lt;flag{xxxxxx}&gt;  
`history`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101816851.png)
其中挖矿程序在`/opt`目录  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101817053.png)
`flag{xmrig}`  
2、攻击者上传了一个挖矿程序，请将该挖矿的地址提交，格式 &lt;flag{xxxxxx}&gt;  
在`config.json`里面  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101818371.png)
`flag{xmrs1.pool.cn.com:55503}`  
3、攻击者上传了一个挖矿程序，但由于DMZ-C机器是不出网的，所以攻击者通过了一种方式将流量转发了出去，请将转发的目标端口提交，格式 &lt;flag{xxxxxx}&gt;  
在`/opt/client`目录下存在`frp`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101821699.png)
`flag{1080}`  
4、攻击者上传了一个挖矿程序，但由于DMZ-C机器是不出网的，所以攻击者通过了一种方式将流量转发了出去，请将用来做转发的工具的名称提交，格式 &lt;flag{xxxxxx}&gt;  
`flag{frpc}`  
5、攻击者最后通过某配置文件配置错误，从而直接可以拥有root用户权限，请将错误配置的那一行等于号后面的内容(不含空格)提交，格式 &lt;flag{xxxxxxx}&gt;  
`sudo -l`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101825399.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101825017.png)
`sudo cat /etc/sudoers`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101827513.png)
`flag{(ALL:ALL)NOPASSWD:ALL}`  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%AC%AC%E4%B9%9D%E7%AB%A0-nopteam-dmz-a-c/  

