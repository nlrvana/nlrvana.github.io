# 第六章-流量特征分析-小王公司收到的钓鱼邮件

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```help  
服务器场景操作系统 None  
服务器账号密码 None None  
任务环境说明  
   注：样本请勿在本地运行！！！样本请勿在本地运行！！！样本请勿在本地运行！！！  
   应急响应工程师小王在 WAF 上发现了一段恶意流量，请分析流量且提交对应 FLAG  
```  
## 0x01  
1、下载数据包文件 hacker1.pacapng，分析恶意程序访问了内嵌 URL 获取了 zip 压缩包，该 URL 是什么将该 URL作为 FLAG 提交 FLAG（形式：flag{xxxx.co.xxxx/w0ks//?YO=xxxxxxx}） (无需 http、https)；  
搜索`http`流量  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101512356.png)
`flag{tsdandassociates.co.sz/w0ks//?YO=1702920835}`  
2、下载数据包文件 hacker1.pacapng，分析获取到的 zip 压缩包的 MD5 是什么 作为 FLAG 提交 FLAG（形式：flag{md5}）；  
找到响应包，选择导出分组字节流  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101524813.png)
保存为压缩文件即可  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101525947.png)
`flag{f17dc5b1c30c512137e62993d1df9b2f}`  
3、下载数据包文件 hacker1.pacapng，分析 zip 压缩包通过加载其中的 javascript 文件到另一个域名下载后续恶意程序， 该域名是什么?提交答案:flag{域名}(无需 http、https)  
将`js`文件进行美化处理后，放到`console`控制台运行一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406101531492.png)
`flag{shakyastatuestrade.com}`  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E5%85%AD%E7%AB%A0-%E6%B5%81%E9%87%8F%E7%89%B9%E5%BE%81%E5%88%86%E6%9E%90-%E5%B0%8F%E7%8E%8B%E5%85%AC%E5%8F%B8%E6%94%B6%E5%88%B0%E7%9A%84%E9%92%93%E9%B1%BC%E9%82%AE%E4%BB%B6/  

