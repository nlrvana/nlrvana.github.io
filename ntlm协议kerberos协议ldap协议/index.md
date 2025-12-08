# NTLM协议&amp;Kerberos协议&amp;LDAP协议

  
  
&lt;!--more--&gt;  
## NTLM协议  
NTLM协议是微软用于Windows身份验证的主要协议之一。  
早期SMB协议以明文口令的形式在网络上传输，产生了安全性问题，因此出现了LM协议。但是还是太简单。于是微软又提出了NTLM协议，以及更新的NTLM第二版。NTLM协议既可以用于工作组中的机器身份验证，又可用于域环境身份验证，还可为SMB、HTTP、LDAP、SMTP登上层微软应用提供身份验证。  
### LM Hash加密算法  
LM是微软推出一个身份验证协议，使用的加密算法是LM Hash。LM Hash本质是DES加密，尽管LM Hash较容易被破解，但为了保证系统的兼容性，Windows只是将LM Hash禁用(从Windows Vista和Windows Server 2008开始，Windows默认禁用LM Hash)。LM Hash明文密码被限定在14位以内，也就是说若要停止使用LM Hash，将用户的密码设置为14位以上即可。如果LM Hash的值为`aad3b435b51404eeaad3b435b51404ee`，说明LH Hash为空值或者被禁用了。  
### NTLM Hash加密算法  
为了解决LM Hash加密和身份验证方案中固有的安全弱点，微软于1993年在Windows NT 3.1中首次引入了NTLM Hash。微软从Windows Vista和Windows Server 2008开始，默认禁用了LM Hash。只存储NTLM Hash，而LM Hash的位置则为空：`aad3b435b51404eeaad3b435b51404ee`。  
### Windows系统存储的NTLM Hash  
用户密码经过NTLM Hash加密后存储在`C:\Windows\system32\config\SAM`文件中。  
在用户输入密码进行本地认证的过程中，所有操作都是在本地进行的。系统将用户输入的米啊米转换为NTLM Hash，然后与SAM文件中的NTLM Hash进行比较。`winlogon.exe`显示登陆界面，在接收到密码将密码交给`lsass.exe`进程，`lsass.exe`进程中会存一份明文密码，将明文密码加密成`NTLM Hash`，与`SAM`数据库进行比较和验证。使用mimikatz就是从lsass.exe进程中抓取明文密码或者Hash密码。  
### NTLM协议认证  
NTLM协议是一种基于`Challenge/Response`(质询/相应)的验证机制，由三种类型消息组成  
- Type1(协商，Negotiate)  
- Type2(质询，Challenge)  
- Type3(认证，Authentication)  
NTLM协议有NTLM v1和v2两个版本，目前用的最多的是NTLM v2。NTLM v1与NTLM v2最显著的区别是Challenge值与加密算法不同，共同之处是都使用NTLM Hash进行加密。  
#### 工作组环境下的NTLM认证  
工作组环境下的NTLM认证流程  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504181149847.png)
#### 域环境下的NTLM认证  
域环境下的NTLM认证流程  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504181344016.png)
### NTML协议的安全问题  
从上面NTLM认证的流程中可以看到，在`Type 3 NTLMMSSP_AUTH`消息中是使用用户密码Hash计算的。因此，当没有拿到用户密码的明文而只拿到了Hash的情况下，可以进行Pass The Hash攻击，也就是常说的哈希传递攻击。同样，在`Type 3`消息中，存在`Net-NTLM Hash`，当获得了Net-NTLM Hash后，可以进行中间人攻击，重放`Net-NTLM Hash`，这种手法也就是常说的`NTLM Relay`(NTLM中继)攻击。由于NTLM v1协议加密过程存在天然缺陷，因此可以对Net-NTLM v1 Hash进行破解，得到NTLM Hash之后即可横向移动。  
## Kerberos协议  
Kerberos协议是由麻省理工学院提出的一种网络身份验证协议，是一种在开放的非安全网络中认证并识别用户身份信息的方法。它旨在使用密钥加密技术为客户端/服务端应用程序提供强身份验证。Kerberos是西方神话中守卫地狱之门的三头犬的名字。之所以使用这个名字，是因为Kerberos需要三方共同参与才能完成一次认证。  
### Kerberos基础  
在Kerberos协议中，主要有以下三个角色：  
- 在访问服务的客户端：Kerveros客户端代表需要访问资源的用户进行操作的应用程序，例如打开文件、查询数据库或打印文档。每个Kerberos客户端在访问资源之前都会请求身份验证。  
- 提供服务的服务端：域内提供服务的服务端，服务端都有唯一的SPN。  
- 提供认证服务的KDC(Key Distribution Center,密钥分发中心)：KDC是一种网络服务，它向活动目录域内的用户和计算机提供会话票据和临时会话密钥，其服务账户为`krbtgt`。KDC作为活动目录域的一部分运行在每个域控制器上。  
`krbtgt`账户，该用户是在创建活动目录时系统自动创建的一个账户，其作用是KDC(密钥发行中心)的服务账户，其密码是系统随机生成的，无法正常登录主机。  
`Kerberos`是一种基于票据`Ticket`的认证方式。客户端想要访问服务端的某个服务，首先需要购买服务端认可的`ST`(Service Ticket，服务票据)。也就是说，客户端在访问服务之前需要先买好票，等待服务验票之后才能访问。但是这张票并不能直接购买，需要一张`TGT(Ticket Granting Ticket、认购权证)`。也就是说，客户端在买票之前必须先获得一张TGT。TGT和ST均是由KDC发送的，因为KDC运行在域控上，所以说TGT和ST均是由域控发放的。  
`Kerberos`使用`TCP/UDP 88`端口进行认证，使用`TCP/UDP 464`端口进行密码重设。  
`Kerberos`协议有两个基础认证模块-`AS_REQ&amp;AS_REP`和`TGS_REQ&amp;TGS_REP`，以及微软扩展的两个认证模块`S4U`和`PAC`。`S4U`是微软为了实现委派而扩展的模块，分为`S4u2Self`和`S4u2Proxy`。在`Kerberos`最初设计的流程里只说明了如何验证客户端的真实身份，并没有说明客户端是否有权限访问该服务，因为域中不同权限的用户能够访问的资源是不同的。因此微软为了解决权限这个问题，引入了`PAC(Privilege Attribute Certificate，特权属性证书)`的概念。  
在`AS-REQ`阶段，使用的是用户密码Hash或AES Key加密的时间戳。当只获得了用户密码Hash时，发起AS-REQ会造成PTH攻击；当只获得用户密码的`AES Key`时，发起`AS-REQ`会造成`PTK`攻击。  
AS-REQ包中 cname 字段的值代表用户名，这个值存在和不存在，这回的包不一样，所以可以用于枚举域内用户名，这种攻击方式被称为域内用户枚举攻击（当未获取到有效域用户权限时，可以使用这个方法枚举域内用户）。当用户名存在，密码正确和错误，返回的包也不一样，所以可以进行用户名密码爆破。  
在AS-REP 阶段，由于返回的TGT是由krbtgt用户的密码Hash加密的，因此如果我们拥有kcbtgt 的密码 Hash 就可以自己制作一个 TGT，这个票据也被称为黄金票据，这种攻击方式被称为黄金票据传递攻击。同样，在TGS-REP 阶段，TGS_REP 中的ST 是使用服务  
的 Hash 进行加密的，如果我们拥有服务的Hash 就可以签发任意用户的ST，这个票据也被称为白银票据，这种攻击方式被称为白银票据传递攻击。相较于黄金票据传递攻击，白银票据传递攻击使用的是要访问服务的 Hash，而不是 krbtgt 的Hash。  
在 AS-REP 阶段，Logon Session Key 是用用户密码Hash加密的。对于域用户，如果设置了“Do notrequire Kerberos preauthentication”（不需要预认证）选项，攻击者会向域控的88端口发送 AS_REQ，此时域控不会做任何验证就将 TGT 和该用户 Hash 加密的Logon Session Key 返回。这样，攻击者就可以对获取到的用户 Hash 加密的 Logon Session Key 进行离线破解，如果破解成功，就能得到该用户的密码明文，这种攻击方式被称为AS-REP Roasting 攻击。  
在 TGS-REP 阶段，由于ST 是用服务 Hash 加密的，因此，如果我们能获取到ST，就  
可以对该 ST进行破解，得到服务的Hash，造成 Kerberoasting 攻击。这种攻击方式存在的另外一个原因是用户向 KDC发起TGS_REQ 请求时，不管用户对服务有没有访问权限，只要TGT 正确，KDC都会返回ST。其实AS_REQ 中的服务就是krbtgt，也就是说这种攻击方式同样可以用于爆破 AS_REP 中的TGT，但之所以没见到这种攻击方式是因为 krbtgt的密码是随机生成的，爆破不出来。  
### S4u2Self&amp;S4u2Proxy协议  
为了在Kereros协议层面对约束性委派进行支持，微软对Kerberos协议扩展了两个自协议:`S4u2Self(Service for USer to Self)`和`S4u2Proxy(Service for User to Proxy)`。`S4u2Self`可以代表任意用户请求针对自身的服务票据；`S4u2Proxy`可以用上一步获得的`ST`以用户的名义请求针对其他服务的`ST`。  
## LDAP基础  
### 基础模型  
信息模型：LDAP的信息表示方式。因为是面向对象的数据库，所以用户对象，计算机对象等通过对象类描述。对象类包含类和属性，为了应对多而复杂的层级关系，在设计时赋予了类集成的特性，降低了复杂度。  
命名模型：LDAP中的条目定位方式。在LDAP中，每个对象都拥有独一无二的DN属性和RDN，DN是整个树中唯一的标识，RDN是DN中每一个用逗号分隔的表达式，如`CN=Admin,DC=a3sroot,DC=com`中就有3个RDN，分别是`CN=Admin、DC=a3sroot、DC=com`。  
功能模型：包括查询类操作，如搜索、比较；更新类操作，如添加条目、删除条目、修改条目或条目名；认证类操作，如绑定、解绑定；其他操作，如放弃和扩展操作。  
安全模型：LDAP中的安全模型提供3种认证机制，即匿名认证、基本认证和SASL认证。匿名认证即不对用户进行认证，该方法仅对完全公开的方式适用；基本认证是通过用户名和密码进行身份识别，又分为简单密码认证和摘要密码认证；SASL认证是在SSL和TLS安全通道基础上进行的身份认证，包括数字证书的认证。  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/ntlm%E5%8D%8F%E8%AE%AEkerberos%E5%8D%8F%E8%AE%AEldap%E5%8D%8F%E8%AE%AE/  

