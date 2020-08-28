大图片显示，先解析图片的宽高，按照控件大小等比缩放后再加载

多图显示，使用LruCache (此类在android-support-v4的包中提供) 。这个类非常适合用来缓存图片，它的主要算法原理是把最近使用的对象用强引用存储在 LinkedHashMap 中，并且把最近最少使用的对象在缓存值达到预设定值之前从内存中移除。


# 使用
https://github.com/bumptech/glide
```xml

dependencies {
    compile 'com.github.bumptech.glide:glide:3.7.0'
}
<uses-permission android:name="android.permission.INTERNET" />

http://cn.bing.com/az/hprichbg/rb/Dongdaemun_ZH-CN10736487148_1920x1080.jpg
Glide.with(this).load(url).into(imageView);
```

首先，调用Glide.with(context)方法用于创建一个加载图片的实例。context决定Glide加载图片的生命周期，如果传入的是Activity或者Fragment的实例，那么当这个Activity或Fragment被销毁的时候，图片加载也会停止。如果传入的是ApplicationContext，那么只有当应用程序被杀掉的时候，图片加载才会停止。

load()方法，这个方法用于指定待加载的图片资源。Glide支持加载各种各样的图片资源，包括网络图片、本地图片、应用资源、二进制流、Uri对象等等。因此load()方法也有很多个方法重载

```Java
// 加载本地图片
File file = new File(getExternalCacheDir() + "/image.jpg");
Glide.with(this).load(file).into(imageView);

// 加载应用资源
int resource = R.drawable.image;
Glide.with(this).load(resource).into(imageView);

// 加载二进制流
byte[] image = getImageBytes();
Glide.with(this).load(image).into(imageView);

// 加载Uri对象
Uri imageUri = getImageUri();
Glide.with(this).load(imageUri).into(imageView);
```

into()方法，这个方法就很简单了，我们希望让图片显示在哪个ImageView上，把这个ImageView的实例传进去就可以了

## 常用功能

```java
Glide.with(this)
     .load(url)
     .asBitmap()            //必须是一张静态图片，判断它到底是静图还是GIF图
     .asGif()               //必须加载动态图片
     .placeholder(R.drawable.loading)    //占位图
     .error(R.drawable.error)            //异常占位图
     .diskCacheStrategy(DiskCacheStrategy.NONE)   //禁用掉Glide的缓存功能
     .override(100, 100)      //指定一个固定的大小。绝大多数情况下我们都是不需要指定图片大小的，因为Glide会自动根据ImageView的大小来决定图片的大小。
     .into(imageView); 
```


# 源码解读


## with()

```Java
 public static RequestManager with(Activity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }
```


with()方法是Glide类中的一组静态方法，它有好几个方法重载。它获取一个`RequestManager`对象。RequestManagerRetriever类中看似有很多个get()方法的重载，什么Context参数，Activity参数，Fragment参数等等，实际上只有两种情况而已，即传入Application类型的参数，和传入非Application类型的参数。

传入Application参数的情况。其实这是最简单的一种情况，因为Application对象的生命周期即应用程序的生命周期，因此Glide并不需要做什么特殊的处理，它自动就是和应用程序的生命周期是同步的，如果应用程序关闭的话，Glide的加载也会同时终止。

传入非Application参数的情况。会向当前的Activity当中添加一个隐藏的Fragment，因为Glide需要知道加载的生命周期。很简单的一个道理，如果你在某个Activity上正在加载着一张图片，结果图片还没加载出来，Activity就被用户关掉了，那么图片还应该继续加载吗？当然不应该。可是Glide并没有办法知道Activity的生命周期，于是Glide就使用了添加隐藏Fragment的这种小技巧，因为Fragment的生命周期和Activity是同步的，如果Activity被销毁了，Fragment是可以监听到的，这样Glide就可以捕获这个事件并停止图片加载了。

## load()

```Java

  public DrawableTypeRequest<String> load(String string) {
        return (DrawableTypeRequest<String>) fromString().load(string);
    }
```

RequestManager中也有很多个load()方法的重载，用于适支持图片URL字符串、图片本地路径等等加载形式。返回一个DrawableTypeRequest对象，它提供了asBitmap()和asGif()这两个方法

## into()

下载图片，展示


# Glide缓存

Glide的缓存设计可以说是非常先进的，考虑的场景也很周全。在缓存这一功能上，Glide又将它分成了两个模块，一个是内存缓存，一个是硬盘缓存。

这两个缓存模块的作用各不相同，内存缓存的主要作用是防止应用重复将图片数据读取到内存当中，而硬盘缓存的主要作用是防止应用重复从网络或其他地方重复下载和读取数据。

内存缓存和硬盘缓存的相互结合才构成了Glide极佳的图片缓存效果






























