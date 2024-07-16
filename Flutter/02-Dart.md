[Dart中文网](https://dart.cn/overview)

[Dart 开发语言概览](https://dart.cn/guides/language/language-tour)

## flutter为什么不使用kotlin作为开发语言？

kotlin是google官方支持的Android开发语言，并且kotlin也是用于现代多平台应用的静态编程语言（跨平台），而**Flutter是google推出的移动UI框架**，可以快速的iOS和Android上构建高质量的原生用户界面。为什么google不选择kotlin作为Flutter的开发语言？而选择了Dart？作为一个android开发者，从最初的Java、Gradle的groovy、然后又是kotlin、最后还要来个Dart，是想整死我们？

Kotlin是由JetBrains（开发了Android Studio和IntelliJ IDEA）于2011年7月推出的项目，其设计目的是**创建一种兼容Java的面向JVM的新语言，这个语言比Java安全（空安全）、简洁（各种语法糖），比Scala编译快，希望这个新语言能推动IntelliJ IDEA的销售，kotlin在安卓上本质上还是Java。**从设计的初衷看来Kotlin并不是要成为一个跨平台的语言，只是它可以被编译成Java字节码、也可以编译为JavaScript方便在没有JVM的设备上运行，除此之外还可以编译成二进制代码直接运行在机器上（嵌入式或者iOS），但是对于编译成二进制执行的支持不太好，因为并不是它设计的初衷。**Google之所以选择Kotlin作为Android开发第一语言根本原因还是Java的官司逼的。**并不是想着使用Kotlin开发的应用也可以运行在iOS设备上，如果是这样的话在iOS上装个jvm不是也能跑Android应用了？

Flutter构建的应用可以运行在iOS和Android或者Web上是因为它自带的渲染引擎，并不是靠原生渲染，所以能实现个平台的统一表现。Flutter的这种跨平台表现和选择哪种语言没关系，但是要求**选择的语言必须支持热加载、解释执行**这些特性，Kotlin也并不支持。**并且Dart是Google自己的语言，Dart和Flutter办公室仅有一墙之隔，更方便合作**。我们需要明白，Flutter仅仅是一个移动UI框架，是原生系统上层的东西，我们可以把它简单的看作是一个开源库，但是这个开源库的绘制不是依靠原生系统，而是自带的渲染引擎。

作为程序员，面对这么多的平台和语言，确实面临不小的挑战和困惑，但是多学几门语言也是必不可少的。况且这些面向对象的语言都是相通的，不同的只是表现形式罢了，我们只需要了解语言的基本语法、类、函数、内置数据类型的使用、以及其新特点就能上手开发了，况且现在的IDE这么强大，根本不需要我们死记硬背语法，自动补全会让我们忘记语言之间的差异。比如现在要上手Dart，只需要看一下[Dart 开发语言概览](https://dart.cn/guides/language/language-tour)，记性不好的可以将重点的内容和特性通过自己的方式记录下来，看的慢2天也能解决了，2天上手一门语言效率不要太高。





# 1. 语言特性

**变量：**虽然 Dart 是代码类型安全的语言，但是由于其支持类型推断，因此大多数变量不需要显式地指定类型

```dart
var name = '旅行者一号';
var year = 1977;
var antennaDiameter = 3.7;
var flybyObjects = ['木星', '土星', '天王星', '海王星'];
var image = {
  'tags': ['土星'],
  'url': '//path/to/saturn.jpg'
};
```

**函数：**和java一样，为函数的参数和返回值都指定类型

```dart
int fibonacci(int n) {
  if (n == 0 || n == 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
var result = fibonacci(20);
```

**注释：** //普通注释，///文档注释，/*多行注释*/

**导包Import：** 

```dart
// 导入核心库
import 'dart:math';
// 从外部 Package 中导入库
import 'package:test/test.dart';
// 导入文件
import 'path/to/my_other_file.dart';
```

**接口和实现&抽象和继承**：Dart 没有`interface` 关键字，所有的类都可以被`implements`。Dart 支持单继承。

```dart
abstract class Describable {
  void describe();
  void describeWithEmphasis() {
    describe();
  }
}
class Orbiter extends Spacecraft implements Describable { //extends继承  implements实现
  double altitude;
  // 构造函数，带有可以直接为成员变量赋值的语法糖this.xxx
  Orbiter(String name, DateTime launchDate, this.altitude)
      : super(name, launchDate);  //:super调用父构造函数
  // 命名构造函数，转发到默认构造函数
  Orbiter.unlaunched(String name) : this(name, null, 0);
}

```

**命名构造方法：为了允许一个类具有多个构造方法，Dart 支持命名构造方法**

```dart
class Point {
  num x, y;
  Point(this.x, this.y);
  Point.origin() {  //命名构造方法格式：类名.方法名(){}   
    x = 0;
    y = 0;
  }
}
//全名调用 命名构造方法
final myPoint = Point.origin();
```

**工厂构造方法：**`factory` 关键字定义的构造方法能够返回其子类甚至 null 对象

```dart
class Square extends Shape {}
class Circle extends Shape {}

class Shape {
  Shape();
    //factory关键字创建一个工厂构造方法
  factory Shape.fromTypeName(String typeName) {
    if (typeName == 'square') return Square();  //返回子类对象
    if (typeName == 'circle') return Circle();
    print('I don\'t recognize $typeName');
    return null;   //返回空
  }
}
```

**重定向构造方法：**一个构造方法仅仅用来重定向到该类的另一个构造方法，重定向方法没有主体，它在冒号`:`之后调用另一个构造方法

```dart
class Automobile {
  String make;
  String model;
  int mpg;
  // 主构造函数
  Automobile(this.make, this.model, this.mpg);
  // 委托给主构造函数
  Automobile.hybrid(String make, String model) : this(make, model, 60);
  // 委托给命名函数
  Automobile.fancyHybrid() : this.hybrid('Futurecar', 'Mark 2');
}
```

**Const 构造方法：**如果你的类生成的对象永远都不会更改，则可以让这些对象成为编译时常量。为此，请定义 `const` 构造方法并确保所有实例变量都是 final 的。

```dart
class ImmutablePoint {
  //Const 构造方法用于构建一个常量对象，该对象中的属性值用不改变（final修饰）
  const ImmutablePoint(this.x, this.y);
  final int x;
  final int y;
  static const ImmutablePoint origin =
      ImmutablePoint(0, 0);
}
```



**Mixin：**是一种在多个类层次结构中重用代码的方法，和kotlin的扩展类似但有着本质区别，是扩展和继承的合体。 **使用 Mixin 为类添加功能，重点是添加（增量）**

```dart
mixin Piloted {   //声明一个Mixin，相当于一个类，这个类并不能调用重用它的类，所以和kotlin扩展有区别，kotlin扩展是静态的工具函数，函数中有隐式的this表示被扩展的对象
  int astronauts = 1;
  void describeCrew() {
    print('Number of astronauts: $astronauts');
  }
}
//重用代码：使用with继承这个类就可将该类中的功能添加给其它类，相当于是继承
class PilotedCraft extends Spacecraft with Piloted {
  // ···
}
```

**字符串插值：**在字符串中使用`${表达式}`引用表达式值，如果表达式为单个标识符，可省略`{}`

**避空运算符：**Dart 提供了一系列方便的运算符用于处理可能会为空值的变量

```dart
//1. ??= 仅当变量为空值时才赋值
int a; 
a ??= 3;  
print(a); // <-- Prints 3.
a ??= 5;
print(a); // <-- Still prints 3.
//2. ?? 如果该运算符左边的表达式返回的是空值，则会计算并返回右边的表达式。
print(1 ?? 3); // <-- Prints 1.
print(null ?? 12); // <-- Prints 12.
```

**条件属性访问器：**`?.`保护可能会为空的属性的正常访问`myObject?.someProperty?.someMethod()`，相当于kotlin中的空安全调用

**胖箭头=>：**定义函数的方法，将执行箭头右侧表达式作为函数返回值，其实就是lambda表达式，但是=>的右侧只能是表达式

```dart
bool hasEmpty = aListOfStrings.any((s) {
    ...//如果函数有多行，不能用=>，必须使用{}括起来
  return s.isEmpty;
});
//函数只有一行时，使用=>简写
bool hasEmpty = aListOfStrings.any((s) => s.isEmpty);

```

**级连调用：**级联调用的返回值是对象的引用，这样就可以连续调用同一个对象的属性或者方法

```dart
var button = querySelector('#confirm')
..text = 'Confirm'
..classes.add('important')
..onClick.listen((e) => window.alert('Confirmed!'));
```

**getter & setter：**需要对属性进行更多控制而不是简单的字段访问时，可自定义getter和setter

```dart
class MyClass {
  //Dart 没有类似于 Java 那样的 public、protected 和 private 成员访问限定符。如果一个标识符以下划线_开头则表示该标识符在库内是私有的
  int _aProperty = 0;   
  int get aProperty => _aProperty;  //自定义aProperty属性的getter
  set aProperty(int value) {    //自定义aProperty的setter
    if (value >= 0) {
      _aProperty = value;
    }
  }
}
```

**可选位置参数：**Dart 有两种传参方法：位置参数和命名参数

```dart
//位置参数
int sumUp(int a, int b, int c) {
  return a + b + c;
}
int total = sumUp(1, 2, 3);
//将参数包裹在[]中表示可选位置参数，可选位置参数永远放在参数列表最后，除非提供了默认值，否则默认为null
int sumUpToFive(int a, [int b, int c, int d, int e]) {
  int sum = a;
  if (b != null) sum += b;
  if (c != null) sum += c;
  if (d != null) sum += d;
  if (e != null) sum += e;
  return sum;
}
int total = sumUptoFive(1, 2);
int otherTotal = sumUpToFive(1, 2, 3, 4, 5);

//{}可选命名参数，一个方法不能同时使用可选位置参数和可选命名参数
void printName(String firstName, String lastName, {String suffix}) {
  print('$firstName $lastName ${suffix ?? ''}');
}
printName('Avinash', 'Gupta');
printName('Poshmeister', 'Moneybuckets', suffix: 'IV');
```

**异常**

```dart
try {
  breedMoreLlamas();
} on OutOfLlamasException {  //使用 on 关键字按类型过滤特定异常
  buyMoreLlamas();
} on Exception catch (e) {   //catch 关键字能够获取捕捉到的异常对象的引用
  print('Unknown exception: $e');
} catch (e) {
  print('Something really unknown: $e');
  rethrow;            //如果你无法完全处理该异常，请使用 rethrow 关键字再次抛出异常
}finally {
  cleanLlamaStalls();
}
```

# 2. 数据类型

Dart 语言内置数据类型：`int`、 `double`、`String`、`bool`、`Lists(arrays)`、`Set`、`Map`、`null`



## 2.1 集合

### 集合字面量

Dart 内置了对 list、map 以及 set 的支持。你可以通过字面量直接创建它们：

```dart
final aListOfStrings = ['one', 'two', 'three'];
final aSetOfStrings = {'one', 'two', 'three'};
final aMapOfStringsToInts = {
  'one': 1,
  'two': 2,
  'three': 3,
};
```

Dart 的类型推断可以自动帮你分配这些变量的类型。在这个例子中，推断类型是 `List<String>`、`Set<String>`和 `Map<String, int>`。你也可以手动指定类型：

```dart
final aListOfInts = <int>[];
final aSetOfInts = <int>{};
final aMapOfIntToDouble = <int, double>{};
```



# 3. 异步编程：futures，async，await







# 4. 版本提示



- 从 Dart 2 开始，`new` 关键字是可选的。
- 只有从 Dart 2 开始才能根据上下文判断省略 `const` 关键字。
- 在 Dart 2.1 之前，在浮点数上下文中使用整数字面量是错误的。
- 在 Dart 2.12中引入了**空安全性**。使用空安全性需要至少2.12的语言版本。











[48 48 00 cf 00 10 05 39 00 01 10 00 40 00 01 01 00 00 dd ab c1 7a 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 80 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 50 27 10 00 00 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 63 00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00 08 00 09 00 0a 00 0b 00 0c 00 0d 00 0e 00 0f 00 10 00 11 00 12 00 13 00 14 00 15 00 16 00 17 00 18 00 19 00 1a 00 1b 00 1c 00 1d 00 1e 00 1f 00 20 00 21 00 22 00 23 00 24 00 25 00 26 00 27 00 28 00 29 00 2a 00 2b 00 2c 00 2d 00 2e 00 2f 00 30 00 31 00 32 00 33 00 34 00 35 00 36 00 37 00 38 00 39 00 3a 00 3b 00 3c 00 3d 00 3e 00 3f 00]















