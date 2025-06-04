# 内网渗透-Windows基础信息收集(5)

  
  
&lt;!--more--&gt;  
## Windows基础信息收集(1)  
### 收集系统信息  
```powershell  
ipconfig /all  //查询本机IP段，所在域等  
netstat -an    //网络连接查询（-a显示所有连接和侦听端口,-n以数字形式显示地址和端口号）  
route print    //路由打印  
tracert        //路由追踪  
arp -a         //ARP表查看  
tasklist       //查看进程信息  
nslookup       //查询DNS的记录，查看域名解析  
whoami         //查询账号  
whoami /all    //查看suid值，组信息，特权信息  
net user       //查看本机用户列表  
net user /domain //查询域用户  
net view /domain //查询域列表  
net view         //查询同一域内在线机器列表  
net localgroup administartors /domain //登陆本机的域管理员  
net localgroup administrators //本机管理员  
net session      //查看当前会话  
net share        //查看SMB指向的路径(即共享)  
net accounts     //查看本地密码策略  
net accounts /domain //查看域密码策略  
```  
### 系统信息  
查看系统信息，查看当前机器的版本号，机器名，补丁号，一般收集系统信息是为了查看当前机器为32/64位，机器是什么版本的机器  
### 进程信息  
分析进程信息，查看本机是否有域用户运行的程序，可以利用其身份进行伪造，获得域内用户权限，进行横向移动，同时查看本机其他进程(是否有杀软，正在运行何种程序。)  
**tasklist**  
```powershell  
tasklist /svc //显示每个进程中支持的服务有哪些  
tasklist /m   //列出调用了某个dll的进程  
taskkill /PID 进程ID //结束进程  
taskkill /im  进程名 //结束进程  
tasklist /s 192.168.0.250 /u admin_username /p admin_password //查看远程主机进程  
```  
**netstat**  
`netstat -ano`  
**Hostname**  
`Hostname 主机名`  
**环境变量**  
```  
命令:set  
Path:可执行文件  
windir:操作系统或目录  
```  
**安装的软件**  
`wmic product get name //查看电脑安装的软件`  
**补丁**  
```  
wmic qfe list  
Systeminfo  
Metasploit的post/enum_patches模块  
Windows Exploit Suggester //github项目  
Sherlock //github项目  
使用工具查询比较少，一般用在线网站查询  
http://bugs.hacking8.com/tiquan/  
```  
**服务信息**  
`sc query`  
**计划任务**  
```  
at或schtasks  
schtasks /query /tn test //查看计划任务、注意win7调整CMD编码936为437美国编码  
```  
**注册表信息**  
```  
注册表是windows中最重的一个数据库，保存了系统和各个软件的配置  
可将注册表全部导出 下载下来分析  
reg export HKLM hklm.reg  
reg export HKCU hkcu.reg  
reg export HKCR hkcr.reg  
reg export HKU hku.reg  
reg export HKCC hkcc.reg  
```  
**日志**  
```  
copy C:\Windows\System32\winevt\Logs\System.evtx  
copy C:\Windows\System32\winevt\Logs\security.evtx  
copy C:\Windows\System32\winevt\Logs\application.evtx  
使用日志查看wevtutil  
wevtutil qe security /f:text /q:*[System[(EventID=4624)]] #查寻所有登陆相关的日志  
可用作判断管理员经常在什么时间段上线  
日志ID查询  
https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/  
```  
**Host文件**  
```  
存储主机名映射的ip  
Type C:\Windows\System32\drivers\etc\hosts //查看hosts文件  
可用于发现额外的系统主机  
```  
**组策略**  
```  
组策略提供了操作系统、应用程序和活动目录中用户设置的集中化管理和配置  
使用gpresult /h c:test.html 生成组策略报表为html格式  
```  
**启动项**  
```  
使用注册表查询  
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Run  
```  
**防护软件**  
```powershell  
tasklist #通过进程名称中的关键字判断 如360、anti-virus  
wmic     #或者使用powershell模块  
  
wmic /namespace:\\root\securitycenter2 path antivirusproduct GET displayname,productState,pathToSignedProdutExe  
  
wmic product get name //查看电脑安装的软件  
wmic qfe list         //获取本机补丁详细信息  
在线网络  
http://ddoslinux.com/windows/index.php  
https://maikefee.com/av_list  
https://www.adminxe.com/CompareAV/index.php  
http://payloads.net/kill_software/  
```  
## 工具  
https://github.com/AlessandroZ/LaZagne  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F-windows%E5%9F%BA%E7%A1%80%E4%BF%A1%E6%81%AF%E6%94%B6%E9%9B%865/  

