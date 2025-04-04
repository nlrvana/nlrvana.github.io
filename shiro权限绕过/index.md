# Shiro权限绕过

  
  
&lt;!--more--&gt;  
## 0x00 环境搭建  
[https://github.com/l3yx/springboot-shiro](https://github.com/l3yx/springboot-shiro)
## 0x01 环境介绍  
shiro 权限逻辑  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031322379.png)
spring业务层admin路由逻辑  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031322810.png)
shiro权限绕过的目标就是不需要登录就等访问`/admin/ctfer`  
### ant匹配  
配置 shiroCOnfig-Filter 的 URL 是 ant 格式，路径支持通配符表示。具体逻辑在  
`AntPathMatcher#doMatch`  
```java  
?：匹配一个字符    
*：匹配零个或多个字符串    
**：匹配路径中的零个或多个路径  
```  
## CVE-2020-1957  
### 影响版本  
Shiro&lt;1.5.2  
Spring框架中只使用了 Shiro 鉴权  
### POC  
```java  
/xxx/..;/admin/ctfer (只适用于Spring低版本&lt;2.3)  
or  
/;/admin/ctfer (只适用于Spring低版本&lt;2.3)  
or  
/admin;/ctfer (Spring全版本)  
漏洞说明  
Spring 与 Shiro 对于 &#34;/&#34; 和 &#34;;&#34; 处理差异导致绕过  
```  
### 漏洞分析  
shiro 鉴权不分`;`截断  
直接定位到 shiro 处理 url 的地方  
`org.apache.shiro.web.util.WebUtils#getPathWithinApplication()`  
这里的request是ShiroHttpServletRequest，继承于HttpServletRequestWrapper，是shiro用来处理web请求自个封装的request  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031329753.png)
跟进`getRequestUri(request)`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031330901.png)
继续跟进，其中通过以下调用链来获取到 uri 的部分，即`/admin;/ctfer`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031331929.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031332663.png)
接着调用了`normalize(decodeAndCleanUriString(request,uri));`来对获取的 uri 进行规范化处理。先跟进`decodeAndCleanUriString`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031333591.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031333909.png)
该方法用于截断`;`后面的部分，因此 uri 变为`/admin`  
继续跟进`normalize`，可以得到`uri`的处理逻辑  
```  
\\ -&gt; /  
/. -&gt; /  
不以/开头，则添加开头/  
// -&gt; / 循环处理  
/./ -&gt; /  
/../ 如果在开头，则返回 null  
```  
```java  
private static String normalize(String path, boolean replaceBackSlash) {    
    
    if (path == null)    
        return null;    
    
    // Create a place for the normalized path    
    String normalized = path;    
    
    if (replaceBackSlash &amp;&amp; normalized.indexOf(&#39;\\&#39;) &gt;= 0)    
        normalized = normalized.replace(&#39;\\&#39;, &#39;/&#39;);    
    
    if (normalized.equals(&#34;/.&#34;))    
        return &#34;/&#34;;    
    
    // Add a leading &#34;/&#34; if necessary    
    if (!normalized.startsWith(&#34;/&#34;))    
        normalized = &#34;/&#34; &#43; normalized;    
    
    // Resolve occurrences of &#34;//&#34; in the normalized path    
    while (true) {    
        int index = normalized.indexOf(&#34;//&#34;);    
        if (index &lt; 0)    
            break;    
        normalized = normalized.substring(0, index) &#43;    
                normalized.substring(index &#43; 1);    
    }    
    
    // Resolve occurrences of &#34;/./&#34; in the normalized path    
    while (true) {    
        int index = normalized.indexOf(&#34;/./&#34;);    
        if (index &lt; 0)    
            break;    
        normalized = normalized.substring(0, index) &#43;    
                normalized.substring(index &#43; 2);    
    }    
    
    // Resolve occurrences of &#34;/../&#34; in the normalized path    
    while (true) {    
        int index = normalized.indexOf(&#34;/../&#34;);    
        if (index &lt; 0)    
            break;    
        if (index == 0)    
            return (null);  // Trying to go outside our context    
        int index2 = normalized.lastIndexOf(&#39;/&#39;, index - 1);    
        normalized = normalized.substring(0, index2) &#43;    
                normalized.substring(index &#43; 3);    
    }    
    
    // Return the normalized path that we have completed    
    return (normalized);    
    
}  
```  
之后就会返回到`PathMatchingFilterChainResolver#getChain`进行权限匹配  
```java  
map.put(&#34;/doLogin&#34;, &#34;anon&#34;);    
map.put(&#34;/admin/*&#34;, &#34;authc&#34;);  
```  
如果此时uri和权限规则定义的路由匹配上了，那么就会进入相应的地方进行鉴权。但是这里是`/admin`并无法匹配上`/admin/*`，所以不会进行鉴权，因此这里就可以绕过了。这里的匹配引擎(规则)使用的是ant风格的匹配规则  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031339225.png)
SpringBoot 路由分发  
绕过了 shiro 鉴权部分后，就会进入到 SpringBoot 路由分发的部分  
断点在`org.springframework.web.servlet.DispatcherServlet#doDispatch()`方法处，这个方法是做 SpringBoot 处理的。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031549296.png)
在`1047`行`DispatcherServle`类收到请求调用 HandlerMapping 处理器映射器。处理器映射器根据请求 url 找到具体的处理器，生成处理器对象 Handler 及处理器拦截器一并返回给 DispatcherServlet  
这里匹配到的第一个`RequestMappingHandlerMapping`，跟进`getHandler()`方法看一下，一路跟进到`org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#getHandlerInternal`方法，在这个方法做了具体业务，这里继续跟进`initLookupPath()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031552467.png)
`initLookupPath()`方法主要做了 URI 处理的初始化，这个 URI 变量最终还是要经过一些处理，继续跟进`remobeSemicolonContent()`方法。`removeSemicolonContent()`方法的意思是判断是否需要删除分号内容，如果需要则跟进`removeSemicolonContentInternal()`方法，如果不需要的话就销毁此 session，将 URI 返回，这一 Uri 就是正确的 URI 了。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031554815.png)
这里需要删除分号，所以跟进`removeSemicolonContentInternal()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031555414.png)
这里会直接删掉`;`后返回  
### SpringBoot 路由处理低版本与高版本的区别  
在`org.springframework.web.servlet.handler#getHandlerInternal`方法处  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031653524.png)
对 URI 处理模式发生了变化，所以导致 POC 在高版本中无法使用  
## CVE-2020-11989  
### 影响版本  
Shiro&lt;1.5.3  
Spring框架只使用了 Shiro 鉴权  
### POC  
```java  
/admin/a%25%32%66a or /admin/a%252fa -&gt; /admin/a%2fa   
Apache Shiro = 1.5.2 &amp; controller需为类似”/hello/{page}”的方式  
  
需要项目部署时ContextPath不是根目录  
/;/context/admin/aaa  
Apache Shiro &lt; 1.5.3 &amp; server.servlet.context-path不为根/  
```  
### 漏洞分析  
`url二次解码\逃逸匹配`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412031756302.png)
在`decodeAndCleanUriString`中会进行`;`的截断处理，再会看一下该方法  
```java  
private static String decodeAndCleanUriString(HttpServletRequest request, String uri) {    
    uri = decodeRequestString(request, uri);    
    int semicolonIndex = uri.indexOf(&#39;;&#39;);    
    return (semicolonIndex != -1 ? uri.substring(0, semicolonIndex) : uri);    
}  
```  
其中`deocdeRequestString(request,uri);`，会对传进来的路径source进行url解码，因此我们POC传进来的URL会经过  
`/admin/a%25%32%66a -&gt; /admin/a%2fa -&gt; /admin/a/a`  
然后返回到`PathMatchingFilterChainResolver#getChain`进行规则匹配鉴权。由于我们的规则是`/admin/*`，根据ant风格的匹配规则，`*`可以匹配单个或多个字符，但不能匹配多个路径，类似`a/a`这样，而`**`可以。所以`/admin/a/a`自然就匹配不上`/admin/*`，所以又绕过了鉴权。  
后面SpringBoot分发部分就一样了，通过`getServletPath`获取到的uri是符合预期的`/admin/a%2fa`，自然可以访问`/admin/{name}`。  
### Apache Shiro = 1.5.2？  
`Shiro1.5.2`使用了`getServletPath()`会进行一次 URL 解码  
```java  
@Override    
public String getServletPath() {    
    return mappingData.wrapperPath.toString();    
}  
```  
`shiro1.5.1`使用了`getRequestURI()`不会进行 URL 解码  
```java  
@Override    
public String getRequestURI() {    
    return coyoteRequest.requestURI().toString();    
}  
```  
## CVE-2020-13933  
### 影响版本  
Shiro&lt;1.6.0  
Spring框架只使用 Shiro 鉴权  
### POC  
`/admin/%3bctfer`  
### 漏洞分析  
`shiro1.5.3`的`org.apache.shiro.web.util.WebUtils#getPathWithinApplication()`函数  
中`removeSemicolon()`函数对`;`的处理  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412032337477.png)
与以上处理一致还是会截取`;`之前的内容，变成了`/admin/`，匹配时无法匹配到`/admin/*`，导致绕过。  
## CVE-2020-17523  
### 影响版本  
`Apache Shiro &lt; 1.7.1`  
`Spring框架只使用Shiro鉴权`  
### POC  
```java  
/admin/%20  
%20可替换为%08、%09、%0a、%0d等其他trim可删除的空白字符，但有时不成功，因为SpringBoot &#43; tomcat可能会400  
```  
### 漏洞分析  
这一次的绕过是`ant`匹配的绕过，之前提到过 ant 语法规定的匹配规则的具体逻辑实现在`AntPathMatcher#doMatch`方法中  
```java  
protected boolean doMatch(String pattern, String path, boolean fullMatch) {    
    if (path.startsWith(this.pathSeparator) != pattern.startsWith(this.pathSeparator)) {    
        return false;    
    }    
    
    String[] pattDirs = StringUtils.tokenizeToStringArray(pattern, this.pathSeparator);    
    String[] pathDirs = StringUtils.tokenizeToStringArray(path, this.pathSeparator);    
    
    int pattIdxStart = 0;    
    int pattIdxEnd = pattDirs.length - 1;    
    int pathIdxStart = 0;    
    int pathIdxEnd = pathDirs.length - 1;    
    
    // Match all elements up to the first **    
    while (pattIdxStart &lt;= pattIdxEnd &amp;&amp; pathIdxStart &lt;= pathIdxEnd) {    
        String patDir = pattDirs[pattIdxStart];    
        if (&#34;**&#34;.equals(patDir)) {    
            break;    
        }    
        if (!matchStrings(patDir, pathDirs[pathIdxStart])) {    
            return false;    
        }    
        pattIdxStart&#43;&#43;;    
        pathIdxStart&#43;&#43;;    
    }    
    
    if (pathIdxStart &gt; pathIdxEnd) {    
        // Path is exhausted, only match if rest of pattern is * or **&#39;s    
        if (pattIdxStart &gt; pattIdxEnd) {    
            return (pattern.endsWith(this.pathSeparator) ?    
                    path.endsWith(this.pathSeparator) : !path.endsWith(this.pathSeparator));    
        }    
        if (!fullMatch) {    
            return true;    
        }    
        if (pattIdxStart == pattIdxEnd &amp;&amp; pattDirs[pattIdxStart].equals(&#34;*&#34;) &amp;&amp;    
                path.endsWith(this.pathSeparator)) {    
            return true;    
        }    
        for (int i = pattIdxStart; i &lt;= pattIdxEnd; i&#43;&#43;) {    
            if (!pattDirs[i].equals(&#34;**&#34;)) {    
                return false;    
            }    
        }    
        return true;    
    } else if (pattIdxStart &gt; pattIdxEnd) {    
        // String not exhausted, but pattern is. Failure.    
        return false;    
    } else if (!fullMatch &amp;&amp; &#34;**&#34;.equals(pattDirs[pattIdxStart])) {    
        // Path start definitely matches due to &#34;**&#34; part in pattern.    
        return true;    
    }    
    
    // up to last &#39;**&#39;    
    while (pattIdxStart &lt;= pattIdxEnd &amp;&amp; pathIdxStart &lt;= pathIdxEnd) {    
        String patDir = pattDirs[pattIdxEnd];    
        if (patDir.equals(&#34;**&#34;)) {    
            break;    
        }    
        if (!matchStrings(patDir, pathDirs[pathIdxEnd])) {    
            return false;    
        }    
        pattIdxEnd--;    
        pathIdxEnd--;    
    }    
    if (pathIdxStart &gt; pathIdxEnd) {    
        // String is exhausted    
        for (int i = pattIdxStart; i &lt;= pattIdxEnd; i&#43;&#43;) {    
            if (!pattDirs[i].equals(&#34;**&#34;)) {    
                return false;    
            }    
        }    
        return true;    
    }    
    
    while (pattIdxStart != pattIdxEnd &amp;&amp; pathIdxStart &lt;= pathIdxEnd) {    
        int patIdxTmp = -1;    
        for (int i = pattIdxStart &#43; 1; i &lt;= pattIdxEnd; i&#43;&#43;) {    
            if (pattDirs[i].equals(&#34;**&#34;)) {    
                patIdxTmp = i;    
                break;    
            }    
        }    
        if (patIdxTmp == pattIdxStart &#43; 1) {    
            // &#39;**/**&#39; situation, so skip one    
            pattIdxStart&#43;&#43;;    
            continue;    
        }    
        // Find the pattern between padIdxStart &amp; padIdxTmp in str between    
        // strIdxStart &amp; strIdxEnd    
        int patLength = (patIdxTmp - pattIdxStart - 1);    
        int strLength = (pathIdxEnd - pathIdxStart &#43; 1);    
        int foundIdx = -1;    
    
        strLoop:    
        for (int i = 0; i &lt;= strLength - patLength; i&#43;&#43;) {    
            for (int j = 0; j &lt; patLength; j&#43;&#43;) {    
                String subPat = (String) pattDirs[pattIdxStart &#43; j &#43; 1];    
                String subStr = (String) pathDirs[pathIdxStart &#43; i &#43; j];    
                if (!matchStrings(subPat, subStr)) {    
                    continue strLoop;    
                }    
            }    
            foundIdx = pathIdxStart &#43; i;    
            break;    
        }    
    
        if (foundIdx == -1) {    
            return false;    
        }    
    
        pattIdxStart = patIdxTmp;    
        pathIdxStart = foundIdx &#43; patLength;    
    }    
    
    for (int i = pattIdxStart; i &lt;= pattIdxEnd; i&#43;&#43;) {    
        if (!pattDirs[i].equals(&#34;**&#34;)) {    
            return false;    
        }    
    }    
    
    return true;    
}  
```  
分析一下这里匹配的逻辑  
以`/`分割`pattern`字符与`path`字符串放到两个数组中，分别是`pattDirs`与`pathDirs`，而后循环遍历`pattDirs`中的值，通过`matchStrings`函数与`pathDirs`数组中的值进行匹配，匹配失败会直接返回false(也就是证明没有权限问题)。如果匹配到了的话，则会每次使`pattIdxStart`自增 1，表示匹配了一项，而`pattIdxEnd`数数组的长度减 1，当`pattIdxStart &gt; pattIdxEnd`时，则一次匹配结束。具体的匹配过程如下：  
 1、先进行第一次处理，正常匹配，匹配不到返回false。直到`pattIdxStart`&gt;`pattIdxEnd`结束第一次处理  
 2、根据第一次处理的结果进行第二次处理，先判断是否是`pathIdxStart &gt; pathIdxEnd`的情况(path数组先到结尾或者两者一起到结尾)  
	 1. 如果`pathIdxStart &gt; pathIdxEnd`，那么说明path和pat数组等长，那么就判断这两个后缀的情况。如果都是`/`或都不是`/`则返回true，否则false。所以path:`/admin/aaa/`是匹配不到pat:`/admin/aaa`的  
	 2. `pattIdxStart == pattIdxEnd &amp;&amp; pattDirs[pattIdxStart].equals(&#34;*&#34;) &amp;&amp; path.endsWith(this.pathSeparator)`则相当于这种情况：path:`/admin/aaa/`，pat:`/admin/aaa/*`，匹配到就返回true  
	 3. 最后就是判断是否有`**`，path:`/admin/aaa`，pat:`/admin/**`，没有直接返回false  
	 4. 判断`pattIdxStart &gt; pattIdxEnd`，则直接返回false，相当于前面的情况都不符合那么就是不匹配了。  
