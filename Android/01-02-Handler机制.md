

对于一个Android程序员来说，Handler可谓是被谈及最多的一个词也不为过。从Android开发刚兴起的时候它就是面试时不可避讳的话题，发展到现在，它的重要地位也未能被替代，很多开源框架的实现原理也都离不开它（比如EventBus），虽然现在面试被问的比较少了，这可能是因为它被认为是Android的一个最基础的知识点了，如果连它都没搞懂，您好意思说是一个合格的Android攻城狮吗？不过话说回来，不少老程序员对于它虽然很熟悉，但是真要问起来还说不死，这篇文章我们就一起将它说死。废话不多说，作为一个老程序员，基本的东西还是知道的，就不用说怎么使用那些废话了，直接进入正题。

Handler机制主要由Handler、Looper、MessageQueue、Message四个类来实现的。


# Looper

Handler最常用的使用方式是在主线程中创建一个Handler对象，子线程中耗时获取数据后发送消息给Handler，Handler收到消息后在主线程刷新页面。如果我们要在子线程中创建Handler则需要多两个步骤，否则会报异常：

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

## prepare()

```Java
#### android.os.Looper
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
//将当前线程初始化为循环程序。请确保在调用此方法后调用loop()，并在循环结束时调用quit()
public static void prepare() {
    prepare(true);
}
private static void prepare(boolean quitAllowed) {
	//从sThreadLocal中获取一个Looper对象，如果获取到则抛出一个异常，说明一个线程只能调用一次Looper.prepare()
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    //如果是第一次调用prepare()，为当前线程创建一个Looper对象，放入sThreadLocal中
    sThreadLocal.set(new Looper(quitAllowed));
}
final MessageQueue mQueue;
final Thread mThread;
private Looper(boolean quitAllowed) {
	//Looper构造方法中初始化了MessageQueue消息队列对象
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}

```

从源码可以看出，`prepare()`方法其实就是为当前线程初始化了一个`Looper`对象并放在`sThreadLocal`中保存，而`Looper`的构造方法里初始化了类型为`MessageQueue`的`mQueue`变量，并持有当前线程的引用。`sThreadLocal`是`ThreadLocal`类的变量，它的作用是将`Looper`对象与线程关联起来，并保证当前线程有且仅有一个`Looper`对象，至于为什么要用`ThreadLocal`放到最后讲一下。

## loop()

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
    //for死循环，不断的从队列中去除Message消息对象
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        //...
    	//Message的target就是当前消息保存的Handler对象的引用，通过handler.dispatchMessage(msg)将消息分发给Handler
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
        //回收可能正在使用的消息。MAX_POOL_SIZE = 50
        msg.recycleUnchecked();
    }
}
```

Looper的prepare()方法为当前线程创建一个唯一的Looper对象，Looper对象中维护了一个消息队列，loop()方法则开启一个for死循环（子线程）不断取出队列中的消息，然后调用handler的dispatchMessage(msg)方法将消息发送给目标Handler处理。到此Handler消息机制最核心的部分就完了，接下来我们看看怎样往队列中发送消息以及Handler是怎样收到消息的。


- Looper通过ThreadLocal与线程绑定，确保每个线程有且仅有一个Looper对象（多次调用prepare()方法抛异常）
- Looper对象的作用是维护了一个MessageQueue消息队列，并调用loop()方法for循环从队列中取出消息

# Handler

```Java
//处理消息
Handler handler = new Handler(){
    @Override
    public void handleMessage(@NonNull Message msg) {
        super.handleMessage(msg);
        switch (msg.what){
            case 1:
                Log.w("handler", "主线程创建Handler收到消息："+msg.obj);
                break;
        }
    }
};

//发送消息
Message msg = handler.obtainMessage();
msg.what = 1;
msg.obj = "这是子线程发送的消息";
handler.sendMessage(msg);

```



# MessageQueue


# Message






```Java
@UnsupportedAppUsage
public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```


final Looper mLooper;
    final MessageQueue mQueue;






## ThreadLocal
https://www.jianshu.com/p/95291228aff7
https://www.cnblogs.com/takumicx/p/9320881.html

ThreadLocal的作用是提供线程内的局部变量，跟其他线程的变量没有关系（所以说起ThreadLocal提到多线程并发就把天聊死了），这种变量在线程的生命周期内起作用，减少同一个线程内多个函数或者组件之间一些公共变量的传递的复杂度。ThreadLocal实例通常来说都是private static类型的，用于关联线程和线程的上下文。


### 同步机制synchronized和ThreadLocal的区别

同步机制是用来解决多线程并发资源共享的问题，通常情况下是多个线程共同操作同一个变量，为了保证数据的正确性，让线程排队处理。

ThreadLocal则是为每一个线程都提供一个属于自己的变量，线程间互不影响的使用自己的变量，但是线程内部可以随时随地的访问该变量。



在讲解源码之前，先看一下总结，这样心里有个谱，看源码也就不迷茫了。ThreadLocal的作用是为每个线程Thread对象分配一个类型为ThreadLocal.ThreadLocalMap的map容器，每个Thread对象都有自己的ThreadLocalMap容器，它存储的值只被当前线程读取和修改。而ThreadLocal相当于一个工具，可通过它提供的set()、get()、remove()方法往map中设置、获取、删除对象，这样就隐藏了ThreadLocalMap。




```Java
#### java.lang.ThreadLocal
public T get() {
	//1.1 获取当前调用者线程对象
    Thread t = Thread.currentThread();
    //1.2 获取当前线程Thread对象的成员变量threadLocals，其类型为ThreadLocal.ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        //1.3 
        if (e != null) {
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
	//Thread线程中有一个threadLocals变量ThreadLocal.ThreadLocalMap threadLocals = null;
    return t.threadLocals;
}
```














