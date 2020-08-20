> 版权声明：本文为openXu原创文章[【openXu的博客】](http://blog.csdn.net/xmxkf)，未经博主允许不得以任何形式转载

@[TOC](目录)

对于一个Android程序员来说，Handler可谓是被谈及最多的一个词也不为过。从Android开发刚兴起的时候它就是面试时不可避讳的话题，发展到现在，它的重要地位也未能被替代，很多开源框架的实现原理也都离不开它，虽然现在面试被问的比较少了，这可能是因为它被认为是Android的一个最基础的知识点了，如果连它都没搞懂，您好意思说是一个Android老手吗？不过话说回来，不少老程序员对于它虽然很熟悉，但是真要问起细节还说不死，这篇文章我们就一起回顾一下这个经典知识。废话不多说，作为一个老程序员，基本的东西还是知道的，就不用说怎么使用那些废话了，直接进入正题。

Handler机制主要由Handler、Looper、MessageQueue、Message四个类来实现的，它们各司其职，首先看看Looper类：

# 1. Looper

Android中最常用的使用方式是在主线程中创建一个Handler对象，子线程中耗时获取数据后发送消息给Handler，Handler收到消息后在主线程刷新页面。但是如果我们要在子线程中创建Handler则需要多两个步骤，否则会报异常：

```Java
new Thread(() -> {
    //1：Looper.prepare()
    Looper.prepare();
    subHandler = new Handler(){
        @Override
        public void handleMessage(@NonNull Message msg) {
            super.handleMessage(msg);
            switch (msg.what){
                case 1:
                    Log.w("handler", "子线程创建Handler收到消息："+msg.obj);
                    break;
            }
        }
    };
    //2：Looper.loop()
    Looper.loop();
}).start();
```

为什么子线程中的Handler需要这两步而主线程中创建Handler却不需要？Looper是个什么东西？`prepare()`、`loop()`方法做了什么事情？

## 1.1 prepare()创建Looper

```Java
#### android.os.Looper
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
//将当前线程初始化为循环程序。请确保在调用此方法后调用loop()，并在循环结束时调用quit()
public static void prepare() {
    prepare(true);
}

//quitAllowed：True 消息队列可以退出（子线程创建的Handler可调用Looper.quit()退出消息队列轮询）
//             false 消息队列不能退出（主线程中创建的Handler不可以退出轮询，请看下面的prepareMainLooper()方法，传的就是false）
private static void prepare(boolean quitAllowed) {
	//从sThreadLocal中获取一个Looper对象，如果获取到则抛出一个异常，说明一个线程只能调用一次Looper.prepare()
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    //★ 如果是第一次调用prepare()，为当前线程创建一个Looper对象(请看下面Looper的构造方法)，放入sThreadLocal中
    sThreadLocal.set(new Looper(quitAllowed));
}
//应用程序的主循环器是由Android环境创建的，所以您不必自己调用这个函数。这个方法在下面将的ActivityThread的main方法中被调用
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}

final MessageQueue mQueue;
final Thread mThread;
private Looper(boolean quitAllowed) {
	//★ Looper构造方法中初始化了MessageQueue消息队列对象
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

从源码可以看出，`prepare()`方法其实就是为当前线程初始化了一个`Looper`对象（循环器）并放在`sThreadLocal`中保存，而`Looper`的构造方法里初始化了类型为`MessageQueue`(消息队列)的`mQueue`变量。`sThreadLocal`是`ThreadLocal`类的变量，它的作用是将`Looper`对象与线程关联起来，并保证当前线程有且仅有一个`Looper`对象，至于为什么要用`ThreadLocal`放到最后讲一下。在Looper中还有一个`prepareMainLooper()`方法，这个方法是系统创建应用程序是由系统调用的，其作用是为主线程创建循环器。

## 1.2 loop()轮询

```Java
 public static void loop() {
 	//获取当前线程的Looper对象=>sThreadLocal.get()
    final Looper me = myLooper();
    //如果没有获取到，抛异常，提示需要先调用Looper.prepare()
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    //获取当前线程Looper对象中的MessageQueue消息队列
    final MessageQueue queue = me.mQueue;
    //...
    //★ for死循环轮询取出消息
    for (;;) {
    	//★ 从队列中取出消息，next()是一个阻塞方法，当队列中没有需要马上处理的消息会阻塞线程，直到获取到一条消息
        Message msg = queue.next(); // might block
        if (msg == null) {
        	//如果没有next()的阻塞，队列中消息为空时，轮询就停止了
            return;
        }
        //...
    	//★ Message的target就是当前消息保存的Handler对象的引用，通过handler.dispatchMessage(msg)将消息分发给Handler
        try {
            msg.target.dispatchMessage(msg);
            if (observer != null) {
                observer.messageDispatched(token, msg);
            }
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } catch (Exception exception) {
            //...
        } finally {
            //...
        }
        //...
        //★ 回收可能正在使用的消息，供Message.obtain()复用
        msg.recycleUnchecked();
    }
}
```

Looper的prepare()方法为当前线程创建一个唯一的Looper对象，Looper对象中维护了一个消息队列，loop()方法则开启一个for循环不断轮询取出队列中的消息，然后调用handler的dispatchMessage(msg)方法将消息发送给目标Handler处理。需要注意的是轮询取消息的动作所在的线程是调用loop()方法的线程，如果实在主线程中创建Handler则轮询也是在主线程，如果在子线程中创建的Handler，轮询动作在子线程，子线程的Handler收到消息后如果要刷新UI，需要使用`runOnUiThread(new Runnable(){})`切换到主线程。总之记住一条，在哪个线程创建Handler，则在那个线程收到消息。

## 1.3 主线程的Looper

为什么子线程中的Handler需要这两步而主线程中创建Handler却不需要？这是因为应用程序在启动时(`ActivityThread`的main方法)已经为主线程创建了Looper循环器，并开启了消息轮询：

```Java
#### android.app.ActivityThread 隐藏API @hide

public static void main(String[] args) {
    //...
    //★ 为主线程创建一个消息队列不能退出的Looper
    Looper.prepareMainLooper();

    //...
    ActivityThread thread = new ActivityThread();
    //
    thread.attach(false, startSeq);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    //...
    //★ 开启主循环器的消息轮询
    Looper.loop();
}
```

## 1.4 主线程looper死循环为什么不会导致ANR

ActivityThread的main方法中为主线程创建了Looper，并调用loop()开启了死循环轮询消息，为什么主线程出现死循环没有导致ANR？Looper在没有消息需要处理时是休眠状态（后面讲解消息队列时会涉及到怎样休眠的），这时候主线程是可以响应用户交互的。Android是由事件驱动的，我们在屏幕上所有的点击触摸交互都是一个事件，这些事件都会被放入主线程的消息队列中，等待Looper轮询分发事件，这样才有了组件的声明周期、事件处理等回调，其实都是对loop到的消息的处理。也正是因为Looper的死循环，才没有导致主线程退出（线程run方法执行完就退出了）。什么情况下会导致ANR？我们在处理消息事件时（生命周期方法、handleMessage()）出现耗时操作，系统发现looper长时间不能正常轮询也不能阻塞休眠时就会发生ANR。

接下来我们看看怎样往消息队列中插入消息以及Handler是怎样收到消息的。

# 2. Handler

```Java
Handler handler = new Handler(){
    @Override
    public void handleMessage(@NonNull Message msg) {
        super.handleMessage(msg);
        //处理消息
        switch (msg.what){
            case 1:
                Log.w("handler", "主线程创建Handler收到消息："+msg.obj);
                break;
        }
    }
};
//★ 获取message实例的方式有很多，应该使用哪一种请看下面Message部分的讲解
//new Message对象（但是获取消息的首选方法是调用Message.obtain())
Message msg1 = new Message();
//从全局池返回新的消息实例，避免在许多情况下分配新对象。
Message msg2 = Message.obtain();
//obtainMessage()方法其实也是调用Message.obtain()
Message msg3 = handler.obtainMessage();
//对msg进行设置
msg3.what = 1;
msg3.obj = "这是一条来自Handler的消息";
//发送消息
handler.sendMessage(msg3);
//发送runnable
handler.post(new Runnable() {
        @Override
        public void run() {

        }
    });

```

上面的代码首先获取到一个Message对象，对其进行数据设置，然后`handler.sendMessage(msg3)`将消息发送出去，最后在Handler的`handleMessage()`中处理消息。这个过程最重要的两步就是发送消息以及消息是怎么被传到`handleMessage()`中的。

## 2.1 发送消息

```Java
#### android.os.Handler

//1. Handler构造方法
public Handler(Callback callback, boolean async) {
    //拿到调用new Handler()所属线程的Looper轮询器
    mLooper = Looper.myLooper();
    //如果没有获取到轮询器抛异常，提示需要调用Looper.prepare()初始化轮询器
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    //持有轮询器的消息队列，方便Handler王其中插入和分发消息
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}

public final boolean sendMessage(@NonNull Message msg) {
    return sendMessageDelayed(msg, 0);
}
public final boolean sendMessageDelayed(Message msg, long delayMillis){
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    //消息被处理的时间 = 当前时间+延迟的时间，SystemClock.uptimeMillis()是自开机启动到目前的毫秒数
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

//2. sendMessage(msg)、sendEmptyMessage(what)、sendMessageDelayed(msg, delayMillis)等常用发消息的方法最终都是调用sendMessageAtTime()
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
	//拿到Handler中的mQueue，其实就是Looper中维护的消息队列
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    //将消息排队到消息队列中
    return enqueueMessage(queue, msg, uptimeMillis);
}
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
	//msg的target指向当前Handler对象，方便在轮询器中取出消息后分发给当前handler
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    //★ 调用MessageQueue的enqueueMessage()方法将消息放到消息队列中，该方法跟踪请看MessageQueue讲解
    return queue.enqueueMessage(msg, uptimeMillis);
}

