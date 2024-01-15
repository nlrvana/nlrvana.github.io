# ThinkPHP5漏洞学习

  
  
  
  
&lt;!--more--&gt;  
## ThinkPHP 5基础  
以ThinkPHP5.1.x版本为例  
`composer create-project topthink/think=5.1.36 thinkphp5.1.36`  
[ThinkPHP5.1完全开发手册](https://www.kancloud.cn/manual/thinkphp5_1/353948)
先来了解目录架构，这里直接看手册  
https://www.kancloud.cn/manual/thinkphp5_1/353950  
### 访问模式  
`5.1`的 URL 访问受路由决定，如果在没有定义或匹配路由的情况下(并且没有开启强制路由)，则基于  
`/index.php（或者其它入口文件）/模块/控制器/操作/参数/值…`  
### 请求参数  
继承了控制器基类`think\Controller`直接使用`this-&gt;request-&gt;param(&#39;name&#39;);`来获取请求参数  
`Facade`调用`Request::param(&#39;name&#39;);`  
助手函数`request()-&gt;param(&#39;name&#39;);`  
  
## 漏洞详解  
**parseWhereItem未过滤处理where查询表达式**  
漏洞 Demo  
```php  
public function sqlDemo(){  
	$username = request()-&gt;get(&#39;username&#39;);  
	$result1 = db(&#39;user&#39;)-&gt;where(&#39;username&#39;,&#39;exp&#39;,$username)-&gt;select();   
}  
```  
Poc `) union select 1,2,updatexml(1,concat(1,user(),1),1)--&#43;`  
`debug`断点调试，跟进`where()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401151951144.png)
这里跟进`parseWhereExp()`方法，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401151953723.png)
利用`whereExp`来处理传入的`$username`，继续跟进`whereExp()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401151954352.png)
进行了赋值之后，直接返回，  
便进入到`select()`方法里面，和`ThinkPHP3 SQL 注入`有类似之处，这里直接进入`parseWhere()`方法里面进行了返回  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401152001902.png)
然后进行了一系列操作之后，就直接执行了。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401152027043.png)

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/thinkphp5%E6%BC%8F%E6%B4%9E%E5%AD%A6%E4%B9%A0/  

