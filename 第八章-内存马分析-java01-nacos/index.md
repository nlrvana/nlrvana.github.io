# 第八章-内存马分析-Java01-Nacos

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```  
靶机来源 @vulntarget  
靶机可以采集本地搭建或者是云端调度  
搭建链接 https://github.com/crow821/vulntarget  
ssh root@ip    密码xjnacos 启动 /var/local/下的 autorun.sh即可正常启动  
```  
## 0x01  
1、nacos 用户密码的密文值作为 flag 提交 flag{密文}  
在`conf/nacos-mysql.sql`文件中找到密码  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081406633.png)
`flag{$2a$10$EuWPZHzz32dJN7jexM34MOeYirDdFAZm2kuWj7VEOJhhZkDrxfvUu}`  
2、shiro 的key为多少  shiro 的 key 请记录下来  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081414246.png)
`flag{KduO0i&#43;zUIMcNNJnsZwU9Q==}`  
3、靶机内核版本为   
`uname -r`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081416903.png)
`flag{5.4.0-164-generic}`  
4、尝试应急分析，运行 get_flag 然后尝试 check_flag 通过后提交 flag  
`cat /etc/passwd`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081422364.png)
发现`bt`为最高权限用户，进行删除  
`userdel bt`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081424770.png)
此用户正在运行一些服务，无法删除。  
直接修改`/etc/passwd`文件进行删除  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081435995.png)
5、尝试修复 nacos 并且自行用 poc 测试是否成功  
`8848`端口运行`nacos`服务  
扫描出几个漏洞  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081449057.png)
进行逐一修复  
第一个未授权  
http://68.79.11.194:8848/nacos/v1/auth/users?pageNo=1&amp;pageSize=9  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081450570.png)
在`conf/application.properties`开启鉴权  
```properties  
nacos.core.auth.enabled=true  
nacos.core.auth.enable.userAgentAuthWhite=false  
nacos.core.auth.server.identity.key=key  
nacos.core.auth.server.identity.value=value  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406081505063.png)
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E5%85%AB%E7%AB%A0-%E5%86%85%E5%AD%98%E9%A9%AC%E5%88%86%E6%9E%90-java01-nacos/  

