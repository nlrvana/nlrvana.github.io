# ThinkPHP5.0学习-某盘审计

  
  
&lt;!--more--&gt;  
[ThinkPHP5.0完全开发手册](https://www.kancloud.cn/manual/thinkphp5/118031)
## 某盘代码审计  
学习完开发手册，先来套源码试试手。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401162144477.png) 
想要快速入手代码审计，就要先看手册了解一些底层架构等等  
从`index.php`开始  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401162146823.png)
和原来的相比改动不大，应用目录是`application`，直接进行查看  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401162147144.png)
先看`config.php`，看一些配置信息能够帮助我们快速了解此套源码  
全局过滤方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401162147609.png)
访问模式，最最最重要的是我们的控制器  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401162148256.png)
可以看到，它开启了路由但是并没有打开强制路由  
这里看一下路由，并没有什么东西  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401162150312.png)
想要得到未授权RCE，还是得先看`Index`模块下面的控制器，因为可能有未授权访问，像这种 MVC 架构`admin`模块下，未授权太少了Orz  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401162152522.png)
除了`Api`、`Login`控制器，其他控制器均继承了`Base`控制器  
那他们为什么要都继承`Base`呢？进入看一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401162155087.png)
`Orz`在构造函数下面做了权限验证，扫了一眼除非知道数据库里的`$header_uid`不然无法绕过  
但是，我们上面还提到了一个`api`控制器，没有继承。进入查看一下  
### 0x01 SSRF  
在`curlfun()`函数下面，看到了一个很明显的`SSRF`  
```php  
 public function curlfun($url, $params = array(), $method = &#39;GET&#39;)  
    {  
  
        $header = array();  
        $opts = array(CURLOPT_TIMEOUT =&gt; 10, CURLOPT_RETURNTRANSFER =&gt; 1, CURLOPT_SSL_VERIFYPEER =&gt; false, CURLOPT_SSL_VERIFYHOST =&gt; false, CURLOPT_HTTPHEADER =&gt; $header);  
  
        /* 根据请求类型设置特定参数 */  
        switch (strtoupper($method)) {  
            case &#39;GET&#39; :  
                $opts[CURLOPT_URL] = $url . &#39;?&#39; . http_build_query($params);  
                $opts[CURLOPT_URL] = substr($opts[CURLOPT_URL],0,-1);  
  
                break;  
            case &#39;POST&#39; :  
                //判断是否传输文件  
                $params = http_build_query($params);  
                $opts[CURLOPT_URL] = $url;  
                $opts[CURLOPT_POST] = 1;  
                $opts[CURLOPT_POSTFIELDS] = $params;  
                break;  
            default :  
  
        }  
  
        /* 初始化并执行curl请求 */  
        $ch = curl_init();  
        curl_setopt_array($ch, $opts);  
        $data = curl_exec($ch);  
        $error = curl_error($ch);  
        curl_close($ch);  
  
        if($error){  
            $data = null;  
        }  
  
        return $data;  
  
    }  
```  
先本地起一个服务  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401162158212.png)
构造一个`poc`试试，  
`/index.php/index/api/curlfun?url=http://localhost:9080/ssrf.txt`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401162159504.png)
### 0x02 SSRF  
`api`控制器下，还存在一个`post_url`方法，也存在`SSRF`  
```php  
public function post_curl($url,$data){  
	$ch = curl_init($url);  
	curl_setopt($ch, CURLOPT_CUSTOMREQUEST, &#34;POST&#34;);  
	curl_setopt($ch, CURLOPT_POSTFIELDS,$data);  
	curl_setopt($ch, CURLOPT_RETURNTRANSFER,true);  
	curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);  
	$result = curl_exec($ch);  
	if (curl_errno($ch)) {  
		print curl_error($ch);  
	}  
	curl_close($ch);  
	return $result;  
}  
```  
### 0x03 后台 RCE  
在`admin`模块`setup`控制器`editconf`方法处，存在文件上传漏洞  
```php  
public function editconf()  
    {  
		  
		// echo &#34;test&#34;;  
		  
		// if($this-&gt;otype != 3){  
		// 	echo &#39;死你全家!&#39;;exit;  
		// }  
          
        if(input(&#39;post.&#39;)){  
  
            $data = input(&#39;post.&#39;);  
              
             
  
            foreach ($data as $k =&gt; $v) {  
                $arr = explode(&#39;_&#39;,$k);  
                $_data[&#39;id&#39;] = $arr[1];  
                $_data[&#39;value&#39;] = $v;  
                $file = request()-&gt;file(&#39;pic_&#39;.$_data[&#39;id&#39;]);  
                  
                if($file){  
                      
                    $info = $file-&gt;move(ROOT_PATH . &#39;public&#39; . DS . &#39;uploads&#39;);  
                    if($info){  
                        $_data[&#39;value&#39;] = &#39;/public&#39; . DS . &#39;uploads/&#39;.$info-&gt;getSaveName();  
                    }  
                }  
                if($_data[&#39;value&#39;] == &#39;&#39; &amp;&amp; isset($arr[2]) &amp;&amp; $arr[2] == 3){  
                    continue;  
                }  
                  
                Db::name(&#39;config&#39;)-&gt;update($_data);  
  
            }  
            cache(&#39;conf&#39;,null);  
            $this-&gt;success(&#39;编辑成功&#39;);  
        }  
  
          
    }  
```  
可以看到直接`input`接受参数，并且利用了`request()-&gt;file`来上传文件  
而在`thinkphp`中的`file`函数，是没有安全设置的  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401162321002.png)
所以可以直接进行上传，  
构造`poc`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401170004509.png)
在`public`目录下面，成功进行了上传  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401170003014.png)
但是文件名字看起来是一串随机的字符串组成的，这怎么办呢？我们根进去查看一下是怎么生成的，跟进`move`函数  
看到了`buildSavename`函数正是文件保存命名规则，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401170006695.png) 
跟进去查看一下，命名规则是`date`日期  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401170007762.png)
前面的`date`函数是 uploads 下面的目录，而后面的`md5(microtime(true))`正是生成的那一串看似随机的字符串，爆破一下即可出文件名  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401170008715.png) 
那针对`microtime(true)`这样的该如何爆破呢？  
在`ctfshow 元旦水友赛`中出过这样一道题  
https://docs.qq.com/doc/DRlBMcWdhZW9ZUnFB  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401170012739.png)
里面有爆破脚本  
在`System`控制器下，同样存在类似的功能点  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401170017144.png)
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/thinkphp5.0%E5%AD%A6%E4%B9%A0-%E6%9F%90%E7%9B%98%E5%AE%A1%E8%AE%A1/  

