[《Flutter实战》](https://book.flutterchina.club/)

# 基础组件

## 1. Widget简介

在Flutter中几乎所有的对象都是一个`Widget`。与原生开发中“控件”不同的是，Flutter中的Widget的概念更广泛，它不仅可以表示UI元素，也可以表示一些功能性的组件如：用于手势检测的`GestureDetector`、用于APP主题数据传递的Theme等等，而原生开发中的控件通常只是指UI元素。

在Flutter中，`Widget`的功能是**描述一个UI元素的配置数据**，它就是说，Widget其实并不是表示最终绘制在设备屏幕上的显示元素，而它只是描述显示元素的一个配置数据。实际上，Flutter中真正代表屏幕上显示元素的类是`Element`，也就是说Widget只是描述Element的配置数据！一个Widget可以对应多个Element。这是因为同一个Widget对象可以被添加到UI树的不同部分，而真正渲染时，UI树的每一个Element节点都会对应一个Widget对象。

```Java
@immutable
//Widget类继承自DiagnosticableTree，DiagnosticableTree即“诊断树”，主要作用是提供调试信息
abstract class Widget extends DiagnosticableTree {
  const Widget({ this.key });
  //主要的作用是决定是否在下一次build时复用旧的widget，决定的条件在canUpdate()方法中
  final Key key;

  //Flutter Framework在构建UI树时，会先调用此方法生成对应节点的Element对象。此方法是Flutter Framework隐式调用的，在我们开发过程中基本不会调用到。
  @protected
  Element createElement();

  @override
  String toStringShort() {
    return key == null ? '$runtimeType' : '$runtimeType-$key';
  }
  //复写父类的方法，主要是设置诊断树的一些特性。
  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.defaultDiagnosticsTreeStyle = DiagnosticsTreeStyle.dense;
  }
  //复写父类的方法，主要是设置诊断树的一些特性。
  static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
  }
}
```

Widget类本身是一个抽象类，其中最核心的就是定义了createElement()接口，在Flutter开发中，我们一般都不用直接继承Widget类来实现一个新组件，相反，我们通常会通过继承`StatelessWidget`或`StatefulWidget`来间接继承Widget类来实现。StatelessWidget和StatefulWidget都是直接继承自Widget类，而这两个类也正是Flutter中非常重要的两个抽象类，它们引入了两种Widget模型，接下来我们将重点介绍一下这两个类。

- widget的构造函数参数应使用命名参数，命名参数中的必要参数要添加@required标注，这样有利于静态代码分析器进行检查
- 另外，在继承widget时，第一个参数通常应该是Key
- 如果Widget需要接收子Widget，那么child或children参数通常应被放在参数列表的最后
- Widget的属性应尽可能的被声明为final，防止被意外改变。

### StatelessWidget

StatelessWidget用于不需要维护状态的场景，它通常在build方法中通过嵌套其它Widget来构建UI，在构建过程中会递归的构建其嵌套的Widget。



## 2. 布局类组件

- Row：行，横向的LinearLayout
- Column：列，垂直方向LinearLayout
- Flex：弹性布局，类似LinearLayout搭配weiget权重，children存放Expanded，设置flex占比
- Wrap：流式布局，自动换行，可设置方向是横向还是纵向
- Flow：相当于自定义ViewGroup，需要重写onLayout布局子控件，而它需要设置一个FlowDelegate重写paintChildren()计算子widget位置
- Stack：允许子组件堆叠，而`Positioned`用于根据`Stack`的四个角来确定子组件的位置
- 









### 2.1 线性布局（Row和Column）

```dart
//Row行：可以在水平方向排列其子widget，横向的LinearLayout
Row({
  ...  
  //水平方向子组件的布局顺序(是从左往右还是从右往左)，默认为系统当前Locale环境的文本方向
  TextDirection textDirection,    
  //主轴(水平)方向占用的空间, 默认是MainAxisSize.max(填充)表示尽可能多的占用水平方向的空间，MainAxisSize.min(包裹子widgets)
  MainAxisSize mainAxisSize = MainAxisSize.max,    
  //子组件水平空间内对齐方式，MainAxisAlignment.start左对齐；MainAxisAlignment.end正好相反；MainAxisAlignment.center表示居中对齐。
  //textDirection是mainAxisAlignment的参考系
  MainAxisAlignment mainAxisAlignment = MainAxisAlignment.start,
  //表示Row纵轴（垂直）的对齐方向，默认是VerticalDirection.down，表示从上到下
  VerticalDirection verticalDirection = VerticalDirection.down,  
  //子组件在纵轴方向的对齐方式，crossAxisAlignment.start、end、 center
  //参考系是verticalDirection
  CrossAxisAlignment crossAxisAlignment = CrossAxisAlignment.center,
  //children ：子组件数组
  List<Widget> children = const <Widget>[],
})

//Column可以在垂直方向排列其子组件。参数和Row一样，不同的是布局方向为垂直，主轴纵轴正好相反，可类比Row来理解
    
```

### 2.2 弹性布局（Flex）

`Flex`组件可以沿着水平或垂直方向排列子组件，如果你知道主轴方向，使用`Row`或`Column`会方便一些，因为`Row`和`Column`都继承自`Flex`，参数基本相同，所以能使用Flex的地方基本上都可以使用`Row`或`Column`。`Flex`本身功能是很强大的，它也可以和`Expanded`组件配合实现弹性布局。

```dart
//Flex继承自MultiChildRenderObjectWidget
Flex({
  ...
  @required this.direction, //弹性布局的方向, Row默认为水平方向，Column默认为垂直方向
  List<Widget> children = const <Widget>[],
})
    
//可以按比例“扩伸” Row、Column和Flex子组件所占用的空间
const Expanded({
  int flex = 1,   //弹性系数，0或null则child是没有弹性的，大于0，所有的Expanded按照其flex的比例来分割主轴的全部空闲空间
  @required Widget child,
})
```

### 2.3 流式布局（Wrap和Flow）

```dart
Wrap({
  ...
  this.direction = Axis.horizontal,      //方向
  this.alignment = WrapAlignment.start,  //主轴方向对齐方式
  this.spacing = 0.0,      //主轴方向子widget的间距
  this.runAlignment = WrapAlignment.start, //纵轴方向对齐方式
  this.runSpacing = 0.0,   //纵轴方向的间距
  this.crossAxisAlignment = WrapCrossAlignment.start,  //纵轴方向对齐方式
  this.textDirection,
  this.verticalDirection = VerticalDirection.down,   //纵轴排列方式，从上到下
  List<Widget> children = const <Widget>[],
})
```

```dart
Flow(
  delegate: TestFlowDelegate(margin: EdgeInsets.all(10.0)),
  children: <Widget>[
    new Container(width: 80.0, height:80.0, color: Colors.red,),
    new Container(width: 80.0, height:80.0, color: Colors.green,),
    new Container(width: 80.0, height:80.0, color: Colors.blue,),
    new Container(width: 80.0, height:80.0,  color: Colors.yellow,),
    new Container(width: 80.0, height:80.0, color: Colors.brown,),
    new Container(width: 80.0, height:80.0,  color: Colors.purple,),
  ],
)
    
    class TestFlowDelegate extends FlowDelegate {
  EdgeInsets margin = EdgeInsets.zero;
  TestFlowDelegate({this.margin});
  @override
  void paintChildren(FlowPaintingContext context) {
    var x = margin.left;
    var y = margin.top;
    //计算每一个子widget的位置  
    for (int i = 0; i < context.childCount; i++) {
      var w = context.getChildSize(i).width + x + margin.right;
      if (w < context.size.width) {
        context.paintChild(i,
            transform: new Matrix4.translationValues(
                x, y, 0.0));
        x = w + margin.left;
      } else {
        x = margin.left;
        y += context.getChildSize(i).height + margin.top + margin.bottom;
        //绘制子widget(有优化)  
        context.paintChild(i,
            transform: new Matrix4.translationValues(
                x, y, 0.0));
         x += context.getChildSize(i).width + margin.left + margin.right;
      }
    }
  }

  @override
  getSize(BoxConstraints constraints){
    //指定Flow的大小  
    return Size(double.infinity,200.0);
  }

  @override
  bool shouldRepaint(FlowDelegate oldDelegate) {
    return oldDelegate != this;
  }
}
```

### 2.4 层叠布局 Stack、Positioned

 Stack指定一个或多个子元素相对于父元素 Stack各个边的精确偏移，并且可以重叠

```dart
Stack({
  //如何去对齐没有定位（没有使用Positioned）或部分定位的子组件
  this.alignment = AlignmentDirectional.topStart,
  //确定alignment对齐的参考系，TextDirection.ltr从左往右的顺序，TextDirection.rtl从右往左的顺序
  this.textDirection,  
  //确定没有定位的子组件如何去适应Stack的大小，StackFit.loose表示使用子组件的大小，StackFit.expand表示扩伸到Stack的大小
  this.fit = StackFit.loose,
  this.overflow = Overflow.clip,
  //此属性决定如何显示超出Stack显示空间的子组件；值为Overflow.clip时，超出部分会被剪裁（隐藏），值为Overflow.visible 时则不会。
  List<Widget> children = const <Widget>[],
})
    
const Positioned({
  Key key,
  //left、top 、right、 bottom分别代表离Stack左、上、右、底四边的距离
  this.left, 
  this.top,
  this.right,
  this.bottom,
  //width和height用于指定需要定位元素的宽度和高度，用于配合left、top 、right、 bottom来定位组件
  this.width,
  this.height,
  @required Widget child,
})
```

```dart
//通过ConstrainedBox来确保Stack占满屏幕
ConstrainedBox(
  constraints: BoxConstraints.expand(),
  child: Stack(
    alignment:Alignment.center , //指定未定位或部分定位widget的对齐方式
    children: <Widget>[
      Container(child: Text("Hello world",style: TextStyle(color: Colors.white)),
        color: Colors.red,
      ),
      Positioned(
        left: 18.0,
        child: Text("I am Jack"),
      ),
      Positioned(
        top: 18.0,
        child: Text("Your friend"),
      )        
    ],
  ),
);
```



http://39.98.164.162:8080/group1/M00/00/0F/rBpzbmDB5HWAapNmAOITrGBThFA772.apk

http://39.98.164.162:8080/jeecg-boot/sys/permission/getPhoneUserPermissionByToken?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2MjMzODQ2MzMsInVzZXJuYW1lIjoiY3lmOTUifQ.rh2w5cqGaxwkkAPlJ3-qVR9ACGyLDIJFWnkIkl8zMso&applicationid=6774929446609883136