3、上面的if分支没进入`pathIdxStart &gt; pathIdxEnd`，说明是pat先结束或碰到了`**`，那么决定匹配是否成功的pat就是`**`。相当于：path:`/admin/aaa/bbb`，pat:`/admin/**`  
而这里漏洞出现的原因是在一开始获取`pathDirs`的`StringUtils.tokenizeToStringArray`方法中，使用了`trim`方法删除了空白字符，导致最后得到的`pathDirs`是`[&#34;admin&#34;]`，并且`path`是`/admin/[空白字符]`，根据上面的逻辑，会进入 2 的逻辑，然后进入 2.3的逻辑直接返回就 false 了。匹配不到patDirs的`[&#34;admin&#34;,&#34;*&#34;]`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412040020221.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202412040021372.png)
## 总结   
这里以一张表作为总结。  
  
| 漏洞             | Shiro版本 | POC                                                        | 原因                                                                          |  
| -------------- | ------- | ---------------------------------------------------------- | --------------------------------------------------------------------------- |  
| CVE-2020-1957  | &lt; 1.5.2 | (1) /xxx/..;/admin/ctfer  &lt;br&gt;(2) /;/admin/ctfer           | `;`截断导致处理差异                                                                 |  
| CVE-2020-11989 | &lt; 1.5.3 | (1) /admin/a%252fa  &lt;br&gt;(2) /;/ctfer/admin/aaa(部署在context) | (1) 二次url解码为/admin/a/a  &lt;br&gt;(2) getContextPath获取部署目录为未处理`;`  &lt;br&gt;导致和前面一样的原因 |  
| CVE-2020-13933 | &lt; 1.6.0 | /admin/%3bctfer                                            | getServletPath()获取问题，导致`;`截断                                                |  
| CVE-2020-17523 | &lt; 1.7.1 | /admin/[空白字符，一般是%20]                                       | ant匹配规则处理不当。不必要的trim导致绕过匹配                                                  |  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/shiro%E6%9D%83%E9%99%90%E7%BB%95%E8%BF%87/  

