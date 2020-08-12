每当我们讨论流媒体，RTMP(Real Time Messaging Protocol)是不可或缺的。RTMP是一个基本的视频/音频直播流协议，但是不幸的是Android标准的VideoView不支持RTMP的播放。因此，如果想在android上播放RTMP直播流，你必须使用支持RTMP协议的库。  库播放由 RTMP 协议传输的流媒体。

[VLC开源的跨平台多媒体播放器](https://www.videolan.org/vlc/)

Android支持播放网络上的视频。在播放网络上的视频时，牵涉到视频流的传输，往往有两种协议，一种是HTTP，一种是RTSP。这两种协议最大的不同是，HTTP协议，不支持实时流媒体的播放，而RTSP协议就支持。Android中自带的播放器，以及VideoView等都支持上述两种协议，因此，可以直接播放网络上的视频，唯一不同的就是URI。


/** 
 * 本实例演示如何在Android中播放网络上的视频，这里牵涉到视频传输协议，视频编解码等知识点 
 * @author Administrator 
 *Android当前支持两种协议来传输视频流一种是Http协议，另一种是RTSP协议 
 *Http协议最常用于视频下载等，但是目前还不支持边传输边播放的实时流媒体 
 *同时，在使用Http协议 传输视频时，需要根据不同的网络方式来选择合适的编码方式， 
 *比如对于GPRS网络，其带宽只有20kbps,我们需要使视频流的传输速度在此范围内。 
 *比如，对于GPRS来说，如果多媒体的编码速度是400kbps，那么对于一秒钟的视频来说，就需要20秒的时间。这显然是无法忍受的 
 *Http下载时，在设备上进行缓存，只有当缓存到一定程度时，才能开始播放。 
 * 
 *所以，在不需要实时播放的场合，我们可以使用Http协议 
 * 
 *RTSP：Real Time Streaming Protocal，实时流媒体传输控制协议。 
 *使用RTSP时，流媒体的格式需要是RTP。 
 *RTSP和RTP是结合使用的，RTP单独在Android中式无法使用的。 
 * 
 *RTSP和RTP就是为实时流媒体设计的，支持边传输边播放。 
 * 
 *同样的对于不同的网络类型（GPRS，3G等），RTSP的编码速度也相差很大。根据实际情况来 
 * 
 *使用前面介绍的三种方式，都可以播放网络上的视频，唯一不同的就是URI 
 * 
 *本例中使用VideoView来播放网络上的视频 
 */  

# ijkplayer

ijkplayer是一个基于FFmpeg的轻量级Android/iOS视频播放器。FFmpeg的是全球领先的多媒体框架，能够解码，编码， 转码，复用，解复用，流，过滤器和播放大部分的视频格式。它提供了录制、转换以及流化音视频的完整解决方案。它包含了非常先进的音频/视频编解码库libavcodec，为了保证高可移植性和编解码质量，libavcodec里很多code都是从头开发的。

android版本最低minSdkVersion 21

[ijkplayer](https://github.com/Bilibili/ijkplayer)

[ijkplayer详解使用教程](https://www.jianshu.com/p/32e2045df7ed)

[使用Ijkplayer播放直播视频](https://www.cnblogs.com/renhui/p/6420140.html)

[在ubuntu下编译ijkplayer-android](https://blog.csdn.net/u010072711/article/details/51438871)

[ijkplayer接入使用](https://www.jianshu.com/p/a57bbdd78798)

查询真实IP:https://www.ipaddress.com/查询raw.githubusercontent.com的真实IP。

[](https://blog.csdn.net/weixin_34306593/article/details/88676946


```xml
//required, enough for most devices. android版本最低minSdkVersion 21
implementation 'tv.danmaku.ijk.media:ijkplayer-java:0.8.8'
implementation 'tv.danmaku.ijk.media:ijkplayer-armv7a:0.8.8'
//Other ABIs: optional
implementation 'tv.danmaku.ijk.media:ijkplayer-armv5:0.8.8'
implementation 'tv.danmaku.ijk.media:ijkplayer-arm64:0.8.8'
implementation 'tv.danmaku.ijk.media:ijkplayer-x86:0.8.8'
implementation 'tv.danmaku.ijk.media:ijkplayer-x86_64:0.8.8'
//ExoPlayer as IMediaPlayer: optional, experimental
implementation 'tv.danmaku.ijk.media:ijkplayer-exo:0.8.8'
```





# Vitamio

到官网或者github下载vitamio资源
官网地址:https://www.vitamio.org/ (最新版本5.0.0,但是官网很难打开…)
github地址:https://github.com/yixia/VitamioBundle (版本4.2.2)



















