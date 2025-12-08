# CodeQL学习之路(1)-基础知识

  
  
&lt;!--more--&gt;  
## 0x00 CodeQL中的Java库  
标准的Java库可以分为五个主要类别  
1、用于表示程序元素(例如类和方法)的类  
2、用于表示AST节点(例如语句和表达式)的类  
3、用于表示元数据(例如注释和评论)的类  
4、用于计算度量的类(例如圈复杂度和耦合)  
5、用于浏览程序调用图的类  
## 0x01 程序元素  
这些类代表命名的程序元素：`package(包)`、`compilationUnit(编译单元)`、`Type(类型)`、`Method(方法)`、`Constructor(构造函数)`和`Variable(变量)`  
它们的共同超类是`Element`，它提供通用成员谓词，用于确定程序元素的名称并检查两个元素是否相互嵌套。  
引用方法和构造函数的元素通常很方便；例如`Callable`类是`Method`和`Constructor`的公共超类。  
## 0x02 类型  
`Type`有许多子类，用户表示不同类型  
`PrimitiveType`表示原始类型，`boolean`、`byte`、`char`、`double`、`float`、`int`、`long`、`short`，`void`、`null`  
`Reftype`表示引用类型，它又有多个子类  
- Class表示一个Java类  
- Interface表示一个Java接口  
- EnumType表示Java enum类型  
- Array表示Java数组类型  
例如，查找程序中的`int`变量  
```q  
import java  
  
from Variable v,PrimitiveType pt   
where pt = v.getType() and  
    pt.hasName(&#34;int&#34;)  
select v  
```  
## 0x03 变量  
Class`Variable`代表Java意义上的变量，可以是类的成员字段，也可以是局部变量，也可以是参数。因此，有三个子类可以满足这些特殊情况  
- Field表示一个Java字段  
- LocalVariableDecl表示局部变量  
- Parameter表示方法或者构造函数参数  
## 0x04 抽象语法树  
Class`Stmt`和表达式Class`Expr`。有关标准QL库中可用的表达式和语句类型有完整列表 [用于 Java 和 Kotlin 程序的抽象语法树类](https://codeql.github.com/docs/codeql-language-guides/abstract-syntax-tree-classes-for-working-with-java-programs/)”。  
`Expr`和`Stmt`提供了用于探索程序抽象语法树的成员谓词  
- `Expr.getAChildExpr`返回给定表达式的子表达式  
- `Stmt.getAChild`返回直接嵌套在给定语句内的语句或表达式  
- `Expr.getParent`和`Stmt.getParent`返回AST节点的父节点  
例如，以下查询所有父级语句为`return`的表达式  
```q  
import java  
  
from Expr e  
where e.getParent() instanceof ReturnStmt  
select e  
```  
例如，以下查询寻找父级为`if`语句的语句  
```q  
import java  
  
from Stmt s  
where s.getParent() instanceof IfStmt  
select s  
```  
此查询将查询程序中所有语句的`then`分支和`else`分支  
方法主体的查询  
```q  
import java  
  
from Stmt s   
where s.getParent() instanceof Method   
select s  
```  
也就是说，表达式的父节点并不总是表达式，他也可能是一个语句，比如`IfStmt`。同样，语句的父节点并不总是语句：它也可能是一个方法或构造函数  
[AST类](https://codeql.github.com/docs/codeql-language-guides/overflow-prone-comparisons-in-java/)  
## 0x05 调用图  
上面介绍的类`Callable`包括方法和构造函数。调用表达式使用类抽象使用类抽象出来`Call`，其中包括方法调用、`new`表达式以及使用`this`或者显式构造函数调用`super`  
使用谓词`Call.getCallee`来找出特定调用表达式引用的方法或构造函数。例如，下面就是找所有方法中调用的`println`  
```q  
import java  
  
from Call c, Method m  
where m = c.getCallee() and  
    m.hasName(&#34;println&#34;)  
select c  
```  
[相关文章](https://codeql.github.com/docs/codeql-language-guides/navigating-the-call-graph/)  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/codeql%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF1-%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/  

