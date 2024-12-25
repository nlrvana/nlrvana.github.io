# PDO Attack Sqlite

  
  
&lt;!--more--&gt;  
## 0x00 漏洞分析  
先来看一段描述  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412260029964.png)
大体意思就是`PDO::FETCH_CLASS`配置开启之后，会将查出来的数据实例化成类，从而调用`__construct`方法  
创建一个 sqlite 数据库  
sqlite demo  
```php  
&lt;?php  
// 数据库文件路径  
$dbFile = &#39;demo.sqlite&#39;;  
  
try {  
    $pdo = new PDO(&#34;sqlite:$dbFile&#34;);  
  
    echo &#34;SQLite 数据库已连接或创建成功！&lt;br&gt;&#34;;  
  
    // 创建表 SQL  
    $createTableSQL = &#34;  
        CREATE TABLE IF NOT EXISTS test (  
            username TEXT NOT NULL,  
            id INTEGER PRIMARY KEY AUTOINCREMENT  
        );  
    &#34;;  
    // 执行创建表语句  
    $pdo-&gt;exec($createTableSQL);  
  
    $insertSQL = &#34;  
        INSERT INTO test (username)   
        VALUES (:username);  
    &#34;;  
    $stmt = $pdo-&gt;prepare($insertSQL);  
  
    $stmt-&gt;execute([  
        &#39;:username&#39; =&gt; &#39;Demo&#39;,  
    ]);  
} catch (PDOException $e) {  
    // 捕获异常并输出错误信息  
    echo &#34;数据库操作失败: &#34; . $e-&gt;getMessage();  
}  
?&gt;  
```  
vuln demo  
```php  
&lt;?php  
class Demo{  
    public function __construct(){  
        echo &#34;bypass&#34;;  
    }  
}  
try {  
    $conn = new PDO(&#34;sqlite:/Users/f10wers13eicheng/Desktop/webserver/src/demo.sqlite&#34;, &#34;test&#34;, &#34;test&#34;);  
      
    $conn-&gt;setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE,PDO::FETCH_CLASS|PDO::FETCH_CLASSTYPE);  
      
    $sth = $conn-&gt;query(&#34;SELECT * FROM test&#34;);  
    $test = $sth-&gt;fetch();  
} catch (PDOException $e) {  
    echo &#39;Connection Error: &#39; . $e-&gt;getMessage();  
}  
?&gt;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412260036181.png)
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/pdo-attack-sqlite/  

