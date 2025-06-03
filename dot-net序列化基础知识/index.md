# Dot NET序列化基础知识

  
  
&lt;!--more--&gt;  
## 0x00 Serializable特性  
默认情况下.NET特性是不可序列化的，如果要让类实现可序列化的最简单方法是使用`Serializable`特性对其进行标记。`Serializable`是一个特性，类似于`Java`中的注解，表示该类可以进行序列化。  
`Demo`  
```c#  
using System;  
using System.Collections.Generic;  
using System.Linq;  
using System.Text;  
using System.Threading.Tasks;  
using System.Runtime.Serialization.Formatters.Binary;  
using System.IO;  
using System.Runtime.Serialization;  
  
namespace ConsoleApp2  
{  
    [Serializable]  
    public class PayloadClass  
    {  
        public int n1;  
        [NonSerialized] public int n2;  
        public string name;  
    }  
    internal class Program  
    {  
        public static void BinaryFormatterSerialize(string file,object o)  
        {  
            BinaryFormatter binaryFormatter = new BinaryFormatter();  
            FileStream fileStream = new FileStream(file, FileMode.Create);  
            binaryFormatter.Serialize(fileStream, o);  
            fileStream.Close();  
        }  
  
        public static object BinaryFormatterDeserialFromfie(string file)  
        {  
            IFormatter formatter = new BinaryFormatter();  
            Stream stream = new FileStream(file, FileMode.Open);  
            Object o = formatter.Deserialize(stream);  
            stream.Close();  
            return o;  
        }  
        static void Main(string[] args)  
        {  
            PayloadClass payload = new PayloadClass();  
            payload.n1 = 1;  
            payload.n2 = 2;  
            payload.name = &#34;huahua&#34;;  
            BinaryFormatterSerialize(&#34;test.bin&#34;, payload);  
            PayloadClass payload1 = (PayloadClass)BinaryFormatterDeserialFromfie(&#34;test.bin&#34;);  
            Console.WriteLine(payload1.n1);  
            Console.WriteLine(payload1.n2);  
            Console.WriteLine(payload1.name);  
  
        }  
    }  
}  
```  
输出  
```  
1  
0  
huahua  
```  
在上述的Demo中，我们引入了一个`BinaryFormatter`类，这个类表示使用二进制的形式进行反序列化，而在`dotnet`中有很多其他的`formatter`类，每一个`formatter`都对应了一种序列化的格式，例如  
```c#  
BinaryFormatter 二进制格式  
SoapFormatter 序列化soap格式  
LosFormatter 序列化Web窗体页的试图状态  
ObjectStateFormatter 序列化状态对象图  
```  
这个`formatter`类都实现了名为`IFormatter`、`IRemotingFormatter`的接口。其中`IRemotingFormatter`是用来远程调用的`RPC`接口，它也实现了`IFormatter`，所以我们重点看一下`IFormatter`接口。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506031457420.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506031458564.png)
`IFormater`定义了序列化和反序列化的方法，和三个字段  
- ISurrogateSelector SurrogateSelector 序列化代理选择器 接管formatter的序列化或反序列化处理。  
- SerializationBinder Binder 用于控制在序列化和反序列化期间使用的实际类型  
- StreamingContext Context 序列化流上下文 其中states字段包含了序列化的来源和目的地。  
BinaryFormatter序列化的生命周期和事件  
- 首先确定formatter是否有代理选择器，如果有则检查代理选择器要处理的对象类型是否和给定的对象类型一致，如果一致，代理选择则调用`ISerializable.GetObjectData()`  
- 如果没有代理选择器，或者代理选择器不处理该对象类型，则检查对象是否有`[Serializable]`特性。如果不能序列化则抛出异常。  
- 检查该对象是否实现`ISerializable`接口，如果实现就调用其`GetObjectData`方法。  
- 如果没有实现`ISerializable`接口就使用默认的序列化策略，序列化所有没标记`[NonSerialized]`的字段。  
在序列化和反序列化的过程中还有四个回调事件  
- OnDeserializingAttribute 反序列化之前 初始化可选字段的默认值  
- OnDeserializedAttribute 反序列化之后 根据其他字段的内容修改可选字段值  
- OnSerializingAttribute 反序列化之前 准备系列化，例如，创建可选数据结构。  
- OnserializedAttribute 序列化之后 记录序列化事件  
一张图来表示序列化及反序列化的生命周期  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506031904308.png)
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/dot-net%E5%BA%8F%E5%88%97%E5%8C%96%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/  

