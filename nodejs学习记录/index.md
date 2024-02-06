# NodeJs学习

  
  
&lt;!--more--&gt;  
## Nodejs   
Nodejs是一个基于 Chrome V8 引擎解析 JavaScript  
## fs 文件系统模块  
```node  
const fs = require(&#34;fs&#34;)  
  
fs.readFile(path[,options],callback) //读取文件  
// 文件路径  
// 以什么编码格式读取文件  
// 文件读取完成后，通过回调函数拿到读取的结果  
fs.writeFile(file,data[,options],callback) //向指定的文件写入内容  
// 文件路径  
// 写入的内容  
// 以什么格式写入  
// 文件写入完成后的回调函数  
```  
读取文件  
```node  
const fs = require(&#34;fs&#34;)  
  
fs.readFile(&#34;./test.txt&#34;,&#34;utf8&#34;,function(err,dataStr){  
    console.log(err)  
    console.log(&#34;------&#34;)  
    console.log(dataStr)  
})  
```  
写入文件  
```node  
const fs = require(&#34;fs&#34;)  
  
fs.writeFile(&#34;test.txt&#34;,&#34;Hello World&#34;,&#34;utf8&#34;,function(err){  
    console.log(err)  
})  
```  
## path路径模块  
path 模块是用来提供处理路径的模块。  
`path.join()`方法，用来将多个路径片段拼接成一个完整的路径字符串  
`path.basename()`方法，用来从路径字符串中，将文件名解析出来  
```node  
const path = require(&#34;path&#34;);  
  
path.join([...paths])   
// 路径  
  
path.basename(path[,ext])  
// 文件路径  
```  
`path.extname()`获取路径中的扩展名部分  
```node  
path.extname(path)  
// 文件路径  
```  
## http模块  
创建 web 服务器的模块  
```node  
const http = require(&#39;http&#39;)  
const server = http.createServer();  
//使用服务器实例的 .on() 方法，为服务器绑定一个 request 事件  
server.on(&#39;request&#39;,(req,res) =&gt;{  
	//req 请求对象  
	//req.url 是客户端请求的 URL 地址  
	//req.method 是客户端的 method 请求类型  
	//只要有客户端请求服务器，就会触发request事件，从而调用这个处理函数  
	//res 响应对象  
	const str = &#34;Hello World&#34;  
	//向客户端响应内容  
	res.end(str)  
	console.log(&#34;Someone visit our web server .&#34;)  
})  
server.listen(9091,()=&gt;{  
	console.log(&#39;http server running at http://127.0.0.1:9091&#39;)  
})  
  
```  
## 模块化  
内置模块、自定义模块、第三方模块  
```node  
 // 加载内置模块  
 const fs = require(&#34;fs&#34;)  
 // 加载用户的自定义模块  
 const custom = require(&#34;./common.js&#34;)  
 // 加载第三方模块  
 const moment = require(&#34;moment&#34;)  
```  
### 向外共享模块作用域中的成员  
module 对象  
```node  
console.log(module)  
  
{  
  id: &#39;.&#39;,  
  path: &#39;/Users/f10wers13eicheng/Desktop/node-project&#39;,  
  exports: {},  
  filename: &#39;/Users/f10wers13eicheng/Desktop/node-project/index.js&#39;,  
  loaded: false,  
  children: [],  
  paths: [  
    &#39;/Users/f10wers13eicheng/Desktop/node-project/node_modules&#39;,  
    &#39;/Users/f10wers13eicheng/Desktop/node_modules&#39;,  
    &#39;/Users/f10wers13eicheng/node_modules&#39;,  
    &#39;/Users/node_modules&#39;,  
    &#39;/node_modules&#39;  
  ]  
}  
```  
`module.exports`对象  
可以将模块内的成员共享出去，供外界使用  
```node  
module.exports.username = &#39;zs&#39;  
module.exports.sayHello = function(){  
	console.log(&#34;Hello&#34;)  
}  
```  
使用`require()`方法导入模块时，导入的结果，永远以`module.exports`指向的对象为准  
### npm与包  
https://www.npmjs.cn/  
`npm install 包的完整名称`  
初次装包完整后，在项目文件夹多一个叫做 node_modules的文件夹和package_locak.json的配置文件  
`npm i moment@2.22.2` 安装指定的版本  
#### 包管理配置文件  
npm规定，在项目跟目录中，必须提供一个叫做 package.json的包管理配置文件。用来记录与项目有关的一些配置信息  
快速创建`package.json`  
`npm init -y`  
`npm install` 会读取`package.json`文件进行下载  
  
