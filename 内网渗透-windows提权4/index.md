# 内网渗透-Windows提权(4)

  
  
&lt;!--more--&gt;  
## 系统溢出漏洞提权  
各类CVE  
Windows内核溢出漏洞提权是通过系统本身存在的一些漏洞，未曾打相应的补丁而暴露出来的提权方法，依托可以提升权限的EXP和它们的补丁编号，进行提升权限。  
本地溢出提权首先要有服务器的一个普通用户权限，攻击者通常会向服务器上传本地溢出程序，在服务器端执行，如果系统存在漏洞，那么将溢出最高权限。  
### **Windows查看补丁信息**  
```  
systeminfo  
wmic qfe get hostfixid  
```  
**windows系统溢出漏洞提权的汇总**  
https://github.com/SecWiki/windows-kernel-exploits  
**Windows下提权辅助工具**  
https://github.com/AonCyberLabs/Windows-Exploit-Suggester  
https://github.com/bitsadmin/wesng  
### 工具简介  
Windows-Exploit-Suggester  
```  
systeminfo &gt;&gt; systeminfo.txt  
```  
实战中最常用的本地溢出提权有CVE-2018-8120、MS16-032和MS14-058  
- CVE-2018-8120：Windows操作系统Win32k的内核提权漏洞  
## BypassUAC提权  
UAC时微软Microsoft WIndows Vista以后版本引入的一种安全机制。其原理是通知用户是否对应用程序使用硬盘驱动器和系统文件授权，以达到帮助阻止恶意程序（有时也称为“恶意软件”）损坏系统的效果。  
通过UAC，应用程序和任务可始终在非普通管理员账户的安全上下文中运行，除非管理员特别授予管理员级别的系统访问权限。UAC可以阻止未经授权的应用程序自动进行安装，并防止无意中更改系统设置。  
下图清晰描述了如何根据是否启用UAC以及应用程序是否具有UAC清单来运行应用  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202503191915410.png)
在开启了UAC之后，如果用户是标准用户，Windows会给用户分配一个标准Access Token。  
如果用户以管理员权限登陆，会生成两份访问令牌，一分时完整的管理员访问令牌(Full Access Token)，一份是标准用户令牌，具体的表现形式如下图，当我们需要其他特权的时候，会弹出窗口，询问你是否要允许以下程序对此计算机更改？如果你有完整的访问令牌（即，你以设备管理员的身份登陆，或者你属于管理员组），则可以选择是，然后继续进行。但是，如果已为你分配了标准的用户访问令牌，则会提示你输入具有特权的管理员凭据。  
UAC需要授权的过程如下：  
- 配置Windows Update  
- 增加或删除用户账户  
- 改变用户的账户类型  
- 改变UAC设置  
- 安装ActiveX  
- 安装或移除程序  
- 设置家长控制  
- 将文件移动或复制到Prigram Files 或 Windows 目录  
- 查看其他用户文件夹  
### SharpBypssUAC使用  
`https://github.com/fatrodzianko/sharpbypassuac`  
C# tool for UAC bypasses  
  
### 申请UAC弹窗钓鱼  
`https://github.com/cnsimo/BypassUAC`  
利用`Autoelevated`属性的`COM`借口配合PEB伪装实现`BypassUAC`，分成了`C`和`C#`版本  
```  
受害机使用  
uac.exe C:\Users\dev\Desktop\artifact.exe  
  
cobalstrike使用  
execute-assembly /root/uac.exe C:\Users\dev\Desktop\artifact.exe  
execute C:\Users\dev\Desktop\uac.exe C:\Users\dev\Desktop\artifact.exe  
```  
### ElevateKit使用  
`https://github.com/rsmudge/ElevateKit/`  
## Windows Service提权  
### 原理简介  
- 在拥有administrator权限之后利用windows系统服务可以将目前权限提升为`system`  
- 利用的是windows系统服务将会以system权限去启动服务的特性  
例如以下最原始的操作  
```  
设置程序路径  
sc create &#34;scAutoRunTest&#34; binpath=&#34;cmd /c start /b C:\Users\Public\scAutoRunTest.exe&#34;  
设置对服务的描述  
sc description &#34;scAutoRunTest&#34; &#34;scAutoRunTest&#34;  
设置这个服务为自动启动  
sc config &#34;scAutoRunTest&#34; start=auto  
启动服务  
net start &#34;scAutoRunTest&#34;  
sc start &#34;scAutoRunTest&#34;  
  
删除服务  
sc delete scAutoRunTest  
```  
### psexec原理&amp;使用  
PsExec利用某一个账号，通过SMB协议连接远程机器的命名管道(基于SMB的RPC)，在远程机器上创建并启动一个名为`PSEXESVC`服务，从而获得了system权限。  
**使用PsExec的最低要求**  
```  
1、远程机器的139或445端口要开启状态，即SMB  
2、明文密码或者NTLM哈希  
3、具备将文件写入共享文件夹的权限  
4、能够在远程机器上创建服务：SC_MANGER_CREATE_SERVICE(访问掩码:0x0002)  
5、能够启动所创建的服务：SERVICE_QUERY_STATUS(访问掩码:0x0004)&#43;SERVICE_START(访问掩码:0x0010)  
```  
**整个服务**  
- 将`PSEXESVC.exe`上传到`ADMIN$`(指向`/admin$/system32.PSEXESVC.EXE`)共享文件夹内；  
- 远程创建用于运行`PSEXESVC.exe`的服务  
- 远程启动服务  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202503202043757.png)
**psexec的使用**  
`python3 psexec.py &#39;administrator:password&#39;@192.168.135.135`  
### atexec 原理&amp;使用  
`https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/schtasks`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202503202050776.png)
atexec的原理类似，利用administrator账号权限去创建一个以system权限运行的计划任务从而达到administrator-&gt;system的效果  
`python3 atexec.py &#39;administrator:password&#39;@192.168.135.135 whoami`  
  
