# 第七章-常见攻击事件分析-钓鱼邮件

  
  
&lt;!--more--&gt;  
## 0x00环境分析  
```  
请分析获取黑客发送钓鱼邮件时使用的IP，flag格式： flag{11.22.33.44}  
请分析获取黑客钓鱼邮件中使用的木马程序的控制端IP，flag格式：flag{11.22.33.44}  
黑客在被控服务器上创建了webshell，请分析获取webshell的文件名，请使用完整文件格式，flag格式：flag{/var/www/html/shell.php}  
flag4: 黑客在被控服务器上创建了内网代理隐蔽通信隧道，请分析获取该隧道程序的文件名，请使用完整文件路径，flag格式：flag{/opt/apache2/shell}  
```  
## 0x01  
1、请分析获取黑客发送钓鱼邮件时使用的IP  
在`钓鱼邮件.eml`中看到`ip`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406042004372.png)
`flag{121.204.224.15}`  
2、请分析获取黑客钓鱼邮件中使用的木马程序的控制端IP  
将`eml`中传输的文件进行恢复  
```python  
import base64  
  
with open(&#34;base64.txt&#34;,&#34;r&#34;) as f:  
    result = f.read().strip().replace(&#34;\n&#34;,&#34;&#34;)  
with open(&#34;kill.exe&#34;,&#34;wb&#34;) as f:  
    f.write(base64.b64decode(result))  
```  
上传到微步查杀中  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406042011353.png)
`flag{107.16.111.57}`  
3、黑客在被控服务器上创建了webshell，请分析获取webshell的文件名，请使用完整文件格式  
利用D盾扫描  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406042033723.png)
验证一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406042034074.png)
  
`flag{/var/www/html/admin/ebak/ReData.php}`  
4、黑客在被控服务器上创建了内网代理隐蔽通信隧道，请分析获取该隧道程序的文件名，请使用完整文件路径  
发现`mysql.conf`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406042035877.png)
同目录下的`mysql`文件很可能就是内网代理工具  
`flag{/var/tmp/proc/mysql}`  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E4%B8%83%E7%AB%A0-%E5%B8%B8%E8%A7%81%E6%94%BB%E5%87%BB%E4%BA%8B%E4%BB%B6%E5%88%86%E6%9E%90--%E9%92%93%E9%B1%BC%E9%82%AE%E4%BB%B6/  