//3. 使用post()方法发送的Runnable对象将被包裹成一个Message消息，并将Runnable作为Message的callback回调
public final boolean post(@NonNull Runnable r) {
   return  sendMessageDelayed(getPostMessage(r), 0);
}
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}

```

Handler发送消息最终都是调用到`queue.enqueueMessage(msg, uptimeMillis)`，也就是调用MessageQueue的方法将消息插入到队列中，详情请看MessageQueue部分讲解


## 2.2 分发消息

上面讲`Looper.loop()`方法时我们知道轮询器开启了一个for循环调用`queue.next()`从消息队列中取出消息，然后调用`msg.target.dispatchMessage(msg)`分发消息，下面我们就看看Handler的`dispatchMessage(msg)`方法是怎样将消息分发的：

```Java
#### android.os.Handler
//承接Looper.loop()方法中调用的dispatchMessage(msg)
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
    	//post(runnable)发送的消息将调用handleCallback()
        handleCallback(msg);
    } else {
        if (mCallback != null) {
        	/* 
        	 * 创建handler时传递了Callback对象的情况，将调用mCallback.handleMessage(msg)
        	 * 这跟重写Handler的handleMessage(msg)方法差不多，但是它的返回值如果是false表示消息没处理，将继续由handleMessage(msg)处理
        	 * new Handler(new Handler.Callback() {
        			@Override
			        public boolean handleMessage(@NonNull Message msg) {
			            return false;
			        }
			    }
        	 */
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        //调用handleMessage(msg)，也就是我们重写的那个方法
        handleMessage(msg);
    }
}

