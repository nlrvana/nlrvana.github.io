# Java中的命令执行

  
  
&lt;!--more--&gt;  
## Runtime  
`Runtime.getRuntime.exec()`  
```java  
import java.io.ByteArrayOutputStream;    
import java.io.InputStream;    
    
public class RuntimeExec {    
    public static void main(String[] args) throws Exception {    
        InputStream in = Runtime.getRuntime().exec(&#34;whoami&#34;).getInputStream();    
        byte[] buffer = new byte[1024];    
        int readSize = 0;    
        ByteArrayOutputStream baos = new ByteArrayOutputStream();    
        while ((readSize = in.read(buffer)) != -1) {    
            baos.write(buffer,0,readSize);    
        }    
        System.out.println(baos.toString());    
    }    
}  
```  
调用栈  
```java  
java.lang.UNIXProcess.&lt;init&gt;(UNIXProcess.java:247)  
java.lang.ProcessImpl.start(ProcessImpl.java:134)  
java.lang.ProcessBuilder.start(ProcessBuilder.java:1029)  
java.lang.Runtime.exec(Runtime.java:620)  
java.lang.Runtime.exec(Runtime.java:450)  
java.lang.Runtime.exec(Runtime.java:347)  
```  
可以看到最终执行命令的是`UNIXProcess`类。  
在`UNIXProcess`类的构造方法中调用了`forkAndExec()` native方法。  
## ProcessBuilder  
从上面可以看出`ProcessBuilder.start()`也可以执行命令  
```java  
import java.io.ByteArrayOutputStream;    
import java.io.InputStream;    
    
public class ProcessExec {    
    public static void main(String[] args) throws Exception {    
        InputStream in = new ProcessBuilder(&#34;whoami&#34;).start().getInputStream();    
        byte[] buffer = new byte[1024];    
        int read = 0;    
        ByteArrayOutputStream baos = new ByteArrayOutputStream();    
        while ((read = in.read(buffer)) != -1) {    
            baos.write(buffer, 0, read);    
        }    
        System.out.println(baos.toString());    
    }    
}  
```  
## ProcessImpl  
`ProcessImpl`也可以执行命令，并且是更底层的实现。  
```java  
import java.io.ByteArrayOutputStream;    
import java.lang.reflect.Method;    
import java.util.Map;    
    
public class ProcessImplExec {    
    public static void main(String[] args) throws Exception {    
        String[] cmds = new String[]{&#34;whoami&#34;};    
        Class clazz = Class.forName(&#34;java.lang.ProcessImpl&#34;);    
        Method m = clazz.getDeclaredMethod(&#34;start&#34;, String[].class, Map.class, String.class, ProcessBuilder.Redirect[].class, boolean.class);    
        m.setAccessible(true);    
        Process process = (Process) m.invoke(null,cmds,null,&#34;.&#34;,null,true);    
        byte[] buffer = new byte[1024];    
        int read = 0;    
        ByteArrayOutputStream baos = new ByteArrayOutputStream();    
        while((read = process.getInputStream().read(buffer)) != -1) {    
            baos.write(buffer, 0, read);    
        }    
        System.out.println(baos.toString());    
    }    
}  
```  
## UnixProcess  
```java  
import sun.misc.Unsafe;    
    
import java.io.ByteArrayOutputStream;    
import java.io.InputStream;    
import java.lang.reflect.Field;    
import java.lang.reflect.Method;    
    
public class UnixProcessExec {    
    public static void main(String[] cmd) throws Exception {    
        String[] strs = new String[]{&#34;whoami&#34;};    
        Field theUnsafeField = Unsafe.class.getDeclaredField(&#34;theUnsafe&#34;);    
        theUnsafeField.setAccessible(true);    
        Unsafe unsafe = (Unsafe) theUnsafeField.get(null);    
        Class processClass = Class.forName(&#34;java.lang.UNIXProcess&#34;);    
        Object processObject = unsafe.allocateInstance(processClass);    
        int[] envc                 = new int[1];    
        int[] std_fds              = new int[]{-1, -1, -1};    
        Field launchMechanismField = processClass.getDeclaredField(&#34;launchMechanism&#34;);    
        Field helperpathField      = processClass.getDeclaredField(&#34;helperpath&#34;);    
        launchMechanismField.setAccessible(true);    
        helperpathField.setAccessible(true);    
        Object launchMechanismObject = launchMechanismField.get(processObject);    
        byte[][] args = new byte[strs.length - 1][];    
        int      size = args.length; // For added NUL bytes    
    
        for (int i = 0; i &lt; args.length; i&#43;&#43;) {    
            args[i] = strs[i &#43; 1].getBytes();    
            size &#43;= args[i].length;    
        }    
    
        byte[] argBlock = new byte[size];    
        int    i        = 0;    
    
        for (byte[] arg : args) {    
            System.arraycopy(arg, 0, argBlock, i, arg.length);    
            i &#43;= arg.length &#43; 1;    
            // No need to write NUL bytes explicitly    
        }    
    
        byte[] helperpathObject      = (byte[]) helperpathField.get(processObject);    
    
        int ordinal = (int) launchMechanismObject.getClass().getMethod(&#34;ordinal&#34;).invoke(launchMechanismObject);    
    
        Method forkMethod = processClass.getDeclaredMethod(&#34;forkAndExec&#34;, new Class[]{    
                int.class, byte[].class, byte[].class, byte[].class, int.class,    
                byte[].class, int.class, byte[].class, int[].class, boolean.class    
        });    
    
        forkMethod.setAccessible(true);    
        int pid = (int) forkMethod.invoke(processObject, new Object[]{    
                ordinal &#43; 1, helperpathObject, toCString(strs[0]), argBlock, args.length,    
                null, envc[0], null, std_fds, false    
        });    
    
        // 初始化命令执行结果，将本地命令执行的输出流转换为程序执行结果的输出流    
        Method initStreamsMethod = processClass.getDeclaredMethod(&#34;initStreams&#34;, int[].class);    
        initStreamsMethod.setAccessible(true);    
        initStreamsMethod.invoke(processObject, std_fds);    
    
        // 获取本地执行结果的输入流    
        Method getInputStreamMethod = processClass.getMethod(&#34;getInputStream&#34;);    
        getInputStreamMethod.setAccessible(true);    
        InputStream in = (InputStream) getInputStreamMethod.invoke(processObject);    
    
        ByteArrayOutputStream baos = new ByteArrayOutputStream();    
        int a= 0;    
        byte[] b = new byte[1024];    
    
        while ((a = in.read(b)) != -1) {    
            baos.write(b, 0, a);    
        }    
        System.out.println(baos.toString());    
    }    
    public static byte[] toCString(String s) {    
        if (s == null)    
            return null;    
        byte[] bytes  = s.getBytes();    
        byte[] result = new byte[bytes.length &#43; 1];    
        System.arraycopy(bytes, 0,    
                result, 0,    
                bytes.length);    
        result[result.length - 1] = (byte) 0;    
        return result;    
    }    
}  
```  
  
