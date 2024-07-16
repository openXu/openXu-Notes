

## flutter商显版问题总结

### 一. 待解决问题

#### 1. iOS文件路径问题

ios升级后应用文件目录会发生变化，升级之后需要手动替换一些文件（自己生成的）中存储的绝对路径：

```xml
# 如节目json文件中存储的视频文件地址
/Users/yqp/Library/Developer/CoreSimulator/Devices/43E3C8C6-DD12-4DB7-98A8-002020E2E1AB/data/Containers/Data/Application/（D89F8810-A7F5-46D1-8124-194D2F749A0C）（可变部分）/Library/Application Support/program/res/a98cd9cc456679e1363a9cd0ee53d094.mov

```

涉及到的内容（在文件中存储的绝对路径）：

- 节目json文件中存储的图片、视频文件路径
- 本地播放器`tempPlayers.xml`数据中存储的播放器文件路径

**解决办法**

- 如上，每次升级后手动修改路径 **（难维护）**
- 使用相对路径 **（flutter商显版改动较大，改动后需大量测试）**



#### 2. 子线程

flutter中主线程处理耗时任务会导致UI卡顿，目前商显版大部分任务都在主线程，包括节目转换、数据传输（http请求插件会自动切换到子线程）。与原生不同的是，flutter的卡顿是页面无法响应、绘制，但不会崩溃，这也是导致这个问题后来才暴露出来的原因。

子线程目前存在几个问题：

- 调用插件方法需要在主线程（可能是flutter子线程与原生主线程数据传输问题，待验证），**这块对调用原生绘制的影响需要验证一下是否会造成卡顿**
- Canvas绘制需要在主线程，节目打包不能简单的将任务扔到子线程，需要细分步骤（图片绘制与存储、视频转码、json生成），视频转码可以放到子线程，图片绘制（表格）这个耗时部分无法解决，还是会造成卡顿。目前解决办法是去掉了动态旋转的UI部分，避免视觉上的卡顿
- 主线程和子线程数据传递只能使用可序列化的类型（基本数据类型、String、List、Map），其他自定义类型需要转换成json字符串



**子线程这部分内容还需要查阅更多实战用例，寻找解决办法**



#### 3. 是否是国内用户

flutter中无法获取到当前时区（只能获取时间偏差，同经度偏差相同），目前通过系统选择语言来进行判断，结果不够准确。原生的好像也有问题：

- Android获取的是`Asia/Shanghai`时区
- ios的`INTERNATIONAL_VERSION`永远是`true`，获取时区`Asia/chongqing`、`Asia/Harbin`、`Asia/Shanghai` **&&** 语言`zh_CN`

```java
// Android原生逻辑
public static boolean isChinaUser() {
    boolean chinaUser;
    //1. 打包标识
    if (AppBuildConfig.INTERNATIONAL_VERSION) {  
        // 2. 是否是中国时区
        chinaUser = getCurrentTimeZone().equals("Asia/Shanghai");
    } else {
        // 国内版任何情况都需要登录
        chinaUser = true;
    }
    return chinaUser;
}
```

**解决办法**

- 写插件调用原生
- 使用定位



#### 4. 甩一甩图片、视频加载问题