//post(runnable)发送的消息将执行Runnable.run()
private static void handleCallback(Message message) {
    message.callback.run();
}
```

消息分发就是将loop()取出的消息传递给我们处理消息的方法，需要注意的是`dispatchMessage()`方法中并没有做线程切换，也就是说Handler在哪个线程创建的（Looper就属于这个线程，消息轮询也在该线程）则消息最终会被发送到这个线程中，如果是在子线程创建的Handler，收到消息后需要更新UI，需要使用`runOnUiThread(new Runnable(){})`切换到主线程。

到此消息的发送、分发、轮询都讲完了，接下来还有一个重要的，那就是消息队列，消息在队列中是怎么排列的？消息是怎样插到队列中的？轮询从队列中取出消息又是怎样一个过程？请看MessageQueue

# 3. MessageQueue

## ★ 3.1 数据结构

MessageQueue是一种单向链表数据结构，Message中有一个next属性指向队列中下一条消息，队列排序是根据Message的when字段（消息延迟时间）来排序的，调用`handler.sendMessage(msg)`发送的消息when==0，表示需要立刻处理的消息，调用`handler.sendMessageDelayed(msg, 1000)`发送的属于延迟消息，when==距开机时间毫秒数+延迟时间。总之when最小值为0，when越小表示消息延迟时间越小越被早处理，在队列中越靠前，下图模拟一下消息队列：

```Java
                      越靠近队头的消息越早被处理
						   -------------------------------------------------------
			          when   0      x+1000    x+1500    x+2000    x+3000      x+5000  (x表示开机时间，后面+值表示延迟时间，when值越大越靠后)
      
