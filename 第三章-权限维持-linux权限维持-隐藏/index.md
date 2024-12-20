# 第三章-权限维持-Linux权限维持隐藏

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```  
ssh root@env.xj.edisec.net -p xjqxwcyc  
1.黑客隐藏的隐藏的文件 完整路径md5  
2.黑客隐藏的文件反弹shell的ip&#43;端口 {ip:port}  
3.黑客提权所用的命令 完整路径的md5 flag{md5}   
4.黑客尝试注入恶意代码的工具完整路径md5  
5.使用命令运行 ./x.xx 执行该文件  将查询的 Exec****** 值 作为flag提交 flag{/xxx/xxx/xxx}  
```  
## 0x01  
1.黑客隐藏的隐藏的文件 完整路径md5  
`find / -name &#34;.*&#34;`，在`tmp`下面发现隐藏路径  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282033736.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282035951.png)
在`1.py`中实现了一个持久化的反向`shell`  
并且在`c`文件中实现了对`1.py`的隐藏  
`flag{109ccb5768c70638e24fb46ee7957e37}`  
2.黑客隐藏的文件反弹shell的ip&#43;端口 {ip:port}  
提交`1.py`中的ip和port  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282039294.png)
3.黑客提权所用的命令 完整路径的md5 flag{md5}  
使用了最常规的`suid`提权  
`find / -perm -u=s -type f 2&gt;/dev/null`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282042058.png)
`find`命令可直接进行提权  
`flag{7fd5884f493f4aaf96abee286ee04120}`  
4.黑客尝试注入恶意代码的工具完整路径md5  
在`/opt`目录下，发现恶意工具  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282044051.png)
`cymothoa`是一款黑客工具  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282045275.png)
`flag{087c267368ece4fcf422ff733b51aed9}`  
5.使用命令运行 ./x.xx 执行该文件  将查询的 Exec****** 值 作为flag提交 flag{/xxx/xxx/xxx}  
使用python3运行一下`/tmp/.temp/libprocesshider/1.py`  
成功运行  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282055274.png)
`which`查找一下`python3`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202405282057697.png)
`flag{/usr/bin/python3.4}`  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E4%B8%89%E7%AB%A0-%E6%9D%83%E9%99%90%E7%BB%B4%E6%8C%81-linux%E6%9D%83%E9%99%90%E7%BB%B4%E6%8C%81-%E9%9A%90%E8%97%8F/  

