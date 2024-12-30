# 第九章-实战篇-运维杰克

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```txt  
服务器场景操作系统 Linux  
服务器账号密码 root root123  
result.pcap  
```  
## 0x01  
1、攻击者使用的的漏洞扫描工具有哪些(两个) flag{xxxan-xxxy}  
查询apache日志看一下哪两个ip访问最多  
`cut -d- -f 1 access.log.1 | sort -nr | uniq -c | sort -nr`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405292328165.png)
可以看到`192.168.150.1`和`192.168.150.2`  
利用`wireshark`查一下这两个ip的流量  
`ip.src_host == 192.168.150.1 &amp;&amp; tcp &amp;&amp; !ssh &amp;&amp; !http`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406051944143.png)
`options`固定长度`4 02 04 05 b4`，并且为`TCP`半连接，符合`nmap`扫描特征  
查询一下`HTTP`流量，发现了大量的漏洞扫描流量  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406051954465.png)
在`/apply_sec.cgi`中发现了`goby`特征，确定为`goby`扫描器  
`ip.src_host == 192.168.150.2 &amp;&amp; tcp &amp;&amp; !ssh &amp;&amp; !http`  
按照时间排序，可以看到开始发送了大量扫描端口的流量  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406051958989.png)
这里跟踪一下`22`端口  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406051959117.png)
可以看到是一次完整的`TCP`请求，  
利用了`icmp echo`探测存活  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406052001331.png)
同时也进行了大量的漏洞扫描  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406052002618.png)
漏洞特征为`fscan`  
`flag{fscan-goby}`  
2、攻击者上传webshell的绝对路径及User-agent  
`netstat -ano`看一下开放的业务  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406052007206.png)
压缩`/var/www/html`目录的文件  
放到`D盾`扫描  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406052024664.png)
打开文件看到反弹`ip`与`port`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406052024564.png)
在流量中找一下如何上传了这个文件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406052034390.png)
`User-Agent`是`my_is_user_agent`  
`flag{14bea1643a4b97600ba13a6dd5dbbd04}`  
3、攻击者反弹shell的IP及端口是什么  
`flag{192.168.150.110:5678}`  
4、攻击者利用提权攻击添加的用户，用户名是什么  
`cat /etc/passwd`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406052312025.png)
`flag{zhangsan}`  
4、攻击者留下了后门脚本，找到绝对路径(有SUID权限)  
`find / -perm 4000 -u=s -type f 2&gt;/dev/null`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406052315449.png)
`flag{/var/www/html/lot/admin/assets/vendor/.shell/.decodeshell.php}`  
5、攻击者留下了持续化监控和后门脚本，找到绝对路径  
根据题目描述，查看定时任务  
`crontab -l`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406052323024.png)
`cat /opt/.script/.script.sh`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406052323159.png)
`flag{/opt/.script/.script.sh}`  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E4%B9%9D%E7%AB%A0-%E5%AE%9E%E6%88%98%E7%AF%87-%E8%BF%90%E7%BB%B4%E6%9D%B0%E5%85%8B/  