MessageQueue.next() 👈 队首msg0 ---> msg1 ---> msg2 ---> msg3 ---> msg4 ---> 队尾msg5                                                
从队头开始取消息                  next      next      next      next      next
						   -------------------------------------------------------
                                              👆
                           enqueueMessage(msg，when)根据时间插入消息到合适位置
```

## 3.2 插入消息到队列enqueueMessage()

```Java
#### android.os.MessageQueue

//承接Handler发送消息部分，往队列中插入消息
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    //1.★ 由于可以在任何线程中调用handler.sendMessage(msg3)发送消息，此处使用synchronized同步机制保证多线程并发时消息队列不会错乱
    synchronized (this) {
    	//如果调用了Looper.quit()退出后，消息不会被插入
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();  //标记消息正在被使用
        //when表示消息应该发送的时间，该时间=发送消息时距离手机启动毫秒数+消息延迟发送时间
        //参考Handler.sendMessage()，它的最小值为0，表示消息需要立刻处理，如果不为0表示是延时消息，该时间值越小，则消息越早被取出处理
        msg.when = when; 
        Message p = mMessages;   //队列头部的消息对象
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            //2.★ 判断（如果队列头部元素为null(空队列)、或者消息需要立刻处理、或者消息延迟时间小于队列头部消息延迟时间），则表示当前插入的消息应该最先被处理，应该作为队列的新头部
            msg.next = p;        //新队头消息的next指向老的队头消息
            mMessages = msg;     //mMessages指向新的头部消息
            needWake = mBlocked; //是否需要唤醒消息轮询
        } else {
  			//3.★ 如果此消息是延时消息，并且不能放到队头，则根据延迟时间将其插入到链表的合适位置
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;   //记录该消息应该插入位置的前一条消息
            //从队列头部开始查找第一条延迟时间大于被插入的消息
            for (;;) {
                prev = p;   
                p = p.next;
                if (p == null || when < p.when) {
                    break;   //如果找到了这条消息就停止
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            //将msg插入到队列中，前后next连接起来
            msg.next = p;    //p记录这msg的下一条消息，也就是队列中第一条延迟时间大于msg的消息
            prev.next = msg; //prev则是msg的前一条消息
        } 

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
        	//4.★ 如果轮询被阻塞，需要唤醒轮询
            nativeWake(mPtr);
        }
    }
    return true;
}
```

## 3.3 从队列轮询取出消息next()

`next()`方法用于从队列中取出队头的消息，它是一个阻塞方法，如果没有取到会阻塞线程，要不然`loop()`轮询就会停止。是否会阻塞分3中情况：

- 如果队头的消息到了发送时间，这时不阻塞，立刻返回该消息
- 如果队头消息没有到发送时间（延迟），应该阻塞直到该消息延迟时间到来
- 如果队列中没有消息，队头消息为null，此时需要一直阻塞，直到有新的消息插入时被唤醒

```Java
Message next() {
    //如果looper已经退出（调用了quit()），则返回null，不再处理消息
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        /*
         * nativePollOnce()是调用底层c++代码，实现阻塞
         * nextPollTimeoutMillis=0，不会阻塞，立即返回
         * nextPollTimeoutMillis=-1，一直阻塞，直到有新消息插入被唤醒
         * nextPollTimeoutMillis>0，阻塞nextPollTimeoutMillis毫秒，直到消息延迟时间完毕
         */
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;   //msg指向mMessages，也就是队列中队头的消息
            if (msg != null && msg.target == null) {
                //target为null，表示是一条屏障消息（postSyncBarrier()发送），这种情况不做讨论
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    //1.★ 队列头部的消息还未准备号(延迟消息还没到发送的时间)，设置阻塞时间=msg.when - now，阻塞时间到会自动唤醒继续
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                	//2.★ 队头的消息现在到了处理时间，应该马上被发送
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next; //让mMessages指向队头的下一条新的队头消息
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                //3.★ 如果队列中没有消息，则设置阻塞时间=-1，表示一直阻塞，直到有消息插入调用enqueueMessage()方法唤醒
                nextPollTimeoutMillis = -1;
            }
            if (mQuitting) {
                dispose();
                return null;
            }
            //...
        }
        //...
    }
}
```

我们分析了消息队列数据结构模型，以及Handler是怎样往消息队列中发送插入消息的，looper是怎样从队列中取出消息的，还有消息队列的轮询阻塞，这部分内容是非常重要的，整个Handler机制最核心的部分就是消息轮询与消息队列。下面我们继续分析一下`Message`，它里面包含消息对象的复用机制。

# 4. Message

```Java
#### android.os.Message

