# Java 动态类加载

  
  
&lt;!--more--&gt;  
## 类加载与反序列化  
类加载的时候会执行代码  
初始化：静态代码快  
实例化：构造代码快、无参构造函数  
## 动态类加载方法  
`Class.forname`  
初始化/不初始化  
`ClassLoader.loadClass`不进行初始化  
底层的原理，实现加载任意的类  
  
`URLClassLoader`任意类加载：file/http/jar  
```java  
URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{new URL(&#34;http://localhost:9080/&#34;)});    
Class&lt;?&gt; c = urlClassLoader.loadClass(&#34;Test&#34;);    
c.newInstance();  
```  
`ClassLoader.defineClass`字节码加载任意类  私有  
```java  
ClassLoader cl = ClassLoader.getSystemClassLoader();    
Method defineClassMethod = ClassLoader.class.getDeclaredMethod(&#34;defineClass&#34;, String.class, byte[].class, int.class, int.class);    
defineClassMethod.setAccessible(true);    
byte[] code = Files.readAllBytes(Paths.get(&#34;/Users/f10wers13eicheng/Desktop/JavaSecuritytalk/JavaThings/VulnDemo/src/main/java/org/example/LoaderDemo/Test.class&#34;));    
Class c= (Class) defineClassMethod.invoke(cl,&#34;Test&#34;,code,0,code.length);    
c.newInstance();  
```  
`Unsafe.defineClass`字节码加载 public类不能直接生成 Spring 里面可以直接生成  
```java  
ClassLoader cl = ClassLoader.getSystemClassLoader();    
Class c = Unsafe.class;    
Field theUnsafeField = c.getDeclaredField(&#34;theUnsafe&#34;);    
theUnsafeField.setAccessible(true);    
Unsafe unsafe = (Unsafe) theUnsafeField.get(null);    
byte[] code = Files.readAllBytes(Paths.get(&#34;/Users/f10wers13eicheng/Desktop/JavaSecuritytalk/JavaThings/VulnDemo/src/main/java/org/example/LoaderDemo/Test.class&#34;));    
Class c2 = unsafe.defineClass(&#34;Test&#34;,code,0,code.length,cl,null);    
c2.newInstance();  
```  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/java-%E5%8A%A8%E6%80%81%E7%B1%BB%E5%8A%A0%E8%BD%BD/  

