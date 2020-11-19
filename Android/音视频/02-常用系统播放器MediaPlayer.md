https://developer.android.google.cn/guide/topics/media?hl=zh_cn

## MediaPlayer生命周期

![](02-01-MediaPlayer生命周期.png)

上图描述了MediaPlayer的各个状态，也列举了主要的方法的调用时序，每种方法只能在一些特定的状态下使用，如果使用时MediaPlayer的状态不正确则会引发IllegalStateException异常。

- Idle 状态：当使用new()方法创建一个MediaPlayer对象或者调用了其reset()方法时，该MediaPlayer对象处于idle状态。这两种方法的一个重要差别就是：如果在这个状态下调用了getDuration()等方法（相当于调用时机不正确），通过reset()方法进入idle状态的话会触发OnErrorListener.onError()，并且MediaPlayer会进入Error状态；如果是新创建的MediaPlayer对象，则并不会触发onError(),也不会进入Error状态。

- End 状态：通过release()方法可以进入End状态，只要MediaPlayer对象不再被使用，就应当尽快将其通过release()方法释放掉，以释放相关的软硬件组件资源，这其中有些资源是只有一份的（相当于临界资源）。如果MediaPlayer对象进入了End状态，则不会在进入任何其他状态了。

- Initialized 状态：这个状态比较简单，MediaPlayer调用setDataSource()方法就进入Initialized状态，表示此时要播放的文件已经设置好了。

- Prepared 状态：初始化完成之后还需要通过调用prepare()或prepareAsync()方法，这两个方法一个是同步的一个是异步的，只有进入Prepared状态，才表明MediaPlayer到目前为止都没有错误，可以进行文件播放。

- Preparing 状态：这个状态比较好理解，主要是和prepareAsync()配合，如果异步准备完成，会触发OnPreparedListener.onPrepared()，进而进入Prepared状态。

- Started 状态：显然，MediaPlayer一旦准备好，就可以调用start()方法，这样MediaPlayer就处于Started状态，这表明MediaPlayer正在播放文件过程中。可以使用isPlaying()测试MediaPlayer是否处于了Started状态。如果播放完毕，而又设置了循环播放，则MediaPlayer仍然会处于Started状态，类似的，如果在该状态下MediaPlayer调用了seekTo()或者start()方法均可以让MediaPlayer停留在Started状态。

- Paused 状态：Started状态下MediaPlayer调用pause()方法可以暂停MediaPlayer，从而进入Paused状态，MediaPlayer暂停后再次调用start()则可以继续MediaPlayer的播放，转到Started状态，暂停状态时可以调用seekTo()方法，这是不会改变状态的。

- Stop 状态：Started或者Paused状态下均可调用stop()停止MediaPlayer，而处于Stop状态的MediaPlayer要想重新播放，需要通过prepareAsync()和prepare()回到先前的Prepared状态重新开始才可以。

- PlaybackCompleted状态：文件正常播放完毕，而又没有设置循环播放的话就进入该状态，并会触发OnCompletionListener的onCompletion()方法。此时可以调用start()方法重新从头播放文件，也可以stop()停止MediaPlayer，或者也可以seekTo()来重新定位播放位置。

- Error状态：如果由于某种原因MediaPlayer出现了错误，会触发OnErrorListener.onError()事件，此时MediaPlayer即进入Error状态，及时捕捉并妥善处理这些错误是很重要的，可以帮助我们及时释放相关的软硬件资源，也可以改善用户体验。通过setOnErrorListener(android.media.MediaPlayer.OnErrorListener)可以设置该监听器。如果MediaPlayer进入了Error状态，可以通过调用reset()来恢复，使得MediaPlayer重新返回到Idle状态。

## 基本使用和相关类

```java
mSurfaceHolder = binding.surfaceview.getHolder();
mSurfaceHolder.addCallback(new SurfaceHolder.Callback() {
    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        mMediaPlayer = new MediaPlayer();
        //给mMediaPlayer添加预览的SurfaceHolder
        mMediaPlayer.setDisplay(holder);
        String uri = "android.resource://" + getContext().getPackageName() + "/" + R.raw.test;
        //设置数据源
        mMediaPlayer.setDataSource(getContext(), Uri.parse(uri));
        mMediaPlayer.setVideoScalingMode(MediaPlayer.VIDEO_SCALING_MODE_SCALE_TO_FIT);//缩放模式
        mMediaPlayer.setLooping(true);//设置循环播放
        mMediaPlayer.prepareAsync();//异步准备
//            mMediaPlayer.prepare();//同步准备
        mMediaPlayer.setOnPreparedListener(new MediaPlayer.OnPreparedListener() { //准备完成回调
            @Override
            public void onPrepared(MediaPlayer mp) {
                mp.start();  //开始播放
            }
        });
    }
    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
    }
    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
    }
});
```

**SurfaceTexture**

Android 3.0(API 11)加入的一个类，它用于从视频解码流中捕获帧作为OpenGL ES纹理。跟
SurfaceView很像，但不同的是，SurfaceTexture在捕获帧之后，不需要显示 出来。因此我们可以用SurfaceTexture接收解码出来的图像流，然后取得图像帧的副本进行处理，处理完毕后再送给另一个SurfaceView用于显示。

**Surface**

处理被屏幕排序的原生的缓冲区，Android中的Surface就是一个用来画图形或者图像的地方，对于View及其子类都是画在Surface上的。在Surface中创建Canvas对象，可用来管理Surface绘图操作，Canvas对应Bitmap，存储Surface中的内容

**SurfaceView**

在Camera、MediaRecorder、MediaPlayer中SurfaceView经常被用来显示图像。 SurfaceView是View的子类，实现了Parcelable接口，其中内嵌了一个专门用于绘制的Surface，SurfaceView可以控制这个Surface的格式和尺寸，以及Surface的绘制位置。可以理解Surface就是管理数据的，SurfaceView是展示数据的。

**SurfaceHolder**

SurfaceHolder是一个接口，可被理解为Surface的监昕器。通过回调函数addCallback(SurfaceHolder.Callback callback)监听Surface的创建，通过获取Surface中的Canvas对象，锁定之。所得到的Canvas对象在完成修改Surface中的数据后，释放同步锁，并提交改变Surface的状态及图像，展示新的图像数据


## 源码解析

### 创建

```java
//创建对象
MediaPlayer mMediaPlayer = new MediaPlayer();
//创建对象并做了setDataSource、prepare动作，只需等待调用start()就能播放了
mMediaPlayer = MediaPlayer.create(getContext(), Uri)
```
MediaPlayer创建可以new，也可以调用create()方法，最终都要调用构造方法创建对象

```java
 public MediaPlayer() {
    super(new AudioAttributes.Builder().build(),
            AudioPlaybackConfiguration.PLAYER_TYPE_JAM_MEDIAPLAYER);
    //定义一个Looper，并创建一个EventHandler的Handler
    Looper looper;
    if ((looper = Looper.myLooper()) != null) {
        mEventHandler = new EventHandler(this, looper);
    } else if ((looper = Looper.getMainLooper()) != null) {
        mEventHandler = new EventHandler(this, looper);
    } else {
        mEventHandler = null;
    }
    mTimeProvider = new TimeProvider(this);
    mOpenSubtitleSources = new Vector<InputStream>();

    /* Native setup requires a weak reference to our object.
     * It's easier to create it here than in C++.
     */
    native_setup(new WeakReference<MediaPlayer>(this));

    baseRegisterPlayer();
}
```




