Handler target;    //消息发送目标Handler对象                
private static Message sPool;        //回收池中第一条可复用的消息对象
private static int sPoolSize = 0;    //回收池中消息数量
private static final int MAX_POOL_SIZE = 50;//最大回收数量

//Message中有一些列的obtain()重载方法，但它们都调用obtain()无参方法
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;//如果回收队列中有消息可复用，则取第一条消息
            sPool = m.next;   //sPool指向下一条可复用消息，供下次调用obtain()时返回
            m.next = null;    //将复用消息的next置空
            m.flags = 0;      //清除正在使用的标记
            sPoolSize--;      //消息回收池数量-1
            return m;
        }
    }
    //如果回收池中没有可复用消息，则new一个Message对象
    return new Message();
}

//调用该方法回收当前msg对象，如果消息正在被使用则会抛异常。一般情况下我们不需要手动回收消息，消息轮询发送后Looper.loop()方法最后会调用recycleUnchecked()实现消息回收
public void recycle() {
    if (isInUse()) {
        if (gCheckRecycle) {
            throw new IllegalStateException("This message cannot be recycled because it "
                    + "is still in use.");
        }
        return;
    }
    recycleUnchecked();
}

//回收该msg对象，该方法对我们是不可见的，如果我们需要手动回收需要调用上面的recycle()
void recycleUnchecked() {
    //清除msg对象的一些标记（重置）
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = UID_NONE;
    workSourceUid = UID_NONE;
    when = 0;
    target = null;
    callback = null;
    data = null;

    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
        	//如果回收池中消息数量<50，则回收
            next = sPool;
            sPool = this;  //将当前回收的msg对象放在回收池队首
            sPoolSize++;   //回收池消息数量+1
        }
    }
}
```

Message维护了一个静态的消息回收队列，这个队列比MessageQueue中的消息队列简单对了，它没有顺序限制，回收到消息后放到队首，obtain()复用消息也是在队首取，需要注意的是回收池最多能回收50个消息对象，超过的则会被垃圾回收。

思考一个问题，既然new一个Message对象也是ok的，为什么还需要回收？Android进程内通信主要就是靠Handler消息机制，发送消息是很频繁的，使用回收机制不但能节约内存，还能节省创建对象造成的时间开支。

到此为止整个Handler机制就over了，内容比较零散，**可以看最后的总结**。在讲`Looper`的时候，我们提到了`ThreadLocal`，留下一个问题。为什么要使用ThreadLocal？它有什么作用和优势？

------

# 5. ThreadLocal

https://www.jianshu.com/p/95291228aff7
https://www.cnblogs.com/takumicx/p/9320881.html

在讲解ThreadLocal之前，我们先思考一个问题，如果没有ThreadLocal，我们应该怎样维护Looper实例？在创建Handler之前，我们应该为其创建一个Looper对象，然后通过Handler的构造方法传给他，而且还要确保在同一个线程中创建的多个Handler传同一个Looper对象，当然通过这种传参的方式是可以实现。

但是任何API暴露给开发者时应该尽量做到极简，既然你要实现消息通信，那就只需要管消息发送和消息处理就行了，其他的不应该暴露出来。还有就是Looper应该和线程绑定，一个线程开启一个轮询，而不是和Handler绑定。ThreadLocal的作用正好是为线程提供局部变量，这种变量在线程的生命周期内起作用，减少同一个线程内多个函数或者组件之间一些公共变量的传递的复杂度。


## 5.1 ThreadLocal、synchronized的区别

synchronized同步机制是用来解决多线程并发资源共享的问题，通常情况下是多个线程共同操作同一个变量，为了保证数据的正确性，让线程排队处理。

ThreadLocal则是为每一个线程都提供一个属于自己的变量，线程间互不影响的使用自己的变量，但是线程内部可以随时随地的访问该变量。所以说起ThreadLocal提到多线程并发就把天聊死了。

## 5.2 ThreadLocal源码分析

```Java
		                                  set
									  |----------->  Thread1
									  |<-----------    map1<sThreadLocal, Looper>
									  |   getEntry
			                          |
   sThreadLocal-----------------------|		  
				get,set时会根据        |
				Thread.currentThread()|      
				找到对应Thread         |
       						          |    set
									  |----------->  Thread2
									  |<-----------    map2<sThreadLocal, Looper>
									  |   getEntry
									  |

