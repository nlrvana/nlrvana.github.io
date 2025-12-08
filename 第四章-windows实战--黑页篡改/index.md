# 第四章-Windows实战-黑页&amp;&amp;篡改

  
  
&lt;!--more--&gt;  
## 0x00 环境分析  
```help  
第四章 windows实战-黑页&amp;&amp;篡改  
  
远程端口 3389  账号密码 administrator xj@123456  
请启动 phpstudy 后访问 127.0.0.1  
具体要求已经在桌面 readme.png 本题为开放式题目，请访问 http://127.0.0.1/dedecms/index.php  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111809415.png)
## 0x01  
1、http://localhost/dedecms/index1.php  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111813904.png)
直接在`index1.php`删除  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111819153.png)
修复完成  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111819552.png)
2、http://localhost/dedecms/index2.php  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111820224.png)
跳转到了`/heiye/index2.html`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111820559.png)
在`index2.php`中并无跳转的`url`，追踪包含的文件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111823690.png)
在`common2.inc.php`中发现跳转的文件  
修复完成  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111824003.png)
3、http://localhost/dedecms/index3.php  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111825412.png)
跳转到了`/heiye/index3.html`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111826716.png)
在`index3.php`中引入了一串恶意的`js`文件  
`&lt;script src=&#34;&amp;#104;&amp;#116;&amp;#116;&amp;#112;&amp;#58;&amp;#47;&amp;#47;&amp;#49;&amp;#50;&amp;#55;&amp;#46;&amp;#48;&amp;#46;&amp;#48;&amp;#46;&amp;#49;&amp;#47;&amp;#100;&amp;#101;&amp;#100;&amp;#101;&amp;#99;&amp;#109;&amp;#115;&amp;#47;&amp;#105;&amp;#110;&amp;#99;&amp;#108;&amp;#117;&amp;#100;&amp;#101;&amp;#47;&amp;#49;&amp;#46;&amp;#106;&amp;#115;&#34;&gt;&lt;/script&gt;`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111828197.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111828513.png)
修复完成  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111828533.png)
4、http://localhost/dedecms/index4.php  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406111829146.png)
跳转到了`/heiye/index2.html`  
`index4.php`中并无发现可以内容，追踪其包含文件  
在目录中并没有看到`common3.inc.php`，利用自带的`everything`进行搜索  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406112152014.png)
可以看到成功搜索出来，打开后进行修复  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406112144468.png)
5、http://localhost/dedecms/heiye/index5.html  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406112138331.png)
在`index5.php`中发现包含了一串加密的东西，解密一下试试  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406112140410.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406112140382.png)
直接删除即可  
修复  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406112141283.png)
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%AC%AC%E5%9B%9B%E7%AB%A0-windows%E5%AE%9E%E6%88%98--%E9%BB%91%E9%A1%B5%E7%AF%A1%E6%94%B9/  

