# 内网渗透-Windows基础信息收集(6)

  
  
&lt;!--more--&gt;  
## 收集网络和用户信息  
### 网络信息：  
分析网络连接情况，得到部分可用ip地址，根据ip地址可以做icmp(Ping检测)探测，可以为横向移动提供部分地址信息，同时分析能否与外网直接通信，是否可以3389，或者分析出本机代理情况，为后续做流量代理以及种植后门做准备。  
### nslookup  
```  
nslookup 域名 #查询域名情况  
nslookup -qt=type domain[dns-server] #查询其他记录  
```  
**qt后的type类型如下**  
```  
A  地址记录  
AAAA 地址记录  
AFSDB Andrew 文件系统数据库服务器记录  
ATMA ATM地址记录  
CNAME 别名记录  
HINHO 硬件配置记录，包括CPU、操作系统信息  
ISDN 域名对应的ISDN号码  
MB 存放指定邮箱的服务器  
MG 邮件组记录  
MINFO 邮件组和邮箱的信息记录  
MR 改名的邮箱记录  
MX 邮件服务器记录  
NS 名字服务器记录  
PTR 反向记录  
RP  负责人记录  
RT  路由穿透记录  
SRV TCP服务器信息记录  
TXT 域名对应的文本信息  
X25 域名对应的X.25地址记录  
```  
### 路由信息  
分析路由信息，查看本机路由情况，同时可以使用路由追踪在网络信息里发现的部分IP地址，得到其路由为如何走向，为分析做准备。  
`route print`打印路由信息  
`tracert`  
#### 查看wifi  
`netsh wlan show profiles`  
查看wifi密码  
`netsh wlan show profiles wifi名 key=clear`  
```  
常用的netsh命令  
设置配置文件属性:netsh WLAN set profileparameter name=&#34;&#34; connection=manual  
列出配置文件:netsh wlan show profile  
导出配置文件:netsh wlan export profile key=clear  
```  
### 网络代理  
```powershell  
reg query &#34;HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Internet Settings&#34;  
reg query &#34;HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Internet Settings&#34; /v &#34;ProxyServer&#34;  
查询window中设置的系统代理，即internet选项设置中的代理信息  
```  
### DNS缓存  
`ipconfig /displaydns`  
### 查看最近rdp连接过的主机  
```  
reg query &#34;HKEY_CURRENT_USER\Software\Microsoft\Terminal Server Client\Default&#34;  
query user 查询在线用户状态  
```  
### 查看回收站信息  
```  
cd C:\$Recycle.Bin\  
使用dir /A 查看隐藏文件  
S-1-xxx 分别对应不同用户的回收站  
```  
## Bloundhund  
### 采集信息  
`SharpHound.exe -c all` 进行数据采集  
生成的数据文件都在采集器所在的目录  
`powershell -exec bypass -command &#34;Import-Module&#34; ./SharpHound.ps1; Invoke-BloodHound -c all&#34;`  
或者`Invoke-BloodHound -CollectAllProperties`  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F-windows%E5%9F%BA%E7%A1%80%E4%BF%A1%E6%81%AF%E6%94%B6%E9%9B%866/  