甩一甩加载系统图片、视频资源使用[photo_gallery](https://pub-web.flutter-io.cn/packages/photo_gallery)插件，如果进入页面时获取所有的图片和视频会很慢，所以使用分页加载提升速度，分页加载在某些Android手机上报错（插件sqlite查询语法错误，请看插件问题总结部分）已解决。但存在一个问题，点击预览某一个图片或者视频时，左右滑动只能切换目前已经加载的资源。

**解决办法**

- 进入时加载所有的资源，转圈让用户等待
- 维持现状



#### 5. 视频卡顿发热问题

换了四个视频播放插件，都存在一些问题：

- [video_player: ^2.8.3](https://pub-web.flutter-io.cn/packages/video_player) flutter 官方播放插件[多路播放OOM](https://github.com/flutter/flutter/issues?q=is%3Aissue+is%3Aopen+label%3A%22p%3A+video_player%22+OutOfMemory+)
- [media_kit: ^1.1.10](https://pub.dev/packages/media_kit) lib库下不下来，一样存在发热
- [fijkplayer: ^0.11.0](https://github.com/befovy/fijkplayer) b站开源的音视频播放器（基于ffmpeg），播放多个100M视频，应用变得非常卡顿
- [flutter_vlc_player: ^7.4.1](https://pub-web.flutter-io.cn/packages/flutter_vlc_player) 当前选择的播放插件，播放多路视频存在不流畅，发热。发热与原生测试对比差别不大，但个人感觉还是有问题



#### 6. 简单密码校验问题



### 二. 已解决的问题

#### 1. 三方打开本地播放器（已解决）

```json
{"fileName":"MagicPlayer_1.9.71.0(rk).apk.1.1.1", "filePath":"/storage/emulated/0/Android/data/cn.huidu.hdlcdapp/cache/MagicPlayer_1.9.71.0(rk).apk.1.1.1", "supportedDeviceModels":"M10,M20,M21,M30,3288S,3566S,3568S,3399F", "versionName":"1.9.71.0"}
```

Apk文件中的`AndroidManifest.xml`是经过编译后的二进制文件，特有的数据排列格式。flutter中首先解压apk，通过流逐字节读取解析，反编译出清单文件xml字符串，然后使用xml解析。存在的问题：

- 代码量较多，需仔细检查提高代码健壮性，目前直接try住了
- ios分享文件到flutter无法读取（权限问题），已解决



#### 2. [json_serializable](https://pub.dev/packages/json_serializable) 序列化模板问题

`AreaNode`的child泛型为`PluginNode`，所有插件都是`PluginNode`的子类，自动生成的模板会将json中所有的插件都反序列化为`PluginNode`类型。所以需要修改自动生成的`area_node.g.dart`的代码。

但是如果新增了一个需要生成json模板的类需要执行`dart run build_runner build`时，`area_node.g.dart`会被删除后重新生成，如果直接注释`@JsonSerializable`，`area_node.g.dart`就会被删除。

**解决办法**

将修改好的`area_node.g.dart`更名为`area_node.j.dart`，并注释`@JsonSerializable`。以后需要特殊处理的都按照这个方式

```dart
part 'area_node.j.dart';
// @JsonSerializable(explicitToJson: true)
class AreaNode extends Node<PluginNode> {}

/// area_node.j.dart
part of 'area_node.dart';
AreaNode _$AreaNodeFromJson(Map<String, dynamic> json) => AreaNode()
  ..children = (json['children'] as List<dynamic>)
      /// 自定义插件反序列化
      .map((e) => AreaNode.pluginNodeFromJson(e['tag'] as String, e as Map<String, dynamic>))
      .toList()
    ...

改动：
.map((e) => PluginNode.fromJson(e as Map<String, dynamic>))
替换为：
    .map((e) => AreaNode.pluginNodeFromJson(e['tag'] as String, e as Map<String, dynamic>))
```



#### 3. PlayTaskConverter.dart 按钮打包不生效

节目打包生成json时，会将对象中所有的字段（值为null或者""）打包进json，原生java会忽略null的字段。协议中有些字段是互斥的，两个同时设置会导致某一个被忽略。在flutter中需要手动判断这些互斥的字段，并且这些字段需要通过`Map`设置。

```dart

Future<AreaPacket> convertArea(AreaNode area) async {
    AreaPacket jArea = AreaPacket();
    jArea.name = area.name;
    jArea.uuid = area.uuid;
    jArea.action = ACTION_ADD;
    jArea.frame = RectPacket(area.x, area.y, area.width, area.height);
    jArea.motion = area.motion;
    jArea.carousel = area.carousel;
    jArea.widgets = [];

    if (area.type == AreaNode.buttonArea) {
	  ...
      jArea.asButton?.onClick = InteractionPacket();
      jArea.asButton?.onClick.changeItems = [];
      /// ☆☆☆
      /// 按钮插件控制数据包（flutter中使用Map代替  原因如下），
      ///
      /// 请参考《LCD设备节目数据协议.docx》 3.6 互动行为
      /// Interaction包含两个字段
      ///       changeProgram	String	可选	设置此属性表示互动行为是切换节目，此属性值为节目切换位置（见切换位置描述）
      ///       changeItems	List<ChangeItem>	可选	设置此属性表示互动行为是切换区域内容，此属性值为内容切换列表
      /// 1. changeProgram表示按钮要控制的节目的uuid，可选，但是不能传""或者null，否则找不到该节目，☆这里统一不传递
      /// 2. ChangeItem更新后的URL（仅支持网页和流媒体，设置此属性会忽略position属性
      /// 而 发送节目打包成json调用jsonEncode(msg.toJson())会将null或者“”打进json
      ///
      // ChangeItemPacket changeItem = ChangeItemPacket();
      // changeItem.area = area.areaUuid;
      Map<String, String> changeItem = {
        "area":area.areaUuid
      };
      if (area.position.isNotEmpty) {
        // changeItem.position = area.position;
        changeItem["position"] = area.position;
      }
      if (area.url.isNotEmpty) {
        //请参考《LCD设备节目数据协议.docx》 3.6 互动行为
        // 更新后的URL（仅支持网页和流媒体，设置此属性会忽略position属性
        // changeItem.url = area.url;
        changeItem["url"] = area.url;
      }
      jArea.asButton?.onClick.changeItems.add(changeItem);

    } else {
      jArea.background = convertColor(area.backgroundColor);
    }
    ...
    return jArea;
  }

```



### 三. 插件问题

#### 1. [excel: ^4.0.3](https://pub-web.flutter-io.cn/packages/excel)解析excel插件问题（修改插件源码解决）

##### 问题1：报错

> 使用wps或者其他软件（非Google Sheets）生成或编辑的excel文件解析会报错[Unhandled Exception: Exception: custom numFmtId starts at 164 but found a value of 41 · Issue #296 · justkawal/excel · GitHub](https://github.com/justkawal/excel/issues/296)

**解决：**

> 修改`External Libraries->Dart Packages->xcel-4.0.3->parse.dart`（具体路径`C:\Users\Admin\AppData\Local\Pub\Cache\hosted\pub.flutter-io.cn\excel-4.0.3\lib\src\parser\parse.dart`）

```dart
void _parseStyles(String _stylesTarget) {
...
if (numFmtId < 164) {
// 注释此处
// throw Exception(
//     'custom numFmtId starts at 164 but found a value of $numFmtId');
}
...
}
```



##### 问题2：读取不到合并信息（合并信息为空）

> [Bug reading SpannedItems as empty · Issue #341 · justkawal/excel · GitHub](https://github.com/justkawal/excel/issues/341)

**解决：**

> 已给作者反馈问题，官方修改了错误，等待发版，可以先将下面两个类的代码贴过来[parse.dart](https://github.com/justkawal/excel/blob/main/lib/src/parser/parse.dart)、[sheet.dart](https://github.com/justkawal/excel/blob/main/lib/src/sheet/sheet.dart)。注意：先copy然后再解决问题1



#### 2. [image_picker: ^1.0.7](https://pub.dev/packages/image_picker)资源(图片、视频)选择插件问题（修改插件源码解决）

##### 问题1：ios选择视频会自动压缩

**解决：**

> 位置：Flutter Plugins -> image_picker_ios-0.8.11
>
> - **FLTImagePickerPlugin_Test.h ** 增加字段
>
>   `@property(nonatomic, assign) BOOL onlyVideo;`
>
> - **FLTImagePickerPlugin.m ** 修改方法

```objective-c
// 1. 替换整个方法
- (void)pickVideoWithSource:(nonnull FLTSourceSpecification *)source
                maxDuration:(nullable NSNumber *)maxDurationSeconds
                 completion:
                     (nonnull void (^)(NSString *_Nullable, FlutterError *_Nullable))completion {

    [self cancelInProgressCall];
    FLTImagePickerMethodCallContext *context = [[FLTImagePickerMethodCallContext alloc]
        initWithResult:^void(NSArray<NSString *> *paths, FlutterError *error) {
          if (paths.count > 1) {
            completion(nil, [FlutterError errorWithCode:@"invalid_result"
                                                message:@"Incorrect number of return paths provided"
                                                details:nil]);
          }
          completion(paths.firstObject, error);
        }];
    context.onlyVideo = true;
    context.maxImageCount = 1;

    if (@available(iOS 14, *)) {
      [self launchPHPickerWithContext:context];
    } else {
      // Camera is ignored for gallery mode, so the value here is arbitrary.
      [self launchUIImagePickerWithSource:[FLTSourceSpecification makeWithType:FLTSourceTypeGallery
                                                                        camera:FLTSourceCameraRear]
                                  context:context];
    }
}

// 2. 修改部分内容
- (void)launchPHPickerWithContext:(nonnull FLTImagePickerMethodCallContext *)context
    API_AVAILABLE(ios(14)) {
   //将下面的部分修改
    //if (context.includeVideo) {
    //  config.filter = [PHPickerFilter anyFilterMatchingSubfilters:@[
    //    [PHPickerFilter imagesFilter], [PHPickerFilter videosFilter]
    //  ]];
    //} else {
    //  config.filter = [PHPickerFilter imagesFilter];
    //}
    //改为
  if (context.onlyVideo) {
    config.filter =  [PHPickerFilter videosFilter];
  } else if (context.includeVideo) {
    config.filter = [PHPickerFilter anyFilterMatchingSubfilters:@[
      [PHPickerFilter imagesFilter], [PHPickerFilter videosFilter]
    ]];
  } else {
    config.filter = [PHPickerFilter imagesFilter];
  }
...
}

// 3. 修改部分内容
- (void)launchUIImagePickerWithSource:(nonnull FLTSourceSpecification *)source
                              context:(nonnull FLTImagePickerMethodCallContext *)context {
  UIImagePickerController *imagePickerController = [self createImagePickerController];
  imagePickerController.modalPresentationStyle = UIModalPresentationCurrentContext;
  imagePickerController.delegate = self;
  // 将下面部分
  //if (context.includeVideo) {
  //  imagePickerController.mediaTypes = @[ (NSString *)kUTTypeImage, (NSString *)kUTTypeMovie ];

  //} else {
  //  imagePickerController.mediaTypes = @[ (NSString *)kUTTypeImage ];
  //}
  // 改为                            
 if (context.onlyVideo) {
    imagePickerController.mediaTypes = @[(NSString *)kUTTypeMovie ];
  } if (context.includeVideo) {
    imagePickerController.mediaTypes = @[ (NSString *)kUTTypeImage, (NSString *)kUTTypeMovie ];

  } else {
    imagePickerController.mediaTypes = @[ (NSString *)kUTTypeImage ];
  }
}                         
```



##### 问题2：android手机选择某些视频文件，后缀被改为.mp4

android获取视频后缀失败，一些文件获取不到后缀名被统一改成mp4，这会导致后面一系列动作失败（缩略图、分辨率、播放）

**解决：**

> 位置：Flutter Plugins -> image_picker_android-0.8.12

```java
// C:\Users\Admin\AppData\Local\Pub\Cache\hosted\pub.flutter-io.cn\image_picker_android-0.8.12\android\src\main\java\io\flutter\plugins\imagepicker\FileUtils.java 
String getPathFromUri(final Context context, final Uri uri) {
 try (InputStream inputStream = context.getContentResolver().openInputStream(uri)) {
     ...
   String fileName = getImageName(context, uri);
     //注释掉下面的部分
//      String extension = getImageExtension(context, uri);
//      Log.w("FileUtils", "获取文件的名称和扩展格式 fileName = " + fileName +"    extension="+extension);

/*      if (fileName == null) {
     Log.w("FileUtils", "Cannot get file name for " + uri);
     if (extension == null) extension = ".jpg";
     fileName = "image_picker" + extension;
   } else if (extension != null) {
     fileName = getBaseName(fileName) + extension;
   }*/
   File file = new File(targetDirectory, fileName);
   try (OutputStream outputStream = new FileOutputStream(file)) {
     copy(inputStream, outputStream);
     return file.getPath();
   }
 } catch (IOException e) {
   return null;
 } catch (SecurityException e) {
   return null;
 }
}
```



#### 3.  [umeng_common_sdk: ^1.2.3](https://developer.umeng.com/docs/119267/detail/174923)友盟统计release包报错（修改插件源码配置解决）

##### 问题1：release包报错

> SDK初始化失败，请检查是否集成umeng-asms-1.2.X.aar库
> 问题原因：打包混淆时没有添加umeng混淆文件，
> https://blog.csdn.net/qq_22007319/article/details/121997354

**解决：**

> 在External Libraries -> Flutter Plugins -> umeng_common_sdk-1.2.7 -> android
> 下添加**proguard-rules.pro**，并添加如下内容：

```xml
-keep class com.umeng.** {*;}
-keepclassmembers class * {
public <init> (org.json.JSONObject);
}
-keepclassmembers enum * {
 public static **[] values();
 public static ** valueOf(java.lang.String);
}
```

> 然后在`C:\Users\Admin\AppData\Local\Pub\Cache\hosted\pub.flutter-io.cn\umeng_common_sdk-1.2.7\android\build.gradle`中加入：

```groovy
android {
 compileSdkVersion 29

 defaultConfig {
     minSdkVersion 16
     // ★★★ 添加下面配置
     consumerProguardFiles 'proguard-rules.pro'
 }
 lintOptions {
     disable 'InvalidPackage'
 }
}
```





#### 4.  [photo_gallery: ^2.2.1](https://pub-web.flutter-io.cn/packages/photo_gallery)获取所有照片或者视频插件分页android原生报错（修改插件源码解决）

##### 问题1：android部分手机sqlite报错

> 该插件分页在android上不返回，跟踪排查是因为sql语句报错（红米note10 Android9.0）:
>
> ```sqlite
> 错误：android.database.sqlite.SQLiteException: near "OFFSET": syntax error (code 1 SQLITE_ERROR): , while compiling: SELECT _id, _data FROM images ORDER BY date_added DESC, date_modified DESC OFFSET 0 LIMIT 30
> 
> 需要改为（LIMIT在前，OFFSET在后）：
> 
> SELECT _id, _data FROM images ORDER BY date_added DESC, date_modified DESC LIMIT 30 OFFSET 0
> ```
>
> 有文章说是因为sqllite版本不支持（其实不是，android5.0以上的sqlite版本号3.7.11以上了，已经支持分页），下附android获取sqlite版本号源码：
>
> ```java
> // 获取sqllite版本号
> SQLiteDatabase sqLiteDatabase = android.database.sqlite.SQLiteDatabase.create(null);
> Cursor cursor1 = sqLiteDatabase.rawQuery("SELECT sqlite_version()", new String[]{});
> Log.w(TAG, "获取sqlite版本："+cursor1.getColumnCount());
> cursor1.moveToFirst();
> int sdfsd = cursor1.getColumnIndex(cursor1.getColumnName(0));
> //3.22.0
> Log.w(TAG, "获取sqlite版本："+cursor1.getString(sdfsd));
> ```

**解决：**

> 位置：Flutter Plugins -> photo_gallery-2.2.1 -> PhotoGalleryPlugin.kt
>

```kotlin
// PhotoGalleryPlugin.kt
// 改动1
private fun getImageCursor(
 albumId: String,
 newest: Boolean,
 projection: Array<String>,
 skip: Int?,
 take: Int?
): Cursor? {

 imageCursor = this.contentResolver.query(
     MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
     projection,
     selection,
     selectionArgs,
     // 修改此处
     // "$orderBy $offset $limit"
     // 改为
     "$orderBy $limit $offset"
 )
}

// 改动2
private fun getVideoCursor(
 albumId: String,
 newest: Boolean,
 projection: Array<String>,
 skip: Int?,
 take: Int?
): Cursor? {
 videoCursor = this.contentResolver.query(
     MediaStore.Video.Media.EXTERNAL_CONTENT_URI,
     projection,
     selection,
     selectionArgs,
     // 修改此处
     // "$orderBy offset $$limit"
     //改为
     "$orderBy $limit $offset"
 )
}
```



### 四.  总结

#### 1. 插件问题

- 插件兼容性问题

  > 兼容性问题需要充分测试，比如视频选择插件需要测试各种视频格式（避免修改为.mp4）、查询所有图片视频资源在某些android手机上sqlite分页报错

- 插件bug或者不满足要求

  > 这两个问题通过修改插件源码可以解决，但是维护相当麻烦，插件库是android studio构建时自动检查下载的，可能一些clean的操作会清除之前下载后已经改过源码的插件，或者换一个工程环境，再次构建会重新下载。这会导致之前已经修改过的问题莫名其妙重现。
  >
  > 这个问题在后期维护时相当严重，需要想办法解决（离线编辑？但是开发时又不方便），需要搞清楚插件缓存什么时候会被清除

- 有些插件问题没办法通过修改源码解决

  > 比如播放器插件



#### 2. android和ios差异性问题

比如ios应用升级会修改应用文件路径，这部分开始没有考虑到，很多逻辑都是参考的原生代码，现在比较被动



#### 3. 模板代码

需要总结模板代码，比如常规页面进入后，加载数据放到子线程（转圈让用户等待） 到 build渲染的流程



#### 4. 工程架构问题

单色版 是否和flutter共用同一个工程

**优点**

- 公用类库（工具类，基类）沉淀
- 同一个工程管理多个项目，避免多窗口切换



**缺点**

- 目前想到的除了工具类，没有其他太多可以复用的，如节目数据结构、预览、传输协议，这样一来上面的两个优点好像并不突出



