```

ThreadLocal的为每个Thread对象分配一个类型为ThreadLocal.ThreadLocalMap的map容器，这个map容器其实不是真正的Map，它只是模拟Map键值对的方式对变量进行存储，其中key是ThreadLocal对象，value是ThreadLocal为线程设置的变量。而在map中key和value被组织在Entry这个弱引用中，当ThreadLocal对象没有了强引用时（Looper退出了）将会被GC回收。ThreadLocal相当于是一个容器外壳，不用我们手动维护Thread的容器副本，它提供了set()、get()、remove()方法往当前线程的map中设置、获取、删除变量，但是真正存储变量的是Thread中的map容器。

下面看一看核心源码，请仔细阅读注释：

```Java
#### android.os.Looper
public final class Looper {   
	//ThreadLocal实例通常来说都是private static类型的，其作用就是将线程和变量关联起来
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    private static void prepare(boolean quitAllowed) {
		//从sThreadLocal中获取一个Looper对象
	    if (sThreadLocal.get() != null) {
	        throw new RuntimeException("Only one Looper may be created per thread");
	    }
	    //第一次调用prepare()，为当前线程创建一个Looper对象放入sThreadLocal中
	    sThreadLocal.set(new Looper(quitAllowed));
	}
}


#### java.lang.ThreadLocal
//1. 为当前线程设置局部变量
public void set(T value) {
    Thread t = Thread.currentThread();
    //1.1 其实真正存储变量的是Thread中的threadLocals容器，它是ThreadLocalMap类型的
    ThreadLocalMap map = getMap(t);
    if (map != null)
    	//1.2 如果map不为空，存储键值对<当前ThreadLocal对象为键，设置的变量为值>
        map.set(this, value);  
    else
    	//1.3 如果map为空，则先创建map，然后存储键值对
        createMap(t, value);
}