`npm config get registry` 查看当前的下包镜像源  
#### 包的分类  
指定`-g` 安装为全局包  
不指定则为 项目包  
## 模块的加载机制  
优先从缓存中加载  
内置模块加载优先级最高  
## Express  
Express 是基于 nodejs 平台，快速、开发、极简的 Web 开发框架  
`npm i express@4.17.1`  
### 创建基本的 Web 服务器  
```node  
const express = require(&#34;express&#34;)  
// 创建 web 服务器  
const app = express()  
  
app.listen(9091,()=&gt;{  
    console.log(&#34;express server running at http://127.0.0.1:9091&#34;)  
})  
```  
### 监听 GET/POST 请求  
`app.get()/app.post()`方法，可以监听客户端 `GET/POST` 请求  
```node  
app.get(&#39;请求URL&#39;,function(req))  
//客户端请求的 URL 地址  
// 请求对应的处理函数  
// req: 请求对象  
// res: 响应对象  
app.get(&#34;/welcome&#34;,(req,res)=&gt;{  
    res.send(&#34;Hello World&#34;)  
})  
```  
### 获取 URL 中携带的查询参数  
通过`req.query`对象，可以放问到客户端通过查询字符串的形式，发送到服务器的参数  
```node  
app.get(&#39;/&#39;,(req.res)=&gt;{  
	// req.query 默认是一个空对象  
	// 客户端使用?name=zs&amp;age=20  
	// 可以通过 req.query 对象访问到  
	// req.query.name req.query.age  
	console.log(req.query)  
})  
```  
### 获取 URL 中的动态参数  
通过`req.params`对象，可以访问到 URL 中，通过`:`匹配到的动态参数  
```node  
app.get(&#39;/user/:id&#39;,(req,res)=&gt;{  
	//req.params 里面存放着通过:动态匹配到的参数值  
	console.log(req.params)  
})  
```  
### 托管静态资源  
express.static()  
express提供了一个static()函数，创建一个静态资源服务器  
例如，通过如下代码可以将public目录下的图片、css 文件、js 文件对外开放访问了  
`app.use(express.static(&#39;public&#39;))`  
#### 挂载路径前缀  
如果希望托管在静态资源访问路径之前，挂载路径前缀  
`app.use(&#39;/public&#39;,express.static(&#39;public&#39;))`  
### nodemon使用  
`npm install -h nodemon` 能够实时监听代码修改  
### 路由  
express中的路由分 3 部分组成，分别是请求的类型、请求的 URL 地址、处理函数  
`app.METHOD(PATH,HANDLER)`  
```node  
app.get(&#34;/&#34;,(req.res)=&gt;{  
	res.send(&#34;Hello World&#34;)  
})sss  
app.post(&#34;/&#34;,(req,res)=&gt;{  
	res.send(&#34;Hello Worlds&#34;)  
})  
```  
#### 模块化  
将路由抽离为单独的模块  
1. 创建路由模块对应的 js 文件  
2. 调用express.Router()函数创建路由对象  
3. 向路由对象上挂载具体的路由  
4. 使用module.exports向外共享路由的对象  
5. 使用app.use()函数注册路由模块  
创建路由模块  
```node  
var express = require(&#34;express&#34;)  
var router = express.Router  
  
router.get(&#39;/user/list&#39;,(req,res)=&gt;{  
    res.send(&#34;Get user list&#34;)  
})  
router.post(&#39;/user/add&#39;,(req,res)=&gt;{  
    res.send(&#39;Add new user.&#39;)  
})  
  
module.exports = router  
```  
注册路由模块  
```node  
const userRouter = require(&#34;./route.js&#34;)  
app.use(userRouter)  
```  
`app.use()`函数的作用，就是来注册全局中间件  
### 中间件  
当一个请求到达 Express 的服务器之后，可以连续调用多个中间件，从而对这次请求进行预处理  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402061315167.png)
客户端发起的任何请求，到达服务器之后，都会触发的中间件，叫做全局生效的中间件  
通过调用`app.use(中间件函数)`，即可定义**全局生效**的中间件函数  
最简单的中间件函数  
```node  
const mw = function(req,res,next){  
	console.log(&#39;中间件函数&#39;)  
	next()  
}  
```  
**局部生效**的中间件，不使用 app.use() 定义的中间件  
```node  
const mw = function(req,res,next){  
	console.log(&#39;中间件函数&#39;)  
	next()  
}  
app.get(&#39;/&#39;,mw,function(req,res){  
	res.send(&#34;Hello World&#34;)  
})  
```  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/nodejs%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/  

