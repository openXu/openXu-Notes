

# 基础组件

## Widget简介

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