## 在不同平台的使用  
### Windows  
因为在`ProcessImpl`中执行的的时候因为没有环境变量，所以我们需要加上`cmd /c`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505262251730.png)
### Linux  
在Linux下我们一般需要通过  
`/bin/bash -c &#34;echo test &gt; test.txt&#34;`来执行命令。  
如果是在Runtime.getRuntime.exec()下，会经过`StringTokenizer()`函数，根据根据空格转换成数组。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505262254845.png)
  
转换成这样的数组，完全就变了达不到我们想要的效果了。  
在`ProcessBuilder`中有几个构造方法，当传入字符串时会分割为数组  
```java  
public ProcessBuilder(String... command) {  
    this.command = new ArrayList&lt;&gt;(command.length);  
    for (String arg : command)  
        this.command.add(arg);  
}  
  
public ProcessBuilder(List&lt;String&gt; command) {  
    if (command == null)  
        throw new NullPointerException();  
    this.command = command;  
}  
```  
当传入的是字符串数组，便会`this.command = command`，避免了`StringTokenizer`的空格解析问题。  
```java  
import java.io.BufferedReader;  
import java.io.IOException;  
import java.io.InputStreamReader;  
import java.nio.charset.Charset;  
  
public class RuntimeExec {  
  
    public static void main(String[] args) {  
        Process process = null;  
        try {  
            String[] cmd = {&#34;/bin/sh&#34;, &#34;-c&#34;, &#34;echo 111 &gt; 3.txt&#34;};  
            process = Runtime.getRuntime().exec(cmd);  
            BufferedReader br = new BufferedReader(new InputStreamReader(process.getInputStream(), Charset.forName(&#34;gbk&#34;)));  
            String line = null;  
            while ((line = br.readLine()) != null) {  
                System.out.println(line);  
            }  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
    }  
}  
```  
## 那参数是字符串和字符串数组有什么区别呢？  
当参数是字符串时，会把第一个参数认为是`prog`(可执行文件)导致无法使用`&amp;&amp;`等特殊符号。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202505262336774.png)
当参数是字符串数组的时候，第一个参数是`/bin/bash`，便可以使用`&amp;&amp;`等特殊符号了  
`String[] cmd = {&#34;/bin/bash&#34;, &#34;-c&#34;, &#34;whoami&amp;&amp;whoami&#34;};`  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/java%E4%B8%AD%E7%9A%84%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C/  

