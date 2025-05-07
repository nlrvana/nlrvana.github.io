# 域基础知识

  
  
&lt;!--more--&gt;  
## 域中常见名词  
### AD  
AD(Active Directory，活动目录)是微软对通用目录服务数据库的实现。  
活动目录使用LDAP作为其主要访问协议，它存储着有关网络中各种对象的信息，如用户账户、计算机账户、组合Kerberos使用的所有相关凭据等，以便管理员和用户能够轻松地查找和使用这些信息。  
### ADDS  
ADDS(Active Directory Domain Service，活动目录域服务)是Windows Server 2000操作系统的中心组件之一，活动目录可以作为活动目录域服务或活动目录轻型目录服务(ADLDS)部署。  
### LDAP  
既然有目录服务数据库，就需要目录访问协议去访问，而LDAP(Lightweight Directory Access Protocol，轻量级目录访问协议)就是用来访问目录服务数据库的协议之一。  
LDAP基于`X.500`标准，是一个开放的、中立的、工业标准的应用协议。它可以用来查询与更新活动目录数据库，可以利用LDAP名称路径来描述对象在活动目录中的位置。LDAP目录类似于文件系统目录，例如，CN=DC01,OU=Domain Controllers,DC=xie,DC=com，类比成文件系统目录，可被看作`xie.com/Domain Controllers/DC01`，这个路径的含义就是DC01对象处于`xie.com`域的`Domain Controllers`组织单位中。在LDAP中，数据以树状方式组织，树状信息中的基本数据单元是条目，而每个条目由属性组成。  
#### X.500标准定义  
LDAP如何快速定位需要查询的对象呢？  
DC  
DC(Domain Component，域组件)不是Domain Controllers，而是类似于DNS中的每个元素，DC对象表示使用DNS来定义其名称空间的LDAP树的顶部。例如域xie.com，用`&#34;.&#34;`分开的每个单元都可以看成一个域组件。  
OU  
在活动目录数据定义了OU(Organization Unit,组织单位)类，最多可以有四级，每级最长32个字符，可以为中文。如OU=Doamin Controllers就是一个组织单位，组织单位中包含对象、容器，还可以包含其他组织单位，组织单位还可以链接组策略。  
CN  
CN(Common Name,通用名称)是对象的名称，最大长度为80个字符，可以为中文。例如，一个用户名为张三，那么张三就是CN，再比如一个计算机名为`Win7`，那么`Win7`就是CN。  
DN  
活动目录中的每个对象都有完全唯一的DN(Distinguished Name,可分辨名称)，其包含对象到LDAP名称空间根的整个路径。DN有三个属性，分别是DC、OU、CN。DN可以表示为LDAP中的某个目录，也可以表示成目录中的某个对象，这个对象可以是用户、计算机等。如&#34;CN=AD,OU=Domain Controllers,DC=xie,DC=com&#34;是一个DN，它是层次结构树，从右向左，表示的是xie.com域的Domain Controllers组织单位写的AD对象。  
RDN  
RDN（Relative Distinguished Name，相对可分辨名称）与目录结构无关。比如CN=  
Administrator,CN=Users,DC=xie,DC=com，它的 RDN 为 CN=Adiministrator。两个对象可以具有相同的 RDN，但是不能具有相同的DN。  
UPN  
UPN（User Principal Name，用户主体名称）是用户的可辨别名称，在域内是唯一的。如域 xie.com 中的 administator 用户，它的UPN 为 administrator@xie.com。用户登录时最好输人UPN，由于无论用户的账号被移动到哪一个域，其UPN 都不会变化，因此用户可以用同一个 UPN 来登录。  
Container在活动目录数据中定义了Container类。容器是一些属性的集合，容器内可以包含其他对象，如用户、计算机等，但是容器中不能再嵌套其他容器和OU。  
FQDN  
FQDN(Fully Qualified Domain Name,全限定名)是同时带有主机名和域名的名称。如域xie.com下的Win7主机，其FQDN为Win7.xie.com。全限定域名可以从逻辑上准确地表示主机在域名树中的位置，也可以说是主机名的一种完全表示形式。  
## 本地账户和活动目录账户  
### 本地账户  
本地账户存储在本地的服务器上。这些账户可以在本地服务器上分配权限，但只能在该服务器上进行分配。默认的本地账户是哪只账户(Administrator、Guest等)，在安装Windows时自动创建，切无法删除。此外，默认的本地账户不提供对网络资源的访问，根据分配给该账户的权限来管理对本地服务器资源的访问。默认的本地账户和后期创建的本地账户都位于&#34;用户&#34;文件夹中。  
### 活动目录账户  
活动目录账户(Active Directory Account)是活动目录中的账户，可分为用户账户、服务账户和机器账户。活动目录账户存储在活动目录数据库中。  
#### 机器账户  
活动目录即去账户其实也是一种特殊的用户账户，只不过它不能用于登陆。机器账户可以代表一个物理实体，如域内机器。在域内，机器账户和域用户一样，也是域内的成员，他在域内的用户名是`机器账户名&#43;$`，比如机器Win8的机器账户为`Win8$`  
## 服务主体名称  
服务主体名称(Service Principal Name,SPN)是服务实例的唯一标识符。Kerberos使用SPN将服务实例与服务登陆账户相关联。如果在整个林或域中的计算机上安装多个服务实例，则每个实例都必须要具有自己的SPN。  
## 域中的访问控制列表  
访问控制列表(Access Control List,ACL)。由一系列访问控制条目(Access Control Entry,ACE)组成，用于定义安全对象的访问策略。而每个ACE可以看作一条策略。每个安全对象的安全描述符包含两种类型的ACL:DACL和SACL。这两种ACL都是由ACE组成的。  
  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E5%9F%9F%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/  

