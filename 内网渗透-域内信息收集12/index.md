# 内网渗透-域内信息收集(12)

  
  
&lt;!--more--&gt;  
## 域信息  
```  
# 查询当前登录域及用户信息  
net config workstation  
```  
利用PowerSploit下的PowerView.ps1脚本  
```  
Import-Module PowerView.ps1  
Get-NetForest  
```  
## 查找域控  
### 查找时间服务器  
`net time /domain`  
https://github.com/PowerShellMafia/PowerSploit  
### 查看domain controllers组  
`net group &#34;domain controllers&#34; /domain`  
### nslookup查询  
`nslookup -qt=ns pkwsec.cn`  
### 查询域控详细信息  
```  
Import-Module PowerView.ps1  
Get-NetDomainController  
```  
##  查询域内用户  
```  
# 查询域内用户  
net group &#34;domain users&#34; /domain  
net user /domain  
# 查询指定域用户信息  
net user test /domain  
# 查询域用户详细信息  
wmic useraccount get /all  
```  
### **powershell脚本**  
```  
Import-Module PowerView.ps1  
#查询域内用户  
Get-NetUser | select name  
#统计域内用户数据  
Get-NetUser | select name | measure | findstr Count  
#查询域用户详细信息  
Get-NetUser  
#查询指定用户在哪些组中  
Get-NetGroup -UserName test  
```  
## 查询域内主机  
```  
只会查询domain computers组内的主机，不包含域控等其他组内的主机  
net group &#34;domain computers&#34; /domain  
```  
**powershell脚本**  
```  
Import-Module PowerView.ps1  
# 查询域内主机、包括域控  
Get-NetComputer  
# 统计域内主机数量  
Get-NetComputer | measure | findstr Count  
# 查询带指定字符的主机  
Get-NetComputer *AD*  
```  
**导出域内主机详细信息(域控执行)**  
```  
powershell &#34;Get-ADComputer -Filter *&#34; | Select-Object Name,DistinguishedName,OperatingSystem,OperatingsSystemVersion,ipv4Address|Export-CSV domain_computer.csv&#34;  
```  
## 查询域内的组  
```  
#查询域内的全局组和通用组  
net group /domain  
#查找指定组含有哪些用户  
net group &#34;domain admins&#34; /domain  
```  
**powershell脚本**  
```  
Import-Module PowerView.ps1  
#查询域内的所有组  
Get-NetGroup  
#查询到指定字符串的组  
Get-NetGroup *admin*  
#查询知道用户在哪些组  
Get-NetGroup -userName test  
```  
## 查询域管理员  
```  
#查看域管理员，该组内的成员对域控拥有完全控制权  
net group &#34;domain admins&#34; /domain  
#查看企业管理组，该组内的成员对域控拥有完全控制权  
net group &#34;enterprise admins&#34; /domain  
```  
## 其他信息  
### 查询域密码策略  
`net accounts /domain`  
### 查询域信任关系  
`nltest /domain_trusts`  
### 查询域内的OU组织单位  
```  
Import-Module PowerView.ps1  
Get-NetOU  
```  
### 查询有几个域  
`net view /domain`  
### 查询用户SID和域SID  
```  
whoami /user  
```  
### 查询域内的DNS记录  
```  
Import-Module PowerView.ps1  
Get-DNSRecord -ZoneName xie.com | select name,data  
```  
### 查询域中所有SPN信息  
`Discover-PSinterestingServices.ps1`脚本  
```  
Import-Module Discover-PSinterestingServices.ps1  
Discover-PSinterestingServices  
```  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F-%E5%9F%9F%E5%86%85%E4%BF%A1%E6%81%AF%E6%94%B6%E9%9B%8612/  

