# Java基础

  
  
&lt;!--more--&gt;  
### for each循环  
`for(variable:collection)statement`  
eg  
```java  
public class Main {    
    public static void main(String[] args) {    
        int[] a = new int[100];    
        for(int i = 0;i&lt;100;i&#43;&#43;){    
            a[i] = i;    
        }    
        for(int element:a)    
            System.out.println(element);    
    }    
}  
```  
### 数组初始化以及匿名数组  
`int[] smallPrimes = {2,3,5,7,11,13};`  
`new int[] {17.19.20};`  
# 对象与类  
## 使用预定义类  
### 对象与对象变量  
`new Date`  
Date类中有一个toString方法，这个方法将返回日期的字符串描述  
`String s = new Date.toString()`  
在这两个例子中，构造的对象仅使用了一次，通常，希望构造的对象可以多次使用，因此，需要讲对象存放在一个变量中  
`Date birthday = new Date();`  
在对象与对象变量之间存在着一个重要的区别  
`Date deadline`  
定义了一个对象变量deadline，它可以引用Date 类型的对象。但是，一定要认识到：变量deadline不是一个对象，实际上也没有引用对象。此时，不能将任何Date 方法应用于这个变量上。语句  
`s = deadline.toString(); // not yet`  
必须首先初始化变量`deadline`，这里有两个选择。当然，可以用新够早的对象初始化这个变量  
`deadline = new Date();`  
## 用户自定义类  
### Employee类  
```java  
import java.time.LocalDate;    
    
public class Employee {    
    public static void main(String[] args) {    
        Employee[] staff = new Employee[3];    
    
        staff[0] = new Employee(&#34;Tom&#34;,1000,1987,12,15);    
        staff[1] = new Employee(&#34;Jim&#34;,2000,1988,12,15);    
        staff[2] = new Employee(&#34;Helen&#34;,3000,1989,12,15);    
    
        for(Employee e : staff)    
            e.raiseSalary(5);    
    
        for(Employee e : staff)    
            System.out.println(&#34;name=&#34;&#43;e.getName()&#43;&#34;,salary=&#34;&#43; e.getSalary()&#43;&#34;,hireDay=&#34;&#43; e.getHireDay());    
    
    }    
    private String name;    
    private double salary;    
    private LocalDate hireDay;    
    // constructor    
    public Employee(String n,double s,int year,int month,int day){    
        name = n;    
        salary = s;    
        hireDay = LocalDate.of(year,month,day);    
    }    
    
    // method    
    public String getName(){    
        return name;    
    }    
    public double getSalary(){    
        return salary;    
    }    
    
    public LocalDate getHireDay(){    
        return hireDay;    
    }    
    
    public void raiseSalary(double byPercent){    
        double raise = salary * byPercent / 100;    
        salary &#43;= raise;    
    }    
}  
```  
### 从构造器开始  
```java  
public Employee(String n,double s,int year,int month,int day){  
	name = n;  
	salary = s;  
	LocalDate hireDay = LocalDate.of(year,month,day);  
}  
```  
可以看到构造器与类同名。在构造Employee类的对象时，构造器会运行，以便将实例域初始化所希望的状态  
构造器与其他的方法有一个重要的不同。构造器总是伴随着new操作符的执行被调用，而不能对一个已经存在的方法调用构造器来达到重新设置实例域的目的。  
### final实例域  
可以将实例域定义为 final。构造对象时必须初始化这样的域。也就是说，必须确保在每一个构造器执行之后，这个域的值被设置，并且在后面的操作中，不能够再对它进行修改。例如，可以将Employee类中的 name 域声明为 final，因为在对象构建之后，这个值不会再被修改，即没有setName方法。  
```java  
class Employee{  
	private final String name;  
}  
```  
## 静态域与静态方法  
在前面给出的示例程序中，main方法都被标记为static修饰符。下面讨论一下这个修饰符  
### 静态域  
如果将域定义为 static，每个类中只有一个这样的域。而每一个对象对于所有的实例域却都有自己的一份拷贝。例如，假定需要给每一个雇员赋予唯一的标识码。这里给Employee 类添加一个实例域id和一个静态域netxId;  
```java  
class Employee{  
	private static int nextId = 1;  
	private int id;  
}  
```  
现在，每一个雇员对象都有一个自己的id域，但这个类的所有实例将共享一个 nextId域。换句话说，如果有 1000 个Employee类的对象，则有1000个实例域 id。但是，只有一个静态域nextId。即使没有一个雇员对象，静态域netxtId也存在。它属于类，而不属于任何独立的对象。  
### 静态常量  
例如  
```java  
public class Math{  
	public static final double PI = 3.1415926585;  
}  
```  
在程序中，可以才用`Math.PI`的形式获得这个常量  
### 静态方法  
静态方法是一种不能向对象实施操作的方法，例如Math 类的pow方法就是一个静态方法  
`Math.pow(x,a)`  
计算 x 的 a 次方  
main 方法也是就一个静态方法  
```java  
public class Application{  
	public static void main(String args[]){  
	//constructor objects here  
	}  
}  
```  
## 对象构造  
### 重载  
有些类有多个构造器。例如，可以构造如下一个空的StringBuilder 对象  
`StringBuilder messages = new StringBuilder()`  
或者，可以指定一个初始化字符串  
`StringBuilder todoList = new StringBuilder(&#34;To do&#34;);`  
这种特征叫做重载。如果多个方法有相同的名字、不同的参数，便产生了重载。编译器必须挑出具体执行哪个方法，它通过用各个方法给出的参数类型与特定方法调用所使用的值类型进行匹配来挑选出相应的方法。  
### 初始化块  
在构造器中设置值  
在声明中赋值  
实际上，Java 还有第三种机制，称为初始化块。在一个类的声明中可以包含多个代码快。只要构造类的对象，这些块就会被执行。例如  
```java  
class Employee{    
    private static int nextId;    
    private int id;    
    private String name;    
    private double  salary;    
    {    
        id = nextId;    
        nextId&#43;&#43;    
    }    
    public Employee(String n,double s){    
        name = n;    
        salary = s;    
    }    
    public Employee(){    
        name=&#34;&#34;;    
        salary= 0;    
    }    
}  
```  
在这个示例中，无论是用哪个构造器构造对象，id 域都在对象初始化块中被初始化。先运行初始化块，然后才运行构造器的主体部分。  
这种机制是不必需的，也不常见。通常会直接将初始化代码放在构造器中。  
如果对类的静态域进行初始化的代码比较复杂，那么可以使用静态的初始化块。  
将代码放在一个块中，并标记关键字 static。下面是一个示例。其功能是将雇员ID的起始值赋予一个小于 10000 的随机整数  
```java  
static{  
	Random generator = new Random();  
	nextId = generator.nextInt(10000);  
}  
```  
在类第一次加载的时，将会进行静态域的初始化。与实例域一样，除非将它们显示地设置成其他值，否则默认的初始化值是 0、false、或null。所有静态初始化语句以及静态初始化块都将依照类定义的顺序执行。  
### 对象析构与 finalize方法  
如果某个资源需要在使用完毕后立刻被关闭， 那么就需要由人工来管理。对象用完时， 可以应用一个 close 方法来完成相应的清理操作。  
## 包  
### 类的导入  
一个类可以使用所属包中所有的类，以及其他包中的公有类。我们可以采用两种方式访问另一个包中的公有类。第一种方式是在每个类名之前添加完整的包名。  
例如  
`java.time.LocalDate today = java.time.LocalDate.now();`  
这很繁琐。更简单常用的方式是使用import语句。import语句是一种引用包含在包中的类的简单描述。一旦使用了import语句，在使用类时，就不必写出包的全名了。  
可以使用 import 语句导入一个特定的类或者整个包。import语句应该位于源文件的顶部(但位于 package 语句的后面)。例如，可以使用下面这条语句导入`java.util`包中所有的类。  
`import java.util.*`  
然后，就可以使用  
`LocalDate today = LocalDate.now()`  
# 继承  
## 类、超类和子类  
### 定义字类  
继承Employee类来定义Manager类的格式，关键字 extends 表示继承。  
```java  
public class Manager extends Employee{   
}  
```  
关键字extends表明正在构造的新类派生于一个已存在的类。已存在的类称为超类(superclass)、基类(base class)或者父类(parent class)；新类称为字类(subclass)、派生类(derived class)。超类和子类是Java 程序员最常用的两个属于，  
尽管Employee类是一个超类，但并不是因为它优于子类或者拥有比子类更多的功能。而恰恰相反，子类拥有比父类更丰富的功能。  
在Manager类中，增加了一个用于存储奖金信息的域，以及一个用于设置这个域的新方法  
```java  
public class Manager extends Employee{    
    private double  bonus;    
    public void setBonus(double bonus){    
        this.bonus = bonus;    
    }    
    
}  
```  
这里定义的方法和域并没有什么特别之处。如果有一个Manger 对象，就可以使用 setBonus 方法  
当然，由于setBonus方法不是在Employee类中定义的，所以属于Employee类的对象不能使用  
### 覆盖方法  
然而，超类中的有些方法对字类Manager并不一定适用。具体来说，Manager类中的getSalary方法应该返回薪水和奖金的总和。为此，需要提供一个新的方法来覆盖超类中的这个方法  
这里主要是利用`super`去调用父类中`private`作用域的`getSalary`方法  
```java  
public double getSalary(){    
    double baseSalary = super.getSalary();    
    return baseSalary &#43; bonus;    
}  
```  
### 子类构造器  
在例子的最后，我们来提供一个构造器  
```java  
public Manager(String n,double s,int year,int month,int day){    
    super(n，s,year,month,day);    
    bonus = 0;    
}  
```  
这里的关键字super具有不同的含义。语句  
`super(n,s,year,year,month,day)`  
是调用超类Employee中含有`n,s,year,month,day`的参数构造器的简写形式  
由于Manger类的构造器不能访问Employee类的私有域，所以必须利用Employee类的构造器对这部分私有域进行初始化，我们可以通过 super 来实现对超类构造器的调用。使用super调用构造器的语句必须是子类构造器的第一条语句  
如果子类的构造器没有显式地调用超类的构造器，则将自动地调用超类默认(没有参数)的构造器。如果超类没有不带参数的构造器，并且在子类构造其中又没有显式地调用超类的其他构造器，则Java 编译器则报告错误  
编写一个demo  
```java  
Manager boss = new Manager(&#34;Tom&#34;,8000,1988,12,15);    
boss.setBonus(1000);    
Employee[] staff = new Employee[3];    
staff[0] = boss;    
staff[1] = new Employee(&#34;Jim&#34;,1000,1985,12,16);    
staff[2] = new Employee(&#34;Lara&#34;,1000,1985,12,16);    
for(Employee e:staff)    
    System.out.println(&#34;name=&#34;&#43;e.getName()&#43;&#34;salary=&#34;&#43;e.getSalary());  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401081653975.png)
需要关注的是`e.getSalary()`  
调用能够确定应该执行哪个`getSalary`方法。请注意，尽管这里将e声明为`Employee`类型，但实际上 e 既可以引用Employee 类型的对象，也可以引用Manger 类型的对象。  
当 e 引用 Employee 对象时， e.getSalary( ) 调用的是 Employee 类中的 getSalary 方法; 当 e 引用 Manager 对象时， e.getSalary( ) 调用的是 Manager 类中的 getSalary 方法。 虚拟机知道 e 实际引用的对象类型， 因此能够正确地调用相应的方法。  
一个对象变量(例如， 变量 e) 可以指示多种实际类型的现象被称为多态(polymorphism)。 在运行时能够自动地选择调用哪个方法的现象称为动态绑定(dynamic binding。)  
### 继承  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401081658086.png)
### 多态  
```java  
Employee e;    
e = new Employee(. . .); // Employee object expected    
e = new Manager(. . .); // OK, Manager can be used as well  
```  
在Java 程序语言设计中，对象变量是多态的。一个Employee变量既可以引用Employee 类对象，也可以引用一个Employee 类的任何一个子类的对象  
置换法则  
```java  
Manager boss = new Manager(. . .);  
Employee[] staff = new Employee[3];   
staff[0] = boss;  
```  
在这个例子中，变量 staff\[0\]与boss引用同一个对象。但编译器将 staff\[0\]看成Employee对象。  
这意味着，可以这样调用  
`boss.setBonus(5000); //OK`  
但不能这样调用  
`staff[0].setBonus(5000) //Error`  
这是因为staff\[0\]声明的类型是Employee，而setBonus 不是Employee 类的方法  
然而，不能将一个超类的引用赋给子类变量。例如，下面的赋值是非法的。  
`Manager m = staff[i] //Error`  
### 阻止继承 ：final 类和方法  
如果不希望Executive有子类，那么需要在前面加上 final 来修饰  
```java  
public final class Executive extends Manger{}  
```  
类中的特定方法也可以被声明为final。如果这样做，字类就不能覆盖这个方法  
### 强制类型转换  
Java 中提供了一种专门用于进行类型转换的表示法  
```java  
double x = 3.4;  
int nx = (int) x;  
```  
对象引用的转换方法与数值表达式的类型转换类似，仅需要一对圆括号将目标类名括起来，并放置在需要转换的对象引用之前就可以了。例如：  
```java  
Manager boss = (Manager) staff[0];  
```  
只能在继承层次内进行类型转换  
在将超类转换成子类之前，应该使用`instanceof`进行检查  
### 抽象类  
使用`abstract`关键字，定义抽象类  
```java  
public abstract String getDescription();  
// no implementation required  
```  
为了提高程序的清晰度，包含一个或多个抽象方法的类本身必须被声明为抽象的。  
```java  
public abstract class Person{  
	public abstract String getDescription();  
}  
```  
除了抽象方法之外，抽象类还可以包含具体数据和具体方法。例如，Person 类还保存着姓名和一个返回姓名的具体方法  
```java  
public abstract class Person{  
	private String name;  
	public Person(String name){  
		this.name = name;  
	}  
	public abstract String getDescription();  
	public String getName(){  
		return name;  
	}  
}  
```  
抽象方法充当着占位的角色，它们的具体实现在子类中。扩展抽象类可以有两种选择。一种是在抽象类中定义部分抽象类方法或不定义抽象类方法，这样就必须将子类也标记为抽象类，另一种是定义全部的抽象方法，这样一来，子类就不是抽象的了。  
写一个Demo  
Main.java  
```java  
public class Main {    
    public static void main(String[] args) {    
        Person[] people = new Person[2];    
        people[0] = new Student(&#34;Jim&#34;);    
        people[1] = new Employee(&#34;Tom&#34;);    
        for(Person p:people)    
            System.out.println(&#34;This is description:&#34;&#43;p.getDescription());    
    }    
}  
```  
Employee.java  
```java  
public class Employee extends Person{    
    private String name;    
    // constructor    
    public Employee(String n){    
        super(n);    
    }    
    // method    
    public String getDescription(){    
        return &#34;I&#39;m a Employee&#34;;    
    }    
}  
```  
Student.java  
```java  
public class Student extends Person{    
    
    public Student(String name){    
        super(name);    
    }    
    public String getDescription(){    
        return &#34;I&#39;m a Student&#34;;    
    }    
    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401082252722.png)
`p.getDescription()`这里不是调用了一个没有定义的方法吗？由于不能构造抽象类Person 的对象，所以变量 p永远不会引用Person 对象，而是引用诸如Employee或者Student 这样的具体子类对象，而这些对象中都定义了`getDescription`方法  
4个访问修饰符  
仅对本类可见-private  
对所有类可见-public  
对本包和所有子类可见-protected  
对本包可见-默认，不需要修饰符  
## Object：所有类的超类  
Object 类是Java 中所有类的始祖，在Java 中每个类都是由它扩展而来的。但是并不需要这样写  
`public class Employee extends Object`  
如果没有明确地指出超类，Object 就被认为是这个类的超类。由于在Java 中，每个类都是由Object 类扩展而来的，所以，熟悉这个类提供的所有服务十分重要。  
可以使用Object类型的变量引用任何类型的对象:  
`Object obj = new Employee(&#34;Jim&#34;,10000);`  
当然，Object 类型的变量只能用于作为各种值的通用持有者。要想对其中的内容进行具体的操作，还需要清楚对象的原始类型，并进行相应的类型转换：  
`Employee e = (Employee) obj;`  
在Java 中，只有基本类型(primitive types)不是对象，例如，数字、字符和布尔类型的值都不是对象。  
所有数组类型，不管是对象数组还是基本类型的数组都扩展了Object类。  
```java  
Employee[] staff = new Employee[10];  
obj = staff;//ok  
obj = new int[10]; //ok  
```  
### equals方法  
`Object`类中的`equals`方法用于检测一个对象是否等于另一个对象。在Object类中，这个方法将判断两个对象是否具有相同的引用。如果两个对象具有相同的引用，它们一定是相等的。从这点上看，将其作为默认操作也是合乎情理的。然而，对于多数类来说，这种判断并没有什么意义。例如，采用这种方式比较两个PrintStram对象是否想等就完全没有意义。然而，经常需要检测两个对象状态的相等性，如果两个对象的状态相等，就认为这两个对象是相等的。  
### hashCode 方法  
散列码(hash code)是由对象导出的一个整型值。散列码是没有规律的。如果 x 和 y 是两个不同的对象，x.hashCode()与y.hashCpde()基本上不会相同。  
String 类使用下列算法计算散列码  
```java  
int hash = 0;  
for(int i = 0;i &lt; length();i&#43;&#43;)  
	hash = 31*hash &#43; charAt(i);  
```  
由于hashCode方法定义在Object 类中，因此每个对象都有一个默认的散列码，其值为对象的存储地址。来看下面这个例子。  
```java  
String s = &#34;Ok&#34;;    
StringBuilder sb = new StringBuilder(s);    
System.out.println(s.hashCode()&#43;&#34; &#34;&#43;sb.hashCode());    
String t = new String(&#34;Ok&#34;);    
StringBuilder tb = new StringBuilder(t);    
System.out.println(t.hashCode()&#43; &#34; &#34; &#43;tb.hashCode());  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401091345602.png)
字符串 t 和 s 拥有相同的散列码，这是因为字符串的散列码是由内容导出的。而字符串缓冲 sb 与 tb 却有着不同的散列码，这是因为在StringBuffer 类中没有定义 hashCode 方法，它的散列码是由Object 类的默认 hashCode 方法导出的对象存储地址。  
如果重新定义 equals 方法，就必须重新定义 hashCode 方法，以便用户可以将对象插入到散列表中。  
hashCode 方法应该返回一个整型数值，并合理地组合实例域的散列码，以便能够让各个不同的对象产生的散列码更加均匀。  
```java  
public class Employee{  
	public int hashCode(){  
		return 7*name.hashCode()   
		&#43;11*new Double(salary).hashCode()  
		&#43;13*hireDay.hashCode();  
	}  
}  
```  
不过，还可以做得更好。首先，最好是用 null 安全的方法Object.hashCode。如果其参数为 null，这个方法会返回 0，否则返回对参数调用 hashCode 的结果。  
### toString方法  
在Object 类中还有一个重要的方法，就是 toString 方法，它用于返回表示对象值的字符串。下面是一个典型的例子。Point 类的 toString 方法将返回下面这样的字符串：  
`java.awt.Point[x=10,y=20]`  
绝大多数的toString方法都遵循这样的格式：类的名字，随后是一对方括号括起来的域值。下面是Employee 类中的toString 方法的实现  
```java  
public String toString(){  
	return &#34;Employee[name=&#34; &#43; name  
	&#43;&#34;]&#34;;  
}  
```  
## 泛型数组列表  
ArrayList是一个采用类型参数的泛型类。为了指定数组列表保存的元素对象类型，需要用一堆尖括号将类名括起来加在后面，例如，ArrayList\&lt;Employee\&gt;。  
下面声明和构造一个保存Employee 对象的数组列表  
`ArrayList&lt;Employee&gt; staff = new ArrayList();`  
使用`add`方法可以将元素添加到数组列表中  
`staff.add(new Employee(&#34;Jim&#34;,...));`  
如果已经清楚或能够估计出数组可能存储的元素数量， 就可以在填充数组之前调用 ensureCapacity 方法:  
`staff.ensureCapacity(lOO);`  
这个方法调用将分配一个包含 100 个对象的内部数组。然后调用 100 次 add, 而不用重新分 配空间。  
另外， 还可以把初始容量传递给 ArrayList 构造器: `ArrayList\&lt;Employee\&gt; staff = new ArrayListo(lOO);`  
`staff.size()`返回数组列表中包含的实际元素数目  
#### 访问数组列表元素  
使用`get`和`set`方法实现访问或改变数组元素的操作，而不使用人们喜爱的`[]`语法格式  
例如设置第i个元素，可以使用  
`staff.set(i.harry)`  
它等价于对数组 a 的元素赋值(数组的下标从 0 开始)  
`a[i] = harry`  
使用下列格式获得数组列表的元素  
`Employee e = staff.get(i);`  
## 反射  
反射机制可以用来  
在运行时分析类的能力  
在运行时查看对象，例如，编写一个 toString 方法供所有类使用  
实现通用的数组操作代码  
利用Method 对象，这个对象很像C&#43;&#43;的函数指针  
### Class 类  
可以通过专门的Java类访问这些信息。保存这些信息的类被称为Class，这个名字很容易让人混淆。Object类中的 getClass()方法将会返回一个Class 类型的实例  
```java  
Employee e;  
Class cl = e.getClass();  
```  
可以调用静态方法 forName 获得类名对应的Class 对象  
`String className = &#39;java.util.Random&#39;;`  
`Class cl = Class.forName(className);`  
获得Class类对象的第三种方法很简单，  
`Class cl1 = Random.class;`  
还有一个很有用的方法`newInstance()`，可以用来动态地创建一个类的实例  
`e.getClass().newInstance();`  
`newInstance`方法会调用默认的构造器初始化新创建的对象。如果这个类没有默认的构造器， 就会抛出一个异常。  
将`forName`与`newInstance`配合起来使用，可以根据存储在字符串中的类名创建一个对象  
`String s = &#34;java.util.Random&#34;;`  
`Object m = Class.forName(s).newInstance();`  
### 利用反射分析类的能力  
检查类的结构  
在`java.lang.reflect`包中有三个类Field,Method 和Constructor 分别用户描述类的域、方法和构造器。这三个类都有一个叫`getName`的方法，用于返回项目的名称。Field 类有一个`getType`方法，用来返回描述域所属类型的Class对象。Method 和 Constructor类能有够报告参数类型的方法，Method 类还有一个可以报告返回类型的方法。这三个类还有一个叫getModifiers 的方法，它将返回一个整型数值，用不同的位开关描述 public 和static这样修饰符使用情况  
Class 类中的 getFields、 getMethods 和 getConstructors 方 法 将 分 别 返 回 类 提 供 的 public 域、 方法和构造器数组，其中包括超类的公有成员  
Class 类中的`getDeclareFields`、`getDeclareMethods`和`getDeclareConstructors`方法将返回类中声明的全部域、方法和构造器，其中包括私有和受保护成员，但不包括超类的成员。  
写一个Demo  
```java  
import org.omg.CORBA.Object;    
    
import java.util.*;    
import java.lang.reflect.*;    
    
    
public class ReflectionTest {    
    public static void main(String[] args) throws Exception{    
        String name;    
        if(args.length &gt; 0) name = args[0];    
        else{    
            Scanner in = new Scanner(System.in);    
            System.out.println(&#34;Enter class name (e.g. java.util.Date)&#34;);    
            name = in.next();    
        }    
        try{    
            Class cl = Class.forName(name);    
            Class supercl = cl.getSuperclass();    
            String modifiers = Modifier.toString(cl.getModifiers());    
            if(modifiers.length() &gt; 0) System.out.println(modifiers &#43; &#34; &#34;)    
            System.out.println(&#34;class &#34;&#43;name);    
            if(supercl != null &amp;&amp; supercl != Object.class) System.out.println(&#34; extends &#34;&#43;supercl.getName());    
            System.out.println(&#34;\n{\n&#34;);    
            printConstructors(cl);    
            System.out.println();    
            printMethods(cl);    
            System.out.println();    
            printFields(cl);    
            System.out.println(&#34;}&#34;);    
        }catch(ClassNotFoundException e) {    
            e.printStackce();    
        }    
    }    
    
    public static void printConstructors(Class cl){    
        Constructor[] constructors = cl.getDeclaredConstructors();    
        for(Constructor c :constructors){    
            String name = c.getName();    
            System.out.println(&#34;     &#34;);    
            String modifiers = Modifier.toString(c.getModifiers());    
            if(modifiers.length()&gt;0) System.out.println(modifiers &#43; &#34; &#34;);    
            System.out.println(name&#43;&#34;(&#34;);    
            Class[] paramTypes = c.getParameterTypes();    
            for(int j = 0;j&lt; paramTypes.length;j&#43;&#43;){    
                if(j&gt;0) System.out.println(&#34;, &#34;);    
                System.out.println(paramTypes[j].getName());    
            }    
            System.out.println(&#34;);&#34;);    
        }    
    }    
    
    public static void printMethods(Class cl){    
        Method[] methods = cl.getDeclaredMethods();    
        for(Method m:methods) {    
            Class retType = m.getReturnType();    
            String name = m.getName();    
            System.out.println(&#34;  &#34;);    
            String modifiers = Modifier.toString(m.getModifiers());    
            if (modifiers.length() &gt; 0) System.out.println(modifiers &#43; &#34; &#34;);    
            System.out.println(retType.getName() &#43; &#34; &#34; &#43; name &#43; &#34;(&#34;);    
            Class[] paramTypes = m.getParameterTypes();    
            for (int j = 0; j &lt; paramTypes.length; j&#43;&#43;) {    
                if (j &gt; 0) System.out.println(&#34;, &#34;);    
                System.out.println(paramTypes[j].getName());    
            }    
            System.out.println(&#34;);&#34;);    
        }    
    }    
    
    public static void printFields(Class cl){    
        Field[] fields = cl.getDeclaredFields();    
        for(Field f:fields){    
            Class type = f.getType();    
            String name = f.getName();    
            System.out.println(&#34;   &#34;);    
            String modifiers = Modifier.toString(f.getModifiers());    
            if(modifiers.length() &gt; 0) System.out.println(modifiers &#43; &#34; &#34;);    
            System.out.println(type.getName()&#43;&#34; &#34;&#43;name&#43;&#34; &#34;);    
        }    
    }    
}  
```  
 - Field\[\] getFields() 1.1    
 - Filed\[\] getDeclaredFields() 1.1  
getFields 方法将返回一个包含 Field 对象的数组， 这些对象记录了这个类或其超类的 公有域。 getDeclaredField 方法也将返回包含 Field 对象的数组， 这些对象记录了这个 类的全部域。 如果类中没有域， 或者 Class 对象描述的是基本类型或数组类型， 这些 方法将返回一个长度为 0 的数组。  
- Method[] getMethods() 1.1    
- Method[] getDeclareMethods() 1.1  
返回包含 Method 对象的数组: getMethods 将返回所有的公有方法， 包括从超类继承 来的公有方法; getDeclaredMethods 返回这个类或接口的全部方法， 但不包括由超类 继承了的方法。  
- Constructor!] getConstructors() 1.1 ;  
- Constructor;] getDeclaredConstructors() 1.1  
返回包含 Constructor 对象的数组， 其中包含了 Class 对象所描述的类的所有公有构造 器(getConstructors) 或所有构造器(getDeclaredConstructors。)  
### 在运行时使用反射分析对象  
查看对象域关键方法是Field 类中的 get 方法。如果 f 是一个Field类型的对象（例如，通过getDeclaredFields得到的对象），obj 是某个包含 f 域的类的对象，f.get(obj)，将返回一个对象，其值为 obj 域的当前值。例如  
```java  
Employee harry = new Employee(&#34;Tom&#34;,35000,10,1,1989);  
Class cl = harry.getClass();  
Field f = cl.getDeclaredField(&#34;name&#34;);  
Object v = f.get(harry);  
//Tom  
```  
这样调用会出现问题，因为 name 是一个私有域，get 方法不能得到  
为了能够调用，需要Field、Method或Constructor 对象的 setAccessible方法  
例如  
`f.setAccessible(1);`  
setAccessible方法是AccessibleObject 类中的一个方法，它是Field、Method 和Constructor 类的公共超类。  
### 调用任意方法  
Method 类中有一个 invoke 方法。它允许调用包装在当前Method 对象中的方法。invoke 方法的签名是  
`Object invoke(Object obj,Object... args)`  
第一个参数是隐式参数，其余的对象提供了显式参数  
对于静态方法，第一个参数可以被忽略，可以将它设置为 null  
例如`m1`代表Employee类的getName方法，下面这条语句显示了如何调用这个方法  
`String n = (String) m1.invoke(harry)`  
如果返回类型是基本类型，invoke方法会返回其包装器类型。例如，假设`m2`表示`Employee`类的`getSalary`方法，那么返回的对象实际上是一个Double，必须相应地完成类型转换。可以使用自动拆装将它转换为一个 double  
`double s = (Double) m2.invoke(harry)`  
如何得到Method 对象呢？当然，可以通过调用`getDeclareMethods`方法，然后对返回的Method 对象数组进行查找，直到发现想要的方法为止。也可以通过调用Class 类中的getMethod方法得到想要的方法。  
getMethod 的签名是  
`Method getMethod(String name,Class ...paramterTypes)`  
例如，下面说明了如何获得Employee 类的 getName 方法和raiseSalary 方法的方法指针  
`Method m1 = Employee.class.getMethod(&#34;getName&#34;);`  
`Method m2 = Employee.class.getMethod(&#34;raiseSalary&#34;,double.class);`  
# 接口、lambda表达式与内部类  
## 接口  
### 接口概念  
在Java 程序设计语言中，接口不是类；而是对类的一组需求描述，这些类要遵从接口描述的统一格式进行定义  
`Arrays`类中sort 方法承诺可以对对象数组进行排序，但要求满足下列前提：对象所属的类必须实现了`Comparable`接口  
下面是`Comparable`接口的代码  
```java  
public interface Comparable{  
	int compareTo(Object other);  
}  
```  
这就是说，任何实现Comparable 接口的类都需要包含`compareTo`方法，并且这个方法的参数必须是一个Object 对象，返回一个整形数值  
接口可以看成没有实例域的抽象类，  
### 接口的特性  
接口不是类，尤其不能使用 new 运算符实例化一个接口  
尽管不能构造接口的对象，却能声明接口的变量  
`Comparable x // ok`  
接口变量必须引用实现了接口的类对象  
`x = new Employee(...);`  
可以使用instance检查一个对象是否实现了某个特定的接口  
`if(anObject instanceof Comparable){...}`  
与可以建立类的继承关系一样，接口也可以被扩展。这里允许存在多条从具有较高通用性的接口到较高专用性的接口的链。例如，假设有一个称为Moveable的接口  
```java  
public interface Moveable{  
	void move(double x,double y);  
}  
```  
然后，可以以它为基础扩展一个叫做`Powered`的接口  
```java  
public interface Powered extends Moveable{  
	double milesPerGallon();  
}  
```  
虽然在接口中不能包含实例域或静态方法，但却可以包含常量。例如  
```java  
public interface Powered extends Moveable  
{  
	double milesPerGallon();  
	double SPEED_LIMIT = 95;  
}  
```  
与接口中的方法都自动地被设置为 public 一样，接口中的域将被自动设为`public static final`  
### 接口与抽象类  
每个类可以实现多个接口，但是不能继承多个抽象类  
### 接口与回调  
回调是一种常见的程序设计模式。在这种模式中，可以指出某个特定时间发生时应该才去的动作，  
Demo  
```java  
package timer;    
    
import java.awt.*;    
import java.awt.event.ActionEvent;    
import java.awt.event.ActionListener;    
import java.util.Date;    
import javax.swing.*;    
    
    
public class TimerTest {    
    public static void main(String[] args) throws Exception {    
        ActionListener listener = new TimePrinter();    
        Timer t = new Timer(3000, listener);    
        t.start();    
        JOptionPane.showMessageDialog(null,&#34;Quit program?&#34;);    
        System.exit(0);    
    }    
}    
    
class TimePrinter implements ActionListener{    
    @Override    
    public void actionPerformed(ActionEvent e) {    
        System.out.println(&#34;At the tone,the time is &#34;&#43; new Date());    
        Toolkit.getDefaultToolkit().beep();    
    }    
}  
```  
### 内部类  
内部类是定义在另一个类中的类。为什么需要使用内部类呢？其主要原因有如下三点  
内部类方法可以访问该类定义所在作用域中的数据，包括私有的数据。  
内部类可以对同一个包中的其他类隐藏起来  
当想要定义一个回调函数且不想编写大量代码时，使用匿名内部类比较便捷  
Demo  
```java  
package innerClass;    
    
import java.awt.*;    
import java.awt.event.ActionEvent;    
import java.awt.event.ActionListener;    
import java.util.*;    
import javax.swing.*;    
import javax.swing.Timer;    
    
public class InnerClassTest {    
    public static void main(String[] args) {    
        TalkingClock clock = new TalkingClock(3000, true);    
        clock.start();    
    
        JOptionPane.showMessageDialog(null, &#34;Quit program?&#34;);    
        System.exit(0);    
    }    
}    
class TalkingClock{    
        private int interval;    
        private boolean beep;    
        public TalkingClock(int interval,boolean beep){    
            this.interval = interval;    
            this.beep = beep;    
        }    
    
        public void start(){    
            ActionListener listenter = new TimePrinter();    
            Timer t = new Timer(interval,listenter);    
            t.start();    
        }    
        public class TimePrinter implements ActionListener{    
    
            @Override    
            public void actionPerformed(ActionEvent e) {    
                System.out.println(&#34;At the tone,the time is &#34;&#43; new Date());    
                if(beep) Toolkit.getDefaultToolkit().beep();    
            }    
        }    
    }  
```  
### 内部类的特殊语法规则  
事实上，使用外部类引用的正规语法还要复杂一些  
`OuterClass.this`  
表示外围类的引用。例如，可以像下面这样编写TimePrinter 内部类的actionPerformed方法  
```java  
public class TimePrinter implements ActionListener{    
  
	@Override    
	public void actionPerformed(ActionEvent e) {    
		System.out.println(&#34;At the tone,the time is &#34;&#43; new Date());    
		if(TalkingClock.this.beep) Toolkit.getDefaultToolkit().beep();    
	}    
}   
```  
在外部类的作用域，可以这样引用内部类  
`OuterClass.InnerClass`  
### 局部内部类  
在 start 方法中定义局部类  
```java  
public void start(){    
	public class TimePrinter implements ActionListener{    
  
		@Override    
		public void actionPerformed(ActionEvent e) {    
			System.out.println(&#34;At the tone,the time is &#34;&#43; new Date());    
			if(beep) Toolkit.getDefaultToolkit().beep();    
		}    
	}   
	ActionListener listenter = new TimePrinter();    
	Timer t = new Timer(interval,listenter);    
	t.start();    
}    
```  
局部类不能用 public或private访问说明符进行声明。它的作用域被限定在声明这个局部类中  
局部类有一个优势，对外部世界可以完全隐藏起来。即使TalkingClock 类中的其他代码也不能访问它。除start方法之外，没有任何方法知道TimePrinter类的存在  
### 匿名内部类  
将局部内部类的使用再深入一步。假如只创建这个类的一个对象，就不必命名了。这种类被称为匿名内部类  
```java  
public void start(){    
	ActionListener listener = new ActionListener(){    
  
		@Override    
		public void actionPerformed(ActionEvent e) {    
			System.out.println(&#34;At the tone,the time is &#34;&#43; new Date());    
			if(beep) Toolkit.getDefaultToolkit().beep();    
		}    
	};  
	ActionListener listenter = new TimePrinter();    
	Timer t = new Timer(interval,listenter);    
	t.start();    
}    
```  
创建一个实现ActionListener 接口的类的新对象，需要实现的方法actionPerformed定义在括号{}内  
语法格式为  
```java  
new SuperType(construction parameters){  
	inner class methods and data  
}  
```  
SuperType 可以是接口，也可以是一个类  
## 代理  
利用代理可以在运行时创建一个实现了一组给定接口的新类  
这种功能只有在编译时无法确定需要实现哪个接口时才有必要使用。  
### 何时使用代理  
代理类可以在运行时创建全新的类。这样的代理类能够实现指定的接口。尤其是，它具有下列方法  
- 指定接口所需要的全部方法  
- Object 类中的全部方法，例如，toString、equals 等  
然而，不能运行时定义这些方法的新代码。而是要提供一个调用处理器`(invocationhandler)`。调用处理器是实现了InvocationHandler 接口的类对象。在这个接口中只有一个方法：  
`Object invoke(Object proxy,Method method,Object[] args)`  
无论何时调用代理对象的方法，调用处理器的invoke方法都会被调用，并向其传递Method对象和原始的调用参数。调用处理器必须给出处理调用的方式。  
### 创建代理对象  
要想创建一个代理对象，需使用Proxy 类的 newProxyInstance 方法。这个方法有三个参数  
- 一个类加载器。  
- 一个Class 对象数组，每个元素都是需要实现的接口。  
- 一个调用处理器。  
还有两个需要解决的问题。如何定义一个处理器？能够用结果代理对象做些什么？当然，这两个问题的答案取决于打算使用代理机制解决什么问题。使用代理可能出于很多种原因  
例如  
- 路由对远程服务器的调用  
- 在程序运行期间，将用户接口事件与动作关联起来。  
- 为调试、跟踪方法调用。  
在示例程序中，使用代理和调用处理器跟踪方法调用，并且定义了一个TraceHandler 包装器类存储包装的对象。其中的 invoke 方法打印出被调用方法的名字和参数，随后用包装好的对象作为隐示参数调用这个方法  
```java  
package ProxyTest;    
    
import java.lang.reflect.InvocationHandler;    
import java.lang.reflect.Method;    
    
public class TraceHandler implements InvocationHandler {    
    
    private Object target;    
    public TraceHandler(Object t){    
        target = t;    
    }    
    @Override    
    public Object invoke(Object proxy, Method m, Object[] args) throws Throwable {    
        return m.invoke(target,args);    
    }    
}  
```  
下面说明如何构造用于跟踪方法调用的代理对象  
```java  
Object value = ...;  
// construct wrapper    
InvocationHandler handler = new TraceHandler(value);    
// construct proxy for one or more interfaces    
Class[] interfaces = new Class[] { Comparable.class };    
Object proxy = Proxy.newProxylnstance(null, interfaces, handler);  
```  
### 代理类的特性  
所有代理类都扩展于Proxy 类。一个代理类只有一个实例域---调用处理器，它定义在Proxy 的超类中。为了履行代理对象的职责，所需要的任何附加数据都必须存储在调用处理器中。  
在上面的代码中，代理Comparable对象时，TraceHandler 包装了实际的对象。  
所有的代理类都覆盖了Object 类中的方法toString、equals和 hashCode。如同所有的代理方法一样，这些方法仅仅调用了调用处理器的 invoke。Object 类中的其他方法（如clone 和 getClass）没有被重新定义  
`java.Iang.reflect.InvocationHandler`  
`Object invoke(Object proxy,Method method,Object[] args)`  
定义了代理对象调用方法时希望执行的动作。  
`java.Iang.reflect.Proxy`  
`static Class&lt;?&gt; getProxyClass( ClassLoader interfaces)`  
返回实现指定接口的代理类。  
`static Object newProxyInstance( ClassLoader interfaces, InvocationHandler handler)`  
构造实现指定接口的代理类的一个新实例。 所有方法会调用给定处理器对象的 invoke 方法。  
`static boolean isProxyClass(Class&lt;?&gt; cl)`  
如果 cl 是一个代理类则返回 true。  
# 部署Java 应用程序  
## JAR文件  
`jar cvf jarfilename File1 File2 ...`  
例如  
`jar cvf CalculatorClasses.jar *.clsss icon.gif`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401101040262.png)
### 清单文件  
除了类文件、图像和其他资源外，每个JAR文件还包含一个用于描述归档特征的清单文件(mainfest)  
清单文件被命名为MANIFEST.MF，它位于JAR文件的一个特殊META-INF子目录中。  
### 可执行JAR文件  
可以使用 jar 命令中的 e 选项指定程序的入口点，即通常需要在调用 java 程序加载器时指定的类  
`jar cvfe MyProgram.jar com.mycompany.mypkg.MainAppClass files to add`  
不管哪一种方法，用户可以简单地通过下面命令来启动应用程序  
`java -jar MyProgram.jar`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202401101046261.png)
无论怎么样，人们对JAR文件中的Java 程序与本地文件有着不同的感觉。在Windows 平台中，可以使用第三方的包装器工具将JAR文件转换成Windows 可执行文件。包装器是一个大家数知的扩展名为`.exe`的Windows 程序，它可以查找和加载Java 虚拟机(JVM)  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/java%E5%9F%BA%E7%A1%80/  