## 利用相关服务提权(烂土豆系列)  
### 原理讲解  
利用Potato提权的前提是拥有`SeImpersionatePrivilege`或`SeAssignPrimaryTokenPrivilege`权限  
以下用户拥有`SeImpersionatePrivilege`权限(SYSTEM用户才拥有SeAssignPrimaryTokenPrivilege权限)  
- 本地管理员账户(不包括管理员组普通账户)  
- 本地服务账户nt/service ---&gt; system  
以下命令可以查看当前权限的特权信息  
```  
whoami /priv  
whoami /priv | findstr &#34;SeImpersonatePrivilege&#34;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202503150010522.png)
Windows服务的登陆账户  
- Local System(Nt AUTHORITY\SYSTEM)  
- Network Service(NT AUTHORITY\Network Service)  
- Local Service(NT AUTHORITY\Local Service)  
也就是说提权是  
Administrator--&gt;SYSTEM  
Service--&gt;SYSTEM  
### 使用场景  
在具有webshell下，用户权限是IIS或者是Apache  
通过SQL执行xp_cmdshell  
通过mmsql命令执行xp_cmdshell、执行mmsql clr  
在以上场景中，如果web服务或mssql是服务账号(nt\service)启动的，那么我们继承的就是服务账号的低权限，此时利用该提权首发可直接获取SYSTEM权限，即service-system  
通过各种方法在本地NTML中继获取SYSTEM令牌，再通过模拟令牌执行命令，通过以上方法提权统称为potato  
### 历史Potato解析  
#### HOT Potato  
需要等待Windows Update  
`https://github.com/foxglovesec/Potato`  
DBNS欺骗，WPAD和Windows update服务  
原理  
- 通过HOST-DNS使UDP端口耗尽-NBNS  
- 通过fake WPAD proxy Server(劫持http)  
- HTTP-&gt;SMB NTML Relay  
#### Rotten Potato  
`https://github.com/foxglovesec/RottenPotato`  
通过DCOM call来使服务向攻击者监听的端口发起连接并运行NTML认证，需要`SeImpersonatePrivilege`权限  
#### Juicy Potato  
`Rotten Potato`的加强版  
`https://github.com/ohpe/juicy-potato`  
本地支持RPC或远程服务器支持RPC并能成功登陆  
用户支持`SeImpersonate`权限  
开启DCOM  
可用的COM对象  
#### Ghost Potato  
`https://github.com/Ridter/GhostPotato`  
利用CVE-2019-1384(Ghost Potato) Bypass MS-08068  
#### Pipe Potato  
攻击者通过`pipeserver.exe`注册一个名为`pipexpipespoolss`的恶意的命名管道等待高权限用户来连接以模拟高权限用户权限，然后通过`spoolssClient.exe`迫使`system`用户来访问攻击者构建的恶意命名管道,从而模拟`system`用户运行任意应用程序  
原理：  
- 调用CreateNamedPipe()创建一个命名管道  
- 调用ConnectNamePipe()接受该命名请求连接  
- 迫使高权限进程连接该命名管道并写入数据  
- 调用ImpersonateNamedPipeClient()派生，一个高权限进程的客户端  
衍生版本  
PrintSpooferw  
https://github.com/itm4n/PrintSpoofer  
badpotato  
.Net 版本的 pipePotato  
https://github.com/BeichenDream/BadPotato  
利用 spoolsv.exe 进程的 RPC 服务器强制 Windows 主机向其他计算机进行身份  
证验证  
需要 SelmpersonatePrivilege/SeAssignPrimaryToken权限  
#### Sweet Potato  
juicy potato的重写  
https://github.com/CCob/SweetPotato  
COM/WinRM/Spoolsv 的集合版  
也就是 Juicy/PrintSpoofer  
从 Windows 7 到 windows10/windows server2019 的本地服务到 system 特权升级  
#### 最常使用  
**Sweet Potato**  
https://github.com/uknowsec/SweetPotato  
Win7-Win10/Win2008-2019  
`SweetPotato.exe -a &#34;whoami&#34;`  
```  
execute-assembly /root/SweetPotatonew.exe -a &#34;C:\inetpub\wwwroot\artifact.exe&#34;  
  
execute-assembly /root/SweetPotato.exe -a &#34;C:\inetpub\wwwroot\artifact.exe&#34;  
```  
**BadPotato**  
https://github.com/BeichenDream/BadPotato  
Windows 2012-2019、Windows 8-10  
`BadPatatoNet2.exe &#34;whoami&#34;`  
`execute-assembly /root/BadPotatoNet2.exe &#34;C:\inetpub\wwwroot\artifact.exe&#34;`  
### 小结  
Potato提权原理简单来说就是如下三条  
- 诱使&#34;SYSTEM&#34;账户通过NTML向控制的TCP节点进行身份验证  
- 以本地协商&#34;NT AUTHORITY\SYSTEM&#34;账户的安全令牌进行NTLM Relay  
- 模拟刚刚协商的令牌，达到提权的目的  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F-windows%E6%8F%90%E6%9D%834/  