//2. 获取当前线程以ThreadLocal对象为key的value值，如果没有调用set方法，将返回null
public T get() {
	//2.1 获取当前调用者线程对象
    Thread t = Thread.currentThread();
    //2.2 获取当前Thread对象的成员变量threadLocals
    ThreadLocalMap map = getMap(t);
    if (map != null) {
    	//2.3 以当前ThreadLocal对象为键获取map中存储的WeakReference
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            T result = (T)e.value;
            return result;
        }
    }
    //2.4 如果map容器为空，初始化map
    return setInitialValue();
}
//获取给定线程的threadLocals，其类型为ThreadLocalMap
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
private T setInitialValue() {
    T value = initialValue();   //return null， value为null
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
    	//为线程创建ThreadLocalMap=>t.threadLocals = new ThreadLocalMap(this, firstValue);
        createMap(t, value); 
    return value;
}

#### java.lang.ThreadLocal.ThreadLocalMap (ThreadLocal中的静态内部类)
	//内部类ThreadLocalMap，Thread中有一个ThreadLocalMap变量threadLocals，它才是真正存储的容器
	static class ThreadLocalMap {

		//Entry数组，用于存储ThreadLocal为线程设置的局部变量，Entry是弱引用，被它引用的ThreadLocal如果没有强引用指向时会被GC回收，而ThreadLocal在Looper类中是静态强引用，也就是说当Looper被回收时，Entry数组中引用的ThreadLocal也会被回收
	    private Entry[] table;
		private int size = 0;
		/*
		 * Entry继承自WeakReference(弱引用)，当一个对象仅被弱引用指向而没有任何强引用(Strong refrence)指向时，如果GC运行，这个对象就会被回收，不论当前内存空间是否足够
		 * Entry是弱引用，ThreadLocal<?>是被弱引用的对象
		 */
	    static class Entry extends WeakReference<ThreadLocal<?>> {
	        //与此ThreadLocal关联的值
	        Object value;
	        Entry(ThreadLocal<?> k, Object v) {
	            super(k);
	            value = v;
	        }
	    }
	    //获取指定ThreadLocal为key的Looper对象
		private Entry getEntry(ThreadLocal<?> key) {
	        int i = key.threadLocalHashCode & (table.length - 1);
	        Entry e = table[i];
	        if (e != null && e.get() == key)
	            return e;
	        else
	            return getEntryAfterMiss(key, i, e);
	    }
	    //设置变量，key为ThreadLocal，value为Looper对象
	    private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                if (k == key) {
                    e.value = value;
                    return;
                }
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
        //移除以指定ThreadLocal为key的变量（Looper）
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
}
```

# 6. 结论

- `ThreadLocal`可以为线程提供局部变量，整个线程声明周期中任何位置都能获取使用该变量，避免线程中变量传参问题
- `Looper`通过`ThreadLocal`与线程绑定，确保每个线程有且仅有一个`Looper`对象（多次调用`prepare()`方法抛异常）
- `Looper`就是一个轮询器，它维护了一个`MessageQueue`消息队列，在调用`loop()`后会开启一个for死循环从队列中不断轮询取出消息发送给`target` Handler处理
- `Looper.loop()`不会导致主线程ANR，Android应用整个生命周期都是运行在Handler机制之上的
- 一个线程可以创建多个`Handler`，它们共用同一个`Looper`轮询器和`MessageQueue`消息队列
- `MessageQueue`的入队与出队使用`synchronized`同步机制保证多线程并发消息队列不会错乱
- `MessageQueue.next()`方法是一个线程阻塞方法，当队列中没有需要立即处理的消息时会阻塞，直到有队列中有需要处理的消息
- Handler在哪个线程被创建，`handleMessage()`则在哪个线程处理消息，如果是子线程需要刷新UI，应该`runOnUiThread()`切换到主线程
- 发送消息获取Message实例时，应该使用`Message.obtain()`复用消息



