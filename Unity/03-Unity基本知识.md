

### 1. 游戏开发基本流程

   - 开发之前，要先对市场进行调研
   - 对于原创游戏，为了规避风险，在投入力量开发之前，可以考虑先进入Prototype(原型)开发阶段，这是一个快速尝试各种想法的阶段，通过较少的投入，开发较少的内容，体现游戏的核心乐趣
   - 对初始的游戏Demo做评估，以确定游戏的乐趣是否达到了预期 ；如果不好玩，那就要考虑继续修改或停止项目
   - 为了有效地控制进度并监督质量，制定一个计划是很有必要的，同时要推出阶段性的版本

### 2. 游戏引擎

- 主要的游戏引擎：**Unity**、**Cocos2d**等
- Why Unity ？Develop once deploy anywhere 跨多平台（IOS、Android、Windows Phone、Windows、Flash、XBOX360、PS3、Wii等）游戏引擎 可以开发2D、3D游戏
- unity几乎支持所有高端的3D动画软件，如3ds Max、Maya、LightWave，将制作的模型动画导出为FBX格式供Unity使用
- **发展方向**：手机游戏开发；虚拟现实；工业控制
- 参考网址：
      > 官方网站：http://unity3d.com/  (qq邮箱、Aa191)
      > Unity3d联盟：http://www.u3dchina.com/
      > U3D圣典社区： http://game.ceeger.com/forum/
      > cg模型网： http://www.cgmodel.cn/

### 3. 坐标系 

&emsp;&emsp;坐标系：世界坐标系、 局部坐标系、 相机坐标系Viewport、 屏幕坐标系Screen 、GUI坐标系

&emsp;&emsp;Unity采用左手坐标系： X轴向右 Y轴向上 Z轴向里

![](https://github.com/openXu/Blog/blob/master/blog_unity/pic03/pic1.png)

**世界坐标系与本地坐标系***

- 世界坐标系（world）：所有物体的世界坐标系都是相同的、不会改变
- 本地坐标系（local）：每个游戏对象都可以以自己为原点建立一套坐标系，当物体旋转时，本地坐标系统也会跟着物体一起旋转
- 把Cube的Rotation改为(45,45,45)，这样世界(world)坐标和本地(local/self)坐标就不一样了

![](https://github.com/openXu/Blog/blob/master/blog_unity/pic03/pic2.png)

### 4. Hello World

**基本流程**
- File->New Project
- 在Hierarchy中Create一个Cube立方体，在Inspector中修改它的Position XYZ为0
- Project中Create一个C# Script，Update表示帧更新，在方法中编写监听输入、改变对象位置的代码
- 把脚本拖到Cube上（拖到Hierarchy中比较准确） 运行
- 发布：File->Build Settings，选择发布平台，默认是PC
- 项目的保存和再次加载：双击打开unity场景文件
- 注意：项目运行过程中的修改不会保存，调试结束一切调试期的修改都消失

**脚本说明**
- **Start、Update**是系统预定义的方法，表示启动、帧更新
- 类继承自**MonoBehaviour**
- 类的名称与文件名称要一致
- 一般放在*Scripts*文件夹中
- 设置脚本的组件菜单，使用[AddComponentMenu(“”)]特性，加到类的前面
- 类Time的静态变量deltaTime：表示距上一次调用Update或FixedUpdate所用的时间
- 帧：就像以前的电影胶卷一样，一帧就是一张胶片
- 更多类及成员说明，可以参见帮助(Help->Script Reference)



**PC发布设置说明**
- Edit->Project Settings->Player，在Inspector中对游戏进行设置
- 选项组Cross-Platform Settings:设置游戏的名称、公司名、图标，这个设置适用各平台
- 选项组Resolution：设置游戏窗口的大小
- Run In Background：在后台运行
- Display Resolution Dialog中选择Disabled，游戏在每次启动时不会显示出用于设置显示分辨率的窗口
- 选项组Icon：设置不同大小的图标，如果不指定则会自动调用Default Icon中设置的图标并自动缩放。自动缩放的图标可能会有锯齿。
- 选项组Splash Image：可以显示自定义的图片

### 5. 基础认识
**开发工具面板**
- Hierarchy：当前场景中的物体
- Project：项目中的所有资源
- Scene：当前场景的预览视图
- Inspector：属性
- Game：游戏视图，以摄像机视角查看场景，可以预览到玩家看到的内容

**鼠标的操作与使用**
- 左键，在菜单栏下方有一组工具
- Q（移动场景）
- W（位置变换）
- E（旋转变换）
- R（缩放变换）
- T（2维精灵的移动、旋转、缩放）
- 点击2D可以切换2维与3维的视角
- 右键：调整视角，坐标系变换
- 中键（滚轮）：移动画布

**常用内置3D游戏对象**
- Cube、 Sphere、 Capsule、 Cylinder、 Plane、 Quad

**基本构成元素**
- GameObject游戏对象
- Component组件
- Material渲染材质
- Texture渲染纹理


























