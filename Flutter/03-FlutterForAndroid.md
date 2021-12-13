

[Flutter for Android 开发者](https://flutterchina.club/flutter-for-android/)通过原生Android开发和Flutter开发中的相关知识对比，让我们快速了解Flutter构建移动应用程序相关知识点：

# 1. Views

## 1.1 Flutter和Android中的View

在Android中View是屏幕上显示的所有内容的基础，在Flutter中，**View相当于是Widget**。Widget仅表示一帧，每次更新状态Flutter都会创建一个Widget实例树(相当于一次性绘制整个界面)。而Android上View绘制结束后，就不会重绘，直到调用invalidate时才会重绘对应的View。

与Android的视图层次系统不同（在framework改变视图），而在Flutter中的widget是不可变的，这允许widget变得超级轻量。

## 1.2 如何更新widget

Widget分为Stateful和Stateless，`StatelessWidget`是不可变的，它每次刷新绘制时都是相同的。而`StatefulWidget`是可变化的，它由具有不同的生命周期State和Widget两个对象组成，Widget是临时的一帧，仅用于构建当前状态下的页面，但是State状态是可以变化的，多次调用State.build()只会构建新的Widget，State对象还是原来的对象。通常在数据发生变化后通过`State.setState()`来触发`State.build()` 更新UI。就像Android的`View.invalidate()`，只是Flutter将View和数据状态分成了两个类。在开发时只要记住**如果页面上的某个部分Widget可能会发生变化，它就是有状态的，但是它的父Widget不一定是有状态的**

## 1.3 如何布局？ XML layout 文件跑哪去了？

- Android通过xml编写布局，然后通过Activity加载解析布局为VIew对象树结构后绘制
- Flutter则直接使用Dart语言编写Widget树，在构建时执行`widget.build()`形成数结构后完成绘制

## 1.4 如何在布局中添加或删除组件

- Android：通过parentView的`addChild()`或`removeChild()`动态添加或删除View
- Flutter：由于widget是不可变的，所以没有类似方法。但是可以使用`StatefulWidget `在代码中通过布尔值状态控制widget的创建。

```dart
class SampleAppPage extends StatefulWidget {
  @override
  _SampleAppPageState createState() => new _SampleAppPageState();
}

class _SampleAppPageState extends State<SampleAppPage> {
  bool toggle = true;  //布尔类型的状态值
  //2. 状态值改变后触发build()方法刷新UI
  void _toggle() {
  	
    setState(() {
      toggle = !toggle;
    });
  }
  _getToggleChild() {
    //3. 通过状态值构建不同的Widget
    if (toggle) {
      return new Text('Toggle One');
    } else {
      return new MaterialButton(onPressed: () {}, child: new Text('Toggle Two'));
    }
  }
  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(
        title: new Text("Sample App"),
      ),
      body: new Center(
        child: _getToggleChild(),
      ),
      floatingActionButton: new FloatingActionButton(
        onPressed: _toggle,   //1. 点击按钮后状态变化
        tooltip: 'Update Text',
        child: new Icon(Icons.update),
      ),
    );
  }
}
```

## 1.5 Flutter中的动画

### 1.5.1 Android中的动画

Android中的动画在早期有传统的帧动画（FrameAnimation）和补间动画（TweenedAnimation）：
**帧动画**就是类似放电影连续播放一些列图片形成的视觉上的动画效果，其实现如下：

```xml
<!--在drawable目录下通过<animation-list>标签组织需要播放的图片-->
<animation-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:drawable="@drawable/a_0"
        android:duration="100" />
    <item
        android:drawable="@drawable/a_1"
        android:duration="100" />
    ...
</animation-list>
```

```java
//在代码中通过ImageView加载动画后，获取AnimationDrawable后调用start()播放动画
ImageView imageView = findViewById(R.id.imageView);
imageView.setImageResource(R.drawable.frame_anim);
AnimationDrawable animationDrawable = (AnimationDrawable) imageView.getDrawable();
animationDrawable.start();
```

**补间动画**是施加给`View`上的动画，控制`View`的`aplha`(透明度淡入淡出)、`translate`(位移)、`scale`(大小缩放)、`rotate`(旋转)，补间动画可以通过xml定义，也可以使用代码定义：

```xml
<!--在anim资源下，定义一个控制透明度的动画-->
<alpha xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="1000"
    android:fromAlpha="1.0"
    android:interpolator="@android:anim/accelerate_decelerate_interpolator"
    android:toAlpha="0.0" />
```

```java
//方式1：在代码中使用AnimationUtils加载动画
Animation animation = AnimationUtils.loadAnimation(mContext, R.anim.alpha_anim);
//方式2：定义一个旋转动画，锚点是相对View自身宽度的50%，高度的50%(中心点)，从0旋转360度
        RotateAnimation animation = new RotateAnimation(0, 360, Animation.RELATIVE_TO_SELF,
                0.5f, Animation.RELATIVE_TO_SELF, 0.5f);
//view开始动画
imageView.startAnimation(animation);
```

传统的动画都是视觉上的动画，补间动画并不是真的对View的属性进行改变，所以就产生一个问题，当View移动到别的位置后，点击View没反应，应为View的位置还是在动画之前的地方。**Android3.0后出现了属性动画**，它是通过不断的对对象（不局限于View）的某个属性值进行变化重绘而产生的，这个值就是通过`ValueAnimator`来计算的（从初始值按照一定规律变化到结束值）。属性动画可以完全代替补间动画。属性动画的核心类结构如下：

```java
Animator            //动画抽象类，在给定的时间段内会线性的生成从0.0到1.0的数字
    |-- AnimatorSet //组合动画，可以同时执行一系列动画
    |-- ValueAnimator //提供了计算引擎用于计算动画值，从初始值到最终值的变化
    	   |-- ObjectAnimator //★在目标对象上设置属性的动画，也就是改变目标对象的某个属性值
    	   |-- TimeAnimator   //为动画提供了一个简单的回调机制，不常用
```

属性动画还有两个重要的类：`Interpolator`插值器用于对Animator产生的线性变化数字施加某种公式从而控制动画的节奏(线性均匀变化、先快后慢...)。`TypeEvaluator`属性评估器用于根据插值器计算的值计算出当前的属性值（属性值是什么类型就要设置什么类型的评估器）：

- `Animator `：在给定的时间段线性均匀的生成0.0-1.0的数字 `a`
- `Interpolator` ：将a施加一种公式得到另一个数字b，控制变化速度。比如`b = Math.cos((a+ 1) * Math.PI) / 2.0f+ 0.5f `开始和结束慢，中间加速
- `TypeEvaluator`：根据`b`计算出对应的属性值，它的返回值类型是`Object`，意思动画可以施加给任何类型，比如Color、Float、Point........

```java
//自定义估值器，sdk自带的有ArgbEvaluator(颜色变化)、FloatEvaluator(Float值变化)、IntEvaluator(Int值变化)、RectEvaluator(矩形变化)等等
public class PointSinEvaluator implements TypeEvaluator {
    @Override
    public Object evaluate(float fraction, Object startValue, Object endValue) {
        Point startPoint = (Point) startValue;
        Point endPoint = (Point) endValue;
        //Point的x坐标根据动画进度增大，实现左到右的动画效果
        float x = startPoint.getX() + fraction * (endPoint.getX() - startPoint.getX());
        //y坐标根据sin公式变化
        float y = (float) (Math.sin(x * Math.PI / 180) * 100) + endPoint.getY() / 2;
        Point point = new Point(x, y);
        return point;  //返回最终的属性值
    }
}

public class PointAnimView extends View {
    ...
    public void startMyAnima(){
        Point startP = new Point(RADIUS, RADIUS);//初始值
        Point endP = new Point(getWidth() - RADIUS, getHeight() - RADIUS);//结束值
        //定义一个值动画，对Point进行改变
        final ValueAnimator valueAnimator = ValueAnimator.ofObject(new PointSinEvaluator(), startP, endP);
        //设置变化率开始和结束缓慢，但从中间加速的插值器，sdk提供了很多插值器，也可以自己定义
        valueAnimator.setInterpolator(AccelerateDecelerateInterpolator());
        valueAnimator.setRepeatCount(-1);  //无限重复
        valueAnimator.setRepeatMode(ValueAnimator.REVERSE); //往复执行
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                //监听动画得到属性值，重绘
                currentPoint = (Point) animation.getAnimatedValue();
                postInvalidate();
            }
        });
    }
}
```

### 1.5.2 Flutter中的动画

Flutter动画和Android的属性动画思想是一样的，甚至连组成部分相关类都是雷同的，对比如下：

|        Android         |          Flutter           |                             说明                             |
| :--------------------: | :------------------------: | :----------------------------------------------------------: |
|        Animator        |         Animation          |         动画核心类（抽象类）：提供0-1线性变化的数字a         |
|     ValueAnimator      |    AnimationController     | 动画控制器：用于决定动画施加在什么类型的对象上，它是上面抽象类的子类 |
|      Interpolator      |      CurvedAnimation       |      插值器：将a施加某种算法得到数字b，控制动画速度节奏      |
|     TypeEvaluator      |           Tween            | 属性评估器：Tween在正在执行动画的对象所使用的数据范围之间生成值。例如，Tween可能会生成从红到蓝之间的色值，或者从0到255。 |
| AnimatorUpdateListener | Listeners和StatusListeners |                       监听动画状态改变                       |
|     ObjectAnimator     |       AnimatedWidget       | 将动画和视图（View、Widget）绑定，动画执行时自动刷新UI，省去了`postInvalidate()`或者`setState()` |

```dart

//★使用步骤
//1. 创建动画控制器，vsync参数防止屏幕外动画消耗不必要的资源
AnimationController controller = new AnimationController(duration: const Duration(milliseconds: 2000), vsync: this);
//2. 定义插值器，将动画传递给它，Curves.easeIn是一个常量值，控制动画开始缓慢，结束迅速
CurvedAnimation curve = new CurvedAnimation(parent: controller, curve: Curves.easeIn);
//3. 将动画施加在Color类型的开始和结束值上，从而得到一个具体的数据类型的值，这里是Color类型
Animation<Color?> colorAnima = new ColorTween(begin: Colors.transparent, end: Colors.black54).animate(curve);
//4. 监听动画
colorAnima.addListener(() {
    //6. 监听动画后通过value属性获取动画当前值
    colorAnima.value;
});
//5. 开始动画
controller.forward();

//★使用示例：通过一个逐渐放大的动画显示logo
import 'package:flutter/animation.dart';
import 'package:flutter/material.dart';
void main() {
  runApp(new LogoApp());
}

//由于动画会使UI变化，所以应该使用StatefulWidget
class LogoApp extends StatefulWidget {
  _LogoAppState createState() => new _LogoAppState();
}

class _LogoAppState extends State<LogoApp> with SingleTickerProviderStateMixin {
  //将Animation对象存储为Widget的成员，然后使用其value值来决定如何绘制
  late Animation<double> animation;
  late AnimationController controller;
  @override
  initState() {
    super.initState();
    //1. 创建动画控制器
    controller = new AnimationController(
        duration: const Duration(milliseconds: 2000), vsync: this);
    //2. 将动画施加在double类型数值上，得到一个动画对象，并设置监听
    animation = new Tween(begin: 0.0, end: 300.0).animate(controller)
      ..addListener(() {  //..级联调用，也就是连续调用
        setState(() {  //4. 动画执行时，setState()会触发build()方法刷新UI
        });
      });
    controller.forward();  //3. 开始执行动画
  }

  Widget build(BuildContext context) {
    return new Center(
      child: new Container(
        margin: new EdgeInsets.symmetric(vertical: 10.0),
        height: animation.value, //5. 在build()中，改变container大小
        width: animation.value,
        child: new FlutterLogo(),
      ),
    );
  }

  dispose() {
    //6. 动画完成时释放控制器(调用dispose()方法)以防止内存泄漏。
    controller.dispose();
    super.dispose();
  }
}
```

#### 用AnimatedWidget简化

从示例中我们可以看出，动画Animation本身和UI渲染没有任何关系，动画只是按照某种公式计算出一个值（beginValue~endValue），而最终能产生可见的动画效果是通过UI渲染完成的。在Android中可以调用`postInvalidate()`重新绘制，Flutter中通过在`addListener()`中调用`setState()`触发Widget的`build()`重新渲染。

可以通过`AnimatedWidget`助手类来将动画和Widget合并，而简化`addListener()`和`setState()`等步骤给Widget添加动画。Flutter API提供了关于`AnimatedWidget`的示例包括：`AnimatedBuilder`、`AnimatedModalBarrier`、`DecoratedBoxTransition`、`FadeTransition`、`PositionedTransition`、`RelativePositionedTransition`、`RotationTransition`、`ScaleTransition`、`SizeTransition`、`SlideTransition`。

使用`AnimatedWidget`改造上面示例：

```dart
import 'package:flutter/animation.dart';
import 'package:flutter/material.dart';
void main() {
  runApp(new LogoApp());
}
class LogoApp extends StatefulWidget {
  _LogoAppState createState() => new _LogoAppState();
}

class _LogoAppState extends State<LogoApp> with SingleTickerProviderStateMixin {
  AnimationController controller;
  Animation<double> animation;

  initState() {
    super.initState();
    controller = new AnimationController(
        duration: const Duration(milliseconds: 2000), vsync: this);
    animation = new Tween(begin: 0.0, end: 300.0).animate(controller);
    controller.forward();
  }

  Widget build(BuildContext context) {
    //★ LogoApp将Animation对象传递给AnimatedWidget，AnimatedWidget(基类)中会自动调用addListener()和setState()，因此它的工作原理与之前完全相同
    return new AnimatedLogo(animation: animation);
  }

  dispose() {
    controller.dispose();
    super.dispose();
  }
}

//自定义AnimatedWidget
class AnimatedLogo extends AnimatedWidget {
  AnimatedLogo({Key key, Animation<double> animation})
      : super(key: key, listenable: animation);

  Widget build(BuildContext context) {
    final Animation<double> animation = listenable;  //获取动画对象
    return new Center(
      child: new Container(
        margin: new EdgeInsets.symmetric(vertical: 10.0),
        height: animation.value,  //AnimatedWidget在绘制时使用动画的当前值
        width: animation.value,
        child: new FlutterLogo(),
      ),
    );
  }
}
```



#### AnimationStatusListener监视动画的过程

```dart
//Flutter动画的四种状态枚举
enum AnimationStatus {
    /// 动画在开始时停止（初始状态）
    dismissed,
    /// 动画从头到尾运行
    forward,
    /// 倒过来运行（从尾到头运行）
    reverse,
    /// 动画在结尾处停止
    completed,
}

//示例：使用addStatusListener()在开始或结束时反转动画，产生循环执行的效果
class _LogoAppState extends State<LogoApp> with SingleTickerProviderStateMixin {
  AnimationController controller;
  Animation<double> animation;

  initState() {
    super.initState();
    controller = new AnimationController(duration: const Duration(milliseconds: 2000), vsync: this);
    animation = new Tween(begin: 0.0, end: 300.0).animate(controller);
	//设置动画监听器
    animation.addStatusListener((status) {  //status是动画当前状态
      //★ 通过监听动画状态使其往复执行
      if (status == AnimationStatus.completed) {
        controller.reverse();  //倒过来运行
      } else if (status == AnimationStatus.dismissed) {
        controller.forward();  //从头运行
      }
    });
    controller.forward();  //从头运行
  }
  //...
}
```



### 并行动画

需要同时并行执行多个动画时，为不同的动画创建自己的Tween估值器，共用同一个Animation对象。如下示例中，改变widget大小的同时控制透明度的动画：

```dart
//核心代码
final AnimationController controller =
    new AnimationController(duration: const Duration(milliseconds: 2000), vsync: this);
final Animation<double> sizeAnimation =
    new Tween(begin: 0.0, end: 300.0).animate(controller);
final Animation<double> opacityAnimation =
    new Tween(begin: 0.1, end: 1.0).animate(controller);

//示例
class AnimSet extends StatefulWidget {
  _LogoAppState createState() => new _LogoAppState();
}

class _LogoAppState extends State<AnimSet> with TickerProviderStateMixin {
  late AnimationController controller;
  late Animation<double> animation;
  initState() {
    super.initState();
    //1. 创建往复执行的动画，这个动画会被多个估值器使用
    controller = new AnimationController(duration: const Duration(milliseconds: 2000), vsync: this);
    animation = new CurvedAnimation(parent: controller, curve: Curves.easeIn);
    //往复执行
    animation.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        controller.reverse();
      } else if (status == AnimationStatus.dismissed) {
        controller.forward();
      }
    });
    controller.forward();  //开始
  }

  Widget build(BuildContext context) {
    return new AnimatedLogo(animation: animation);
  }

  dispose() {
    controller.dispose();
    super.dispose();
  }
}
class AnimatedLogo extends AnimatedWidget {
  //2. 多个动画估值器
  static final _opacityTween = new Tween<double>(begin: 0.1, end: 1.0);
  static final _sizeTween = new Tween<double>(begin: 0.0, end: 300.0);

  AnimatedLogo({required Animation<double> animation})
      : super(listenable: animation);

  Widget build(BuildContext context) {
    final Animation<double> animation = listenable as Animation<double>;
    return new Center(
      child: new Opacity(
        //通过各个估值器计算动画当前的值
        opacity: _opacityTween.evaluate(animation),  //3. 控制child透明度的动画
        child: new Container(
          margin: new EdgeInsets.symmetric(vertical: 10.0),
          height: _sizeTween.evaluate(animation),  //4. 控制child大小的动画
          width: _sizeTween.evaluate(animation),
          child: new FlutterLogo(),
        ),
      ),
    );
  }
}
```





xml创建动画、`view.startAnimation()`

- Flutter：

**如何使用Canvas draw/paint**

- Android：
- Flutter：

**如何构建自定义 Widgets**

- Android：
- Flutter：

手势检测和触摸事件处理
如何将一个onClick监听器添加到Flutter中的widget
如何处理widget上的其他手势

# 2. Layouts

LinearLayout在Flutter中相当于什么
RelativeLayout在Flutter中等价于什么
ScrollView在Flutter中等价于什么

Listview & Adapter
ListView在Flutter中相当于什么
怎么知道哪个列表项被点击
如何动态更新ListView
使用 Text
如何在 Text widget上设置自定义字体
如何在Text上定义样式
表单输入
Input的”hint”在flutter中相当于什么
如何显示验证错误



Activities 和 Fragments
Activity和Fragment 在Flutter中等价于什么
如何监听Android Activity生命周期事件

Intents
Intent在Flutter中等价于什么？

```dart
//路由跳转1
Navigator.of(context).push(new MaterialPageRoute(builder: (BuildContext context){
                return new Scaffold(
                  appBar: new AppBar(
                    title: new Text("Hello"),
                  ),
                  body: new HelloFlutter(),
                );
              }));

//路由跳转2，arguments可以填写需要传入的参数
Navigator.of(context).pushNamed("/hello_flutter", arguments: {});
//新页面Widget的build方法中获取参数
var arguments = ModalRoute.of(context).settings.arguments;//获取传入的值


              
```



var arguments = ModalRoute.of(context).settings.arguments;//获取传入的值



如何在Flutter中处理来自外部应用程序传入的Intents
startActivityForResult 在Flutter中等价于什么
异步UI
runOnUiThread 在Flutter中等价于什么
AsyncTask和IntentService在Flutter中等价于什么
OkHttp在Flutter中等价于什么
如何在Flutter中显示进度指示器
项目结构和资源
在哪里存储分辨率相关的图片文件? HDPI/XXHDPI
在哪里存储字符串? 如何存储不同的语言
Android Gradle vs Flutter pubspec.yaml

Flutter 插件
如何使用 GPS sensor
如何访问相机
如何使用Facebook登陆
如何构建自定义集成Native功能
如何在我的Flutter应用程序中使用NDK
主题
如何构建Material主题风格的app
数据库和本地存储
如何在Flutter中访问Shared Preferences ?
如何在Flutter中访问SQLite
通知
如何设置推送通知

