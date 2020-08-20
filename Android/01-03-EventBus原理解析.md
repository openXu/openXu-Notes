

[EventBus](https://github.com/greenrobot/EventBus)是一个轻量级的发布/订阅事件总线框架，解耦事件发送者和接收者，简化组件之间的通信。什么事件总线、什么消息机制其实都是为了实现组件之间（两个类）的通信问题，事件就是一个对象，消息也是一个对象，说白了就是一个类想要发送一个数据对象给另一个类。

为了实现组件间的通信，我们可以使用古老的接口方式，一个类实现接口的方法，另一个类持有接口引用并调用方法实现数据传递，这种方式在MVP模式中被大量使用，缺点就是需要定义大量的接口使得项目非常臃肿。另一种就是使用Handler机制，但是事件发布者必须要持有Handler的引用，这又带来了局限性。还有一种就是使用观察者模式，事件接收者作为观察者订阅某个主题，事件发布者调用主题发布事件，它发出通知时并不需要直到谁订阅了它，只要是订阅了该主题的观察者都能收到事件。EventBus就是基于观察者模式实现的，这篇文章通过翻阅源码看看它是怎么工作的。

# 1. EventBus使用

```xml
//添加依赖项
implementation 'org.greenrobot:eventbus:3.2.0'
```

```xml
//定义事件类
public static class MessageEvent {}
```

```Java
//声明订阅方法，订阅MessageEvent事件。如果订阅粘性事件，注解属性加上sticky=true
@Subscribe(threadMode = ThreadMode.MAIN)  
public void onMessageEvent(MessageEvent event) {};
```

```Java
//在生命周期方法上注册和注销订阅者
@Override
public void onStart() {
	super.onStart();
	EventBus.getDefault().register(this);
}
@Override
public void onStop() {
	super.onStop();
	EventBus.getDefault().unregister(this);
}
```

```Java
//发布事件
EventBus.getDefault().post(new MessageEvent());
//发布粘性事件
EventBus.getDefault().postSticky(new MessageEvent());
```

EventBus的使用非常简单，这里之所以还贴出来，因为我们翻阅源码的时候要知道从哪里开头到哪里结束。我们需要搞清楚的无非就是下面几个问题：

- 谁是订阅者，谁是主题
- 注册和注销订阅者做了哪些事情，内部是怎么管理主题和订阅者的
- 事件事件是怎么发布出去的
- 订阅者是怎么收到事件的

# 2. @Subscribe

Subscribe注解用于标记事件订阅者的方法，也就是事件应该通知回调的方法，在EventBus3.0之前，是使用固定方法名来标记订阅方法的，3.0之后注解的方式更加优雅，配套编译期注解解析自动生成注册类使得运行时性能更佳。

```Java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Subscribe {
	/**
	 * 指定订阅方法在那个线程执行
	 * POSTING：订阅方法将在发布事件的同一线程中被调用。这是默认值。它完全避免了线程切换意味着开销最小
	 * MAIN： 订阅方法将在Android的主线程中被调用
	 * BACKGROUND：订阅方法将在子线程中被调用
	 * ASYNC：订阅方法将在单独的线程中调用（非发布线程和主线程）
	 */
    ThreadMode threadMode() default ThreadMode.POSTING;

    //如果是true，则表示传递最近的粘性事件给订阅方法
    boolean sticky() default false;

    //订阅方法的优先级，默认值为0，此值越大优先级越高越先接受到事件
    int priority() default 0;
}
```

`Subscribe`用于标记在订阅者的方法上，可以配置订阅方法调用的线程，优先级和是否接受粘性事件。有了订阅者后，我们就可以注册订阅者了

# 3. 注册订阅者

需要弄清楚的是订阅者是指拥有订阅方法（`Subscribe`注解标记的方法）的类，在注册订阅者时传递的类this而非订阅方法，一个订阅者可能订阅多个主题（多个订阅方法），订阅方法只是用于接受事件的。下面我们看看`EventBus.getDefault().register(this)`做了什么事：

```Java
public class EventBus {

    static volatile EventBus defaultInstance;
    //3.1 获取单例对象
	public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
    
    //3.2 注册订阅者
    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }

}
```


https://www.jianshu.com/p/d9516884dbd4












