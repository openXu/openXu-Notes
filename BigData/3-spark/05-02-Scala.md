
Scala详细教程网上有很多，比如[易百教程](https://www.yiibai.com/scala/scala_functions.html)，本文只列举出
相对于Java有明显不同的概念和相关知识点，用于熟悉Java的同学快速了解Scala。

Scala与Java完全不同的功能如下：

- 所有类型都是对象
- 类型推断
- 函数是对象
- 域特定语言（DSL）支持
- 性状
- 闭包
- 并发支持灵感来自Erlang

如果您熟悉Java语言语法和编程，那么学习Scala将会很容易。

# 基础语法

## 语句

Scala行结束字符的分号(;) 是可选的。如果在单行上写多个语句，则需要分号

## 变量、常量

```Scala
// var关键字定义变量
var myVar : String = "Foo"
// 类型推断
var myVar = 10

// val关键字定义常量（相当于java中final）
val myVal = "Hello, Scala!"
val myVal = 10

// Scala支持多个赋值。如果代码块或方法返回一个元组(Tuple - 保持不同类型的对象的集合)，则可以将元组分配给一个val变量。
val (myVar1: Int, myVar2: String) = Pair(40, "Foo")
val (myVar1, myVar2) = Pair(40, "Foo")
```

## 访问修饰符

- 包，类或对象的成员可以使用private、protected、public标注，public可省略不写
- private私有只能在类内部使用
- protected成员只能从定义成员的类的子类中访问，同一个包中不能访问（java允许）
- public可以从任何地方访问

```Scala
// 1. private  私有方法只能在类内部使用
class Outer {
   class Inner {
	  // 私有方法
      private def f() { println("f") }
      class InnerMost {
         f() // 正确: 在类内部使用
      }
   }
   (new Inner).f() // 错误: 所有方法不能在类外部使用
}

// 2. protected 成员只能从定义成员的类的子类中访问
package p {
   class Super {
      protected def f() { println("f") }
   }
   class Sub extends Super {
      f()    //正确: 从定义成员的类的子类中访问
   }
   class Other {
      (new Super).f() // 错误: 在外部类调用了protected成员（这在Java中是允许的，java中的protected可在同一个包中访问）
   }
}

// 3. Public 没有明确使用修饰符则自动归为公共成员，这些成员可以从任何地方访问
class Outer {
   class Inner {
      def f() { println("f") }
      class InnerMost {
         f() // 正确
      }
   }
   (new Inner).f() // 正确
}
```

## 数据类型

Scala中没有类似Java中的原始数据类型（int、long、float等）

| 序号 | 数据类型 | 说明 |
|:--|:--:|--:|
|1|Byte|8位有符号值，范围从-128至127|
|2|Short|16位有符号值，范围从-32768至32767|
|3|Int|32位有符号值，范围从-2147483648至2147483647|
|4|Long|64位有符号值，范围从-9223372036854775808至9223372036854775807|
|5|Float|32位IEEE 754单精度浮点值|
|6|Double|64位IEEE 754双精度浮点值|
|7|Char|16位无符号Unicode字符。范围从U+0000到U+FFFF|
|8|String|一个Char类型序列，三个引号""" ... """表示多行字符串|
|9|Boolean|文字值true或文字值false|
|10|Unit|对应于无值|
|11|Null|null或空引用
|12|Nothing|每种其他类型的亚型; 不包括无值|
|13|Any|任何类型的超类型; 任何对象的类型为Any|
|14|AnyRef|任何引用类型的超类型|

## 运算符

与java中基本一致

## 循环语句

```Scala
// 1. while
while( a < 20 ){
 println( "Value of a: " + a );
 a = a + 1;
}

//2. do while
do {
 println( "Value of a: " + a );
 a = a + 1;
}while( a < 20 )

//3. for 语法如下，与java写法稍有区别
for( var x <- Range ){
   statement(s);
}
for(a <- 1 to 10){
 println( "Value of a: " + a );
}
```

## 函数

Scala中具有两种函数，术语方法和函数是可以互换的。方法是类的一部分，函数可以分配给一个变量
的完整对象（函数是对象）。定义为某个对象的成员的函数称为方法。函数定义可以出现在源文件的任何位置，
Scala允许嵌套函数定义。下面列举函数定义以及与Java中不同的一些函数类型：

```Scala
object add {
   /*
	* 1. 定义函数   格式 : 
	* def 函数名 (参数列表) : 返回值类型 = {函数体;return 值}
	* 
	* 函数无返回值，返回值类型为Unit，相当于Java中的void，该函数称为过程
	* 
	* 返回语句中return也可以省略，默认最后一个表达式的值作为返回值
	* 
	* (: 返回值类型)也可以省略，类型自动推导
    */
   def addInt( a:Int, b:Int ) : Int = {
      var sum:Int = 0
      sum = a + b
      return sum
   }
}

/**
 * 2. 名称参数 :通常，函数的参数是按值参数;如果一个函数需要接受一个表达式作为参数，该参数在Scala中称为名称参数
 * 格式: (参数名称:参数类型 => 函数参数返回值类型)
 */
object Demo {
   def main(args: Array[String]) {
        delayed(time());
   }
   def time() = {
      println("Getting time in nano seconds")
      System.nanoTime
   }
   //定义名称参数函数，该函数接受一个表达式作为参数，表达式的值就是该函数的实参
   def delayed( t: => Long ) = {
      println("In delayed method")
      println("Param: " + t)
   }
}

/**
 * 3. 高阶函数: Scala允许定义高阶函数。它是将其他函数作为参数或其结果是函数的函数。
 */
object Demo {
   def main(args: Array[String]) {
      println( apply( layout, 10) )
   }
   //定义高阶函数，将其他函数作为参数，或者其结果是函数
   def apply(f: Int => String, v: Int) = f(v)

   def layout[A](x: A) = "[" + x.toString() + "]"
}

/**
 * 4. 命名参数函数: 正常函数调用传参都是按照函数定义的参数顺序匹配。
 *                 名参数允许以不同的顺序将参数传递给函数，但需要每个参数前面都有一个参数名称和一个等号
 */
object Demo {
   def main(args: Array[String]) {
	  //使用命名参数调用函数
      printInt(b = 5, a = 7);
   }
   def printInt( a:Int, b:Int ) = {
      println("Value of a : " + a );
      println("Value of b : " + b );
   }
}
/**
 * 5. 可变参数:函数最后一个参数可重复，Java中使用...，Scala中使用*
 */
object Demo {
   def main(args: Array[String]) {
      printStrings("Hello", "Scala", "Python");
   }
   //定义可变参数函数，实际上是一个数组Array[String]
   def printStrings(args:String*) = {
      var i : Int = 0;
      for( arg <- args ){
         println("Arg value[" + i + "] = " + arg );
         i = i + 1;
      }
   }
}

/**
 * 6. 默认参数值函数:定义函数时指定参数默认值，调用时可选择性省略传递实参
 */
object Demo {
   def main(args: Array[String]) {
      println( "Returned Value : " + addInt() );
   }
   def addInt( a:Int = 5, b:Int = 7 ) : Int = {
      var sum:Int = 0
      sum = a + b
      return sum
   }
}

/**
 * 7. 嵌套函数: 在函数内部定义函数，
 */
object Demo {
   def main(args: Array[String]) {
      println( factorial(0) )
      println( factorial(1) )
      println( factorial(2) )
      println( factorial(3) )
   }
   //阶乘计算器
   def factorial(i: Int): Int = {
	  //定义嵌套函数
      def fact(i: Int, accumulator: Int): Int = {
         if (i <= 1)
            accumulator
         else
            fact(i - 1, i * accumulator)
      }
	  //嵌套函数只能在函数内部调用
      fact(i, 1)
   }
}


/**
 * 8. 匿名函数: 将函数分配给一个变量
 */
//定义匿名函数
var inc = (x:Int) => x+1
//调用匿名函数: 变量inc现在是一种可以像函数那样使用的函数
var x = inc(7)-1
//定义具有多个参数的函数：
var mul = (x: Int, y: Int) => x*y
println(mul(3, 4))
//定义不带参数的函数
var userDir = () => { System.getProperty("user.dir") }
println(userDir)

/**
 * 9. 柯里化(Currying)函数: 相当于带参数的匿名函数
 *    带有多个参数，并引入到一个函数链中的函数，每个函数都使用一个参数。 
 *    柯里化(Currying)函数用多个参数表定义
 * def strcat(s1: String)(s2: String) = s1 + s2
 * 或者，还可以使用以下语法定义柯里化(Currying)函数 
 * def strcat(s1: String) = (s2: String) => s1 + s2
 */
object Demo {
   def main(args: Array[String]) {
      val str1:String = "Hello, "
      val str2:String = "Scala!"
      println( "str1 + str2 = " +  strcat(str1)(str2) )
   }
   def strcat(s1: String)(s2: String) = {
      s1 + s2
   }
}

/**
 * 10. 部分应用函数: 调用一个函数时，只传递几个参数并不是全部参数，
 *                 那么将返回部分应用的函数
 */
object Demo {
   def main(args: Array[String]) {
      val date = new Date
	  //调用log函数，只传date一个实参，返回部分应用函数
      val logWithDateBound = log(date, _ : String)
      //调用部分应用函数
      logWithDateBound("message1" )
      Thread.sleep(1000)
      logWithDateBound("message2" )
      Thread.sleep(1000)
      logWithDateBound("message3" )
   }
   def log(date: Date, message: String) = {
      println(date + "----" + message)
   }
}

```

## 闭包


## 字符串

```Scala
//创建字符串，创建之后字符串是不可变的，如果需要修改，可使用StringBuilder类
var greeting = "Hello world!"
var greeting1:String = "Hello world!"

//字符串长度
var len = greeting.length()
//字符串连接，也可使用+
string1.concat(string2)

//创建格式化字符串使用 %
var floatVar = 12.456
var intVar = 2000
var stringVar = "Hello, Scala!"
var fs = printf("floatVar is " + "%f, intVar is %d, string" + "is %s", floatVar, intVar, stringVar);

//★ 字符串插值(Scala-2.10及更高版本) : 直接在过程字符串文字中嵌入变量引用的机制
//三种类型(插值器)实现
//s’字符串插值器:允许在处理字符串时直接使用变量
val name = "James"
println(s"Hello, $name")      //Hello, James
println(s"1 + 1 = ${1 + 1}")  //1+1=2

//‘f’插值器: 允许创建一个格式化的字符串，类似于C语言中的printf.所有变量引用都应该是printf样式格式说明符，如％d，％i，％f等。
val height = 1.9d
val name = "James"
println(f"$name%s is $height%2.2f meters tall") //James is 1.90 meters tall

//“原始”插值器: 'raw'内插器类似于's'插值器，除了它不执行字符串内的文字转义
println(s"Result = \n a \n b")   //支持\n换行转义
println(raw"Result = \n a \n b") //不支持转义，Result = \n a \n b








```

## 数组

数组是一种存储了相同类型元素的固定大小顺序集合。通过下标arr(index)访问。
Scala不直接支持各种数组操作，它提供各种方法来处理任何维度的数组。如果要
使用不同的方法，则需要导入`Array._`包。有关可用方法的完整列表，请查看Scala的官方文档。

```Scala
var z:Array[String] = new Array[String](3)
//或者
var z = new Array[String](3)
var z = Array("Maxsu", "Nancy", "Alen")
//数组遍历
var myList = Array(1.9, 2.9, 3.4, 3.5)
for ( x <- myList ) {
 println( x )
}
var total = 0.0;
for ( i <- 0 to (myList.length - 1)) {
 total += myList(i);
}

//以下是使用数组时可以使用的重要方法。如上所示，必须在使用任何上述方法之前导入Array._包
//创建一个T对象数组，其中T可以是：Unit，Double，Float，Long，Int，Char，Short，Byte，Boolean
def apply( x: T, xs: T* ): Array[T]
//将所有数组连接成一个数组
def concat[T]( xss: Array[T]* ): Array[T]
//将一个数组复制到另一个。 相当于Java的System.arraycopy(src，srcPos，dest，destPos，length)
def copy( src: AnyRef, srcPos: Int, dest: AnyRef, destPos: Int, length: Int ): Unit
//返回长度为0的数组
def empty[T]: Array[T]
//返回一个包含函数的重复应用程序到一个起始值的数组
def iterate[T]( start: T, len: Int )( f: (T) => T ): Array[T]
//返回一个包含某些元素计算结果的数组
def fill[T]( n: Int )(elem: => T): Array[T]
//返回一个包含一些元素计算结果的二维数组
def fill[T]( n1: Int, n2: Int )( elem: => T ): Array[Array[T]]
//返回一个包含一个函数的重复应用程序到一个起始值的数组
def iterate[T]( start: T, len: Int)( f: (T) => T ): Array[T]
//创建具有给定维度的数组
def ofDim[T]( n1: Int ): Array[T]
//创建一个二维数组
def ofDim[T]( n1: Int, n2: Int ): Array[Array[T]]
//创建一个3维数组
def ofDim[T]( n1: Int, n2: Int, n3: Int ): Array[Array[Array[T]]]
//返回一个整数间隔中包含相等间隔值的数组
def range( start: Int, end: Int, step: Int ): Array[Int]
//返回一个包含一个范围内增加整数序列的数组
def range( start: Int, end: Int ): Array[Int]
//从0开始的整数值范围内，返回一个包含给定函数值的数组
def tabulate[T]( n: Int )(f: (Int)=> T): Array[T]
//返回一个二维数组，其中包含从0开始的整数值范围内给定函数的值
def tabulate[T]( n1: Int, n2: Int )( f: (Int, Int ) => T): Array[Array[T]]
```



## 集合

## 模式匹配

模式匹配是Scala函数值和闭包后第二大应用功能。模式匹配包括一系列备选项，每个替代项以关键字
大小写为单位。每个替代方案包括一个模式和一个或多个表达式，如果模式匹配，将会进行评估计算。
箭头符号`=>`将模式与表达式分离。具有case语句的块定义了一个将整数映射到字符串的函数。match
关键字提供了一种方便的方式来应用一个函数(如上面的模式匹配函数)到一个对象。

使用case类匹配: case类是用于与case表达式进行模式匹配的特殊类。

```Scala
object Demo {
   def main(args: Array[String]) {
      println(matchTest("two"))   //2
      println(matchTest("test"))  //many
      println(matchTest(1))       //one
   }
   def matchTest(x: Any): Any = x match {
      case 1 => "one"
      case "two" => 2
      case y: Int => "scala.Int"
      case _ => "many"
   }
}

object Demo {
   def main(args: Array[String]) {
      val alice = new Person("Alice", 25)
      val bob = new Person("Bob", 32)
      val charlie = new Person("Charlie", 32)
      for (person <- List(alice, bob, charlie)) {
         person match {
            case Person("Alice", 25) => println("Hi Alice!")
            case Person("Bob", 32) => println("Hi Bob!")
            case Person(name, age) => println(
               "Age: " + age + " year, name: " + name + "?")
         }
      }
   }
   //使用case类匹配
   case class Person(name: String, age: Int)
}
```

## 提取器

Scala中提取器是一个拥有unapply()方法的对象，该方法目的是匹配一个值并将其分开。通常提取器对象还定义一种用于
构建值的双重方法apply()，但不是必须的。



## 异常处理

```Scala
object Demo {
   def main(args: Array[String]) {
      try {
         val f = new FileReader("input.txt")
      } catch {
         case ex: FileNotFoundException => {
            println("Missing file exception")
         }
         case ex: IOException => {
            println("IO Exception")
         }
      } finally {
         println("Exiting finally...")
      }
   }
}
```





## 文件I/O
  



# 面向对象

所有面向对象语言在概念上都是相通的，都有类、对象、继承、this、单例、构造函数、方法覆盖等，
唯一不同的就是表现方式，或者同一个概念在不同语言中叫法不同（比如static和object），下面列
举Scala中与Java表示有差异的相关内容，其他没有列举的都和Java差不多，java中怎么使用就怎么
使用，如果有问题查一查就行。

## 类和对象

```Scala
//定义类，类名后可带参数，作为构造方法
class Person(val cname:String){
	var name: Stirng = cname
	def eat(food : String){
		println (name+ " 吃 " + food)
	}
	def work(time : int){
		println (name+ " 需要工作 " + time +"小时")
	}
}

//extends继承类，方法重写需要override关键字，只有主构造函数可以通过参数调用基类构造函数
class Student(override val cname: String, val sNum :Int) 
		extends Person(cname){
   var num: Int = sNum
   
   override def work(time : int){
		println (name+ " 需要学习 " + time +"小时")
	}
}
object Demo {
   def main(args: Array[String]) {
	  //使用关键字new来创建对象
      val openxu = new Student("openxu", 2);
      openxu.work(8);
   }
}
```

## 隐性类

当类在范围内时，隐式类允许与类的主构造函数进行隐式对话。隐式类是一个标有`implicit`关键字的类。此功能在Scala 2.10中引入。

```Scala
object Run {
   //创建一个名为IntTimes的隐式类，为其定义一个times方法，该方法为高阶函数（参数是函数），times方法中有一个嵌套函数
   implicit class IntTimes(x: Int) {
      def times [A](f: =>A): Unit = {
		  //定义嵌套函数
         def loop(current: Int): Unit =
         if(current > 0){
            f
            loop(current - 1)
         }
         loop(x)
      }
   }
}

//导入隐式函数
import Run._

object Demo {
   def main(args: Array[String]) {
      4 times println("hello")
   }
}

```

## 构造函数

**默认主构造函数**

定义类时如果不指定主构造函数，编译器将创建一个默认的主构造函数，相当于Java中的默认构造函数。

```Scala
class Student{  
    println("Hello from default constructor");  
}
```

**主构造函数**

如果只有一个构造函数，则可以不需要定义明确的构造函数。可以定义具有零个或多个参数的主构造函数

```Scala
class Student(id:Int, name:String){  
    def showDetails(){  
        println(id+" "+name);  
    }  
} 
```

**次要(辅助)构造器**

可以在类中创建任意数量的辅助构造函数，必须要从辅助构造函数内部调用主构造函数。`this`关键字用于
从其他构造函数调用构造函数。当调用其他构造函数时，要将其放在构造函数中的第一行。

```Scala
class Student(id:Int, name:String){  
    var age:Int = 0  
    def showDetails(){  
        println(id+" "+name+" "+age)  
    }  
	//定义辅助构造器
    def this(id:Int, name:String,age:Int){  
		//在辅助构造器中调用主构造器，必须放在第一行
        this(id,name)      
        this.age = age  
    }  
}  

object Demo{  
    def main(args:Array[String]){  
        var s = new Student(1010,"Maxsu", 25);  
        s.showDetails()  
    }  
}
```

## 单例和伴生对象

Java中的单例是通过私有构造函数和静态方法创建实例实现的，而Scala中没有静态static的概念，是通
过object关键字声明单例类。

**单例示例**

```Scala
object Singleton{  
    def main(args:Array[String]){  
		//不需要创建对象，直接调用单例类中的方法，相当于静态方法
        SingletonObject.hello()
    }  
}  
//创建单例对象
object SingletonObject{  
    def hello(){  
        println("Hello, This is Singleton Object")  
    }  
}
```

**伴生对象**

由于Scala中没有static，如果一个类需要定义静态方法或者成员，这时就需要为该类创建一个伴生对象，
伴生对象名称和类名一样，需要定义在同一个源文件中，类称为对象的伴生类，对象称为类的伴生对象。
伴生类和伴生对象可相互访问其私有方法和成员。下面示例模拟Java中单例的实现:

```Scala
//创建一个私有构造方法的Manager类，该类不能在外部通过new创建对象
class NetManager privete(val ip:String){
	var this.ip = ip
}
//创建该类的伴生对象，用于定义静态成员
object NetManager{
	var namager : NetManager
	//定义单例方法
	def getInstance() : NetManager={
		if(namager == null){
			namager = new NetManager("127.0.0.1:8080")
		}
		manager
	}
}
```

## Case类

Scala中的case类默认实现了toString、hashCode、copy、equals方法，通常用于模式匹配，
叫做样例类。case类创建对象不需要new(可选)，可序列化(实现类Serializable)，不能修改属性值。









