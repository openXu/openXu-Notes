
# Android N 7.0 FileProvider

[Android N 7.0(API24) 行为变更](https://developer.android.google.cn/about/versions/nougat/android-7.0-changes)

对于面向Android 7.0的应用，Android框架执行的StrictMode(严格模式，用于检测不安全的代码) API政策禁止在应用外部公开`file://` URI。如果一项包含文件URI的intent离开您的应用，则会抛出`FileUriExposedException`异常。要在应用间共享文件，应发送一项`content://` URI，并授予URI临时访问权限。进行此授权的最简单方式是使用`FileProvider`类。

## 7.0之前file://

比如在7.0之前，打开系统相机拍照时，我们需要为Intent传递一个Uri作为拍照存储的临时文件，使用的是`Uri.fromFile(file)`的方式生成`file://`格式的Uri，下面代码在Android7.0之前的系统中运行是没有问题的(注：需要在设置中处理6.0存储权限)。

```java
private String mCurrentPhotoPath;
public void takePhotoNoCompress(View view) {
    Intent captureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    if (captureIntent.resolveActivity(getPackageManager()) != null) {
        String filename = new SimpleDateFormat("yyyyMMdd-HHmmss", Locale.CHINA)
                .format(new Date()) + ".png";
        File file = new File(Environment.getExternalStorageDirectory()+File.separator+"Images", filename);
        mCurrentPhotoPath = file.getAbsolutePath();
        //file:///storage/emulated/0/Images/20201203-175000.png
		Uri uri = Uri.fromFile(file);
        captureIntent.putExtra(MediaStore.EXTRA_OUTPUT, uri);
        startActivityForResult(captureIntent, 1);
    }
}

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if (resultCode == RESULT_OK && requestCode == 1) {
        Bitmap bitmap = BitmapFactory.decodeFile(mCurrentPhotoPath);
    }
}
```

但是运行在7.0以上系统时(需要设置targetSdkVersion >=24)就会抛出`FileUriExposedException`。应该将`file://`格式的Uri换成`content://`格式，并对其授权

## 7.0之后content://

**指定FileProvider**

由于`FileProvider`继承自`ContentProvider`，所以需要在清单文件中注册。`android:authorities`属性指定由该FileProvider生成的URI的授权唯一标识，它是全局唯一的，通常使用包名作为前缀。`android:grantUriPermissions`属性(授权文件的临时访问权限)必须为true，`android:exported`属性(表示该FileProvider是否需要公开出去)必须为false，否则会抛异常(请看`FileProvider.attachInfo()`方法) 。`meta-data`用于指定共享目录配置文件。

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:grantUriPermissions="true"    此值必须为true
    android:exported="false">             此值必须为false
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

**指定可共享的目录**

在xml文件夹中创建`file_paths.xml`，用于指定应用可共享的目录。`FileProvider`只能为预先指定的目录中的文件生成内容URI，`<paths>`元素必须包含以下子元素中的一种或多种：

- `<root-path/>` 代表设备的根目录new File("/");
- `<files-path/>` ：代表files/应用程序内部存储区域的子目录Context.getFilesDir()
- `<cache-path/>` ：代表应用程序内部存储区域的cache子目录Context.getCacheDir()
- `<external-path/>`：代表外部存储区根目录Environment.getExternalStorageDirectory()
- `<external-files-path>`：代表应用程序外部存储区根目录Context.getExternalFilesDirs()
- `<external-cache-path>`：代表应用程序外部缓存区域根目录Context.getExternalCacheDirs()
- `<external-media-path>`：代表应用程序外部媒体区域根目录Context.getExternalMediaDirs()(仅在API 21+设备上可用)

这些子元素都使用下面两个属性:

- `name`：URI路径段。**为了加强安全性，此值将隐藏您要共享的子目录的名称**。该值的子目录名称包含在 path属性中
- `path`：共享的实际子目录名称，如果该属性值为空（会报警告Value must not be empty），则表示共享子元素对应目录下的所有

下面的配置中表示共享外部存储区根目录下的Images子文件夹下的所有内容，也就是说路径中包含`Environment.getExternalStorageDirectory()/Images`的File都可以被共享，可通过`FileProvider`生成File对应的Uri。

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-path name="external" path="Images" />
</paths>
```

**FileProvider.getUriForFile()**

`FileProvider`提供了`getUriForFile()`方法根据文件获取Uri，该方法第二个参数为`authority`，也就是在清单文件中配置的provider的`android:authorities`属性值，FileProvider根据第二个参数就能找到对应的`provider`，然后根据该`provider`中配置的共享文件目录生成Uri。该方法返回的Uri格式为`content://authorities/xml中定义的name属性/文件的相对路径`

```java
public void takePhotoNoCompress() {
    Intent captureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    if (captureIntent.resolveActivity(getPackageManager()) != null) {
        String filename = new SimpleDateFormat("yyyyMMdd-HHmmss", Locale.CHINA)
                .format(new Date()) + ".png";
        File file = new File(Environment.getExternalStorageDirectory()+File.separator+"Images", filename);
        mCurrentPhotoPath = file.getAbsolutePath();
        // Uri uri = Uri.fromFile(file);
        // content://com.hk.custom.fileprovider/external/20201203-175000.png
        Uri uri = FileProvider.getUriForFile(this, getPackageName()+".fileprovider", file);
        captureIntent.putExtra(MediaStore.EXTRA_OUTPUT, uri);
        startActivityForResult(captureIntent, 1);
    }
}
```

## 兼容低版本

### 手动授权的方式

使用`FileProvider.getUriForFile()`获取Uri的方式在低于API24的系统中运行会抛异常`java.lang.SecurityException: Permission Denial`

> 针对拍照功能，intent的action为ACTION_IMAGE_CAPTURE，当调用startActivity后会调用到`Instrumentation的execStartActivity()`，然后调用`intent.migrateExtraStreamToClipData()`方，该方法中针对action为ACTION_IMAGE_CAPTURE的意图通过addFlags的方式添加读写权限，而这部分内容是API21之后添加的，所以针对拍照功能演示低版本的权限问题需要运行在低于API21的系统中。

因为低版本系统中将`FileProvider`当成了一个普通的`ContentProvider`，所以并不具备`FileProvider`的共享文件授权功能（清单文件中配置的`android:grantUriPermissions="true"`是无效的），所以我们需要手动授权。`Context`提供了下面两个方法：

- `grantUriPermission(String toPackage, Uri uri,
int modeFlags)` 授权
- `revokeUriPermission(Uri uri, int modeFlags)` 撤销授权

```java
private String mCurrentPhotoPath;
public void takePhotoNoCompress() {
    Intent captureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    if (captureIntent.resolveActivity(getPackageManager()) != null) {
        String filename = new SimpleDateFormat("yyyyMMdd-HHmmss", Locale.CHINA)
                .format(new Date()) + ".png";
        File file = new File(Environment.getExternalStorageDirectory()+File.separator+"Images", filename);
        mCurrentPhotoPath = file.getAbsolutePath();
        Uri uri = FileProvider.getUriForFile(this, getPackageName()+".fileprovider", file);
        captureIntent.putExtra(MediaStore.EXTRA_OUTPUT, uri);
        //为了兼容低版本（android24以下），手动为uri授权
        //获取远程意图对应的Activity信息集合，因为Intent可能打开多个应用
        List<ResolveInfo> resInfoList = getPackageManager()
                .queryIntentActivities(captureIntent, PackageManager.MATCH_DEFAULT_ONLY);
        //因为不知道用户会选择打开哪个应用，这里需要对所有应用授权
        for (ResolveInfo resolveInfo : resInfoList) {
            //获取远程包名
            String packageName = resolveInfo.activityInfo.packageName;
            //为当前uri对包名为packageName的应用授予读写权限
            grantUriPermission(packageName, uri, Intent.FLAG_GRANT_READ_URI_PERMISSION
                    | Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
        }
        startActivityForResult(captureIntent, 1);
    }
}
```

我们还可以调用`Intent.addFlags()`方法授权，这种方式相比上面更加简洁:

```java
captureIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
captureIntent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
```

### 版本判断的方式

低于Android24系统中使用`content://`需要手动授权，但是使用`file://`就不需要上面的授权步骤，所以为了兼容低版本，我们还可以通过版本判断的方式来实现：

```java
Uri uri;
if (Build.VERSION.SDK_INT >= 24) {
    uri = FileProvider.getUriForFile(this, getPackageName()+".fileprovider", file);
} else {
    uri = Uri.fromFile(file);
}
```

## 通用适配方案

清单文件中配置`provider`时`android:authorities`使用包名`${applicationId}`做前缀：

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:grantUriPermissions="true"   
    android:exported="false">            
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

`file_paths.xml`中配置共享所有根文件目录：

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <root-path name="root" path="" />
    <files-path name="files" path="" />
    <cache-path name="cache" path="" />
    <external-path name="external" path="" />
    <external-files-path name="external_file_path" path="" />
    <external-cache-path name="external_cache_path" path="" />
</paths>
```

根据版本获取Uri，笔者发现有些文章中在这里还需要手动授权，其实是没必要的，因为已经根据版本判断了，只有在API24及以上系统中才使用`FileProvider`，而清单文件中配置`provider`时已经设置`android:grantUriPermissions="true"`了。对于应用安装的案例中也不需要手动授权，至少本人测试过程中是这样的。

```java
Uri uri;
if (Build.VERSION.SDK_INT >= 24) {
    uri = FileProvider.getUriForFile(this, getPackageName()+".fileprovider", file);
} else {
    uri = Uri.fromFile(file);
}
```



# Android10分区存储

[Android 11中的存储更新](https://developer.android.google.cn/about/versions/11/privacy/storage)

Android 10(API29)提出了分区存储的概念，当`targetSdkVersion`设置为29及以上后，我们无法向之前一样通过File操作外部存储，当然为了兼容低版本，可在清单文件中配置`android:requestLegacyExternalStorage="true"`，表示请求使用旧的外部存储策略而不是分区存储。

但是到了Android 11(API30)，当`targetSdkVersion`设置为30后，系统将忽略`requestLegacyExternalStorage`标志，所以我们不得不对分区存储做适配了。




https://www.jianshu.com/p/37feb5116191

“”

            +"~"+
            FTimeUtils.dateStrTranc(data.EndTime, "yyyy/MM/dd", "yyyy.MM.dd"))}'






FileUtils
FpcFiles



2020-12-28 10:00:16.344 17401-17401/com.hk.operator I/FZY: (ParseBigDataFunction.java:49) 返回数据：{"status":200,"data":{"monthlyGeneralStat":{"month":"202012","startDate":"2020-12-05","endDate":"2020-12-28","estimateDate":"4","elecQuantities":3.0645152E7,"elecQuantitiesSRatio":null,"elecQuantitiesCRatio":1.2,"quantityCost":9365066.0,"quantityCostSRatio":null,"quantityCostCRatio":1.2,"powerCoefficient":0.9015231,"evaluation":"合格","powerCoefficientSRatio":null,"powerCoefficientCRatio":-0.01,"carbonEmission":4.5218756E7,"carbonEmissionSRation":null,"carbonEmissionCRation":1.24},"quantityCategory":{"spireQuantity":0.0,"peakQuantity":2909464.0,"flatQuantity":1.1715392E7,"valleyQuantity":1.6020296E7},"costCategory":{"quantityCost":9365066.0,"fundamentalCost":41000.0,"adjustedCost":0.0,"attachedCost":1532257.6},"alarmStats":[{"levelName":"一般预警","alarmNum":819},{"levelName":"紧急预警","alarmNum":93},{"levelName":"严重预警","alarmNum":1}]},"message":"操作成功！"}
2020-12-28 10:00:16.350 17401-17401/com.hk.operator E/FZY: (FragmentElectric.java:270) 用户电系统统计：ElectricStatisticGeneral{carbonEmission=4.5218756E7, carbonEmissionCRation=1.24, carbonEmissionSRation=null, elecQuantities=3.0645152E7, elecQuantitiesCRatio=1.2, elecQuantitiesSRatio=null, endDate='2020-12-28', estimateDate='4', month='202012', powerCoefficient=0.9015231, powerCoefficientCRatio=-0.01, powerCoefficientSRatio=null, quantityCost=9365066.0, quantityCostCRatio=1.2, quantityCostSRatio=null, startDate='2020-12-05'}

