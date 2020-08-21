> 版权声明：本文为openXu原创文章[【openXu的博客】](http://blog.csdn.net/xmxkf)，未经博主允许不得以任何形式转载

@[TOC](目录)

[EventBus](https://github.com/greenrobot/EventBus)是[greenrobot](https://greenrobot.org/)开源的一个轻量级的发布/订阅事件总线框架，解耦事件发送者和接收者，简化组件之间的通信。什么事件总线、什么消息机制其实都是为了实现组件之间（两个类）的通信问题，事件就是一个对象，消息也是一个对象，说白了就是**一个类想要发送一个数据对象给另一个类**。

为了实现组件间的通信，我们可以使用古老的接口方式，一个类实现接口的方法，另一个类持有接口引用并调用方法实现数据传递，这种方式在MVP模式中被大量使用，缺点就是需要定义大量的接口使得项目非常臃肿，而且耦合比较严重。另一种就是使用Handler机制，但是事件发布者必须要持有Handler的引用，这又带来了局限性。还有一种就是使用观察者模式，事件接收者作为观察者订阅某个主题，事件发布者调用主题发布事件，它发出通知时并不需要直到谁订阅了它，只要是订阅了该主题的观察者都能收到事件。EventBus就是基于观察者模式实现的，这篇文章基于EventBus3.2版本源码讲解其工作原理。

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

> 调用regist(this)中的this就是订阅者，它里面带有Subscribe注解的订阅方法的参数类型是主题

- 注册和注销订阅者做了哪些事情，内部是怎么管理主题和订阅者的

> 内部通过两个Map缓存了订阅者和主题信息，注册和注销就是往Map中添加或者删除信息

- 事件是怎么发布出去的

> 根据事件类型从Map注册表中获取到所有订阅了该事件的订阅方法集合，遍历集合发布事件

-订阅者是怎么收到事件的

> 根据注解上的线程模式，切换到对应线程（主线程通过Handler切换、子线程放入线程池处理）后，通过反射调用订阅方法处理事件

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

# 3. register()注册订阅者

需要弄清楚的是订阅者是指拥有订阅方法（`Subscribe`注解标记的方法）的类，在注册订阅者时传递的类this而非订阅方法，一个订阅者可能订阅多个主题（多个订阅方法），订阅方法只是用于接受事件的。下面我们看看`EventBus.getDefault().register(this)`做了什么事：

```Java
public class EventBus {

    static volatile EventBus defaultInstance;
    //1 获取单例对象
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
    
    //2 注册订阅者
    public void register(Object subscriber) {
    	//获取需要注册的订阅者Class对象
        Class<?> subscriberClass = subscriber.getClass();
        //★3.1 根据订阅者Class查找订阅事件方法集合（有Subscribe注解的方法）
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
            	//3.2 遍历订阅者所有订阅方法，一个个注册
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
}
```

`register(subscriber)`方法首先查找订阅者中的所有订阅方法集合，然后遍历注册，下面看看查找方法过程：

## 3.1 查找订阅方法

```Java
class SubscriberMethodFinder {
	 //METHOD_CACHE是一个Map对象，缓存<订阅者Class为key，订阅方法集合为value>
	 private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();

	 List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
	 	//从缓存中获取方法集合，如果获取到就直接返回集合
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
        //是否忽略注解生成器，默认为false（翻看源码好像没有地方置为true，所以不管怎样，都会执行B）
        if (ignoreGeneratedIndex) {
        	//★A. 通过反射的方式获取订阅方法
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
        	//★B. 根据编译器注解生成类信息获取订阅方法(如果没有找到注册索引类，将还是通过反射的方式获取订阅方法)
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        //如果没有使用了Subscribe注解的方法，抛异常
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            //缓存订阅者和订阅方法
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
    
    //==>A. 通过反射的方式获取订阅方法（这个方法好像没有使用到）
    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
    	//FindState类包含订阅者class对象、订阅方法，它记录一个查找状态，方便遍历订阅者及父类中所有订阅方法
        FindState findState = prepareFindState();
        //初始化findState，查找状态为当前订阅类
        findState.initForSubscriber(subscriberClass);
        //findState.clazz==subscriberClass
        while (findState.clazz != null) {
        	//★ 3. 反射查询
            findUsingReflectionInSingleClass(findState);
            //findState.clazz=clazz.getSuperclass()遍历父类，以查找父类中的订阅方法
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
	//==>B. 根据编译器注解生成类信息获取订阅方法
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            //findState.subscriberInfo就是注解处理其生成的注册索引
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                //根据生成的注册索引查找订阅方法
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
            	//★ C. 如果没有注册信息，则通过反射获取订阅方法
                findUsingReflectionInSingleClass(findState);
            }
            //从父类中查找
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

    //==>C. 反射查找订阅方法
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        //反射获取订阅类的所有方法，返回Method数组
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        //遍历所有方法，找出带有@Subscribe的
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            //方法修饰符为public，并且非abstract和static的（MODIFIERS_IGNORE=Modifier.ABSTRACT | Modifier.STATIC ）
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            	//获取参数类型
                Class<?>[] parameterTypes = method.getParameterTypes();
                //如果只有一个参数
                if (parameterTypes.length == 1) {
                	//判断是否有Subscribe注解
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                    	//获取参数类型
                        Class<?> eventType = parameterTypes[0];
                        //判断是否已经添加过，如果没有则添加该订阅方法
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            //★★★添加一个订阅方法对象SubscriberMethod，它记录了方法对象、参数事件类型、Subscribe注解信息
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                	//订阅方法必须有且仅有一个参数，否则抛异常
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            	//订阅方法必须为public的，并且非abstract、static
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
}
```

查找订阅方法由`SubscriberMethodFinder`完成，`findSubscriberMethods(subscriberClass)`方法中有两个分叉：

- `findUsingReflection(subscriberClass)`直接通过反射的方式获取订阅方法，这个方法不会被调用
- `findUsingInfo()`使用注解处理器生成的注册索引查找订阅方法，如果没有找到注册索引类，将通过反射的方式。根据订阅类获取注册索引后面会讲到

反射获取订阅方法最后都是调用`findUsingReflectionInSingleClass(findState)`查找出符合条件（带Subscribe注解、public修师的非abstract非static、有且仅有一个参数）的订阅方法。

## 3.2 subscribe()注册

接着分析`register()`方法，查找出订阅类的所有订阅方法集合`subscriberMethods`后遍历调用`subscribe()`实现订阅注册：

```Java
#### org.greenrobot.eventbus.EventBus
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
	//获取订阅方法的参数，其实就是事件类型
    Class<?> eventType = subscriberMethod.eventType;
    //Subscription类保存了订阅类对象和当前订阅方法 
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    //subscriptionsByEventType是一个Map<Class<?>, CopyOnWriteArrayList<Subscription>>，保存着事件类型为key的所有订阅者数据集合
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
    	//如果该事件还没有注册过订阅者，创建一个集合保存到map中
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
    	//如果发现已经注册过抛异常
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }
    //将上面创建的Subscription信息保存到subscriptions集合合适的位置（根据订阅优先级）
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    //typesBySubscriber是Map<Object, List<Class<?>>>，订阅类为key，订阅方法参数(事件类型)为value
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    //将事件类型添加到该订阅类中所有的事件类型集合中
    subscribedEvents.add(eventType);
    //粘性事件最后一个部分讲解
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

订阅的动作其实就是将订阅信息缓存到两个Map中，一个`subscriptionsByEventType`缓存了以事件类型为key，订阅方法信息(订阅方法信息、方法所在的订阅类)集合，可以想象当post(eventType)发布事件后，就是根据这个map查找到哪些订阅方法应该被执行，然后通过反射执行订阅方法。`typesBySubscriber`则保存了订阅类为key，订阅事件集合为value的数据，当取消订阅`unregister(this)`时会根据订阅类找到所有的事件集合，然后遍历事件集合清除第一个Map中的注册信息.

```Java
                第一个Map                                    第二个Map
         subscriptionsByEventType                       typesBySubscriber
      Map<eventType, List<Subscription>>            Map<Object, List<eventType>>
      key=事件类型    value=订阅方法集合              key=订阅类  value=订阅类中事件集合
```

# 4. unregister()注销订阅

分析了注册订阅的过程，我们大概都能猜到注销订阅`unregister(this)`无非就是将订阅信息从两个Map中移除，还是看看源码验证一下：

```Java
#### org.greenrobot.eventbus.EventBus
public synchronized void unregister(Object subscriber) {
    //从第二个Map中根据订阅类对象获取到该类中 所有订阅方法 的 事件类型
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
    	//★ 遍历事件，从第一个Map中移除 该事件 对应的 属于当前订阅者的 所有订阅方法
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        //★ 第二个Map移除该订阅者的所有信息
        typesBySubscriber.remove(subscriber);
    } else {
        Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}

private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
	//根据事件获取所有订阅方法
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            //如果订阅方法所属的类 == 需要注销的订阅类 ，则从集合中移除
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```

正如上面的猜测，`unregister(this)`方法就是根据this在两个Map中找到相关订阅事件及方法信息，并将其移除。

# 5. post()发布事件

```Java
#### org.greenrobot.eventbus.EventBus
//ThreadLocal用于为每个线程提供线程内的局部变量
private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
    @Override
    protected PostingThreadState initialValue() {
    	//为线程初始化一个变量
        return new PostingThreadState();
    }
};

public void post(Object event) {
	//★ 获取post所在的线程的PostingThreadState对象，PostingThreadState保存了事件队列、订阅者信息等
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    //将事件添加到事件队列中
    eventQueue.add(event);

    if (!postingState.isPosting) {
    	//是否为主线程isMainThread赋值
        postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
        postingState.isPosting = true;   //更改状态为正在发布
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
            	//★ 遍历事件队列，发布事件并将事件移除
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}

private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
	//当前事件类型
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    //eventInheritance是否向上查找事件的父类，默认为true
    if (eventInheritance) {
    	//获取当前事件类型及其所有父类类型的集合
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            //遍历事件及其父类，发布事件
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
   		//发布事件
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            Log.d(TAG, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            //当发布的时间没有订阅者时，EventBus将发布此事件
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```

`post(event)`方法通过`ThreadLocal`获取发布事件所在线程的`PostingThreadState`对象，该对象保存了事件队列以及该事件的订阅者信息，这跟Handler机制中**Looper**保存当前线程的消息队列是类似的实现方式。将事件添加到事件队列中然后遍历队列调用`postSingleEvent()`方法，`postSingleEvent()`方法中主要根据`eventInheritance`判断是否需要向上查找事件的父类，最后都调用了`postSingleEventForEventType()`发布事件：

```Java
#### org.greenrobot.eventbus.EventBus
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
    	//从上面讲过的第一个Map中根据事件类型获取所有该事件订阅者信息集合
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
    	//遍历订阅者（这里的订阅者Subscription是每一个订阅方法，以及该方法所属的订阅类信息）
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                //★ 继续调用postToSubscription()分发事件
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    //如果没有订阅者，返回false，上一个方法会发布一个默认的NoSubscriberEvent事件
    return false;
}
```

`postSingleEventForEventType()`方法根据事件类型，从`subscriptionsByEventType`这个注册Map中得到该事件所有订阅信息集合，然后遍历集合，调用`postToSubscription()`分发事件：

# 6. 事件分发

```Java
private final HandlerPoster mainThreadPoster;          //Handler
private final BackgroundPoster backgroundPoster;       //Runnable
private final AsyncPoster asyncPoster;                 //Runnable
/*
 * POSTING：订阅方法将在发布事件的同一线程中被调用。这是默认值。它完全避免了线程切换意味着开销最小
 * MAIN： 订阅方法将在Android的主线程中被调用
 * BACKGROUND：订阅方法将在子线程中被调用
 * ASYNC：订阅方法将在单独的线程中调用（非发布线程和主线程）
 */
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
	//判断订阅方法的线程模式 
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:   //在发布事件的线程中调用订阅方法处理事件
            invokeSubscriber(subscription, event);
            break;
        case MAIN:      //需要在主线程中处理事件
            if (isMainThread) { //如果发布事件在主线程，那就直接调用订阅方法
                invokeSubscriber(subscription, event);
            } else {            //如果是在主线程发布事件，则将事件放到mainThreadPoster队列，通过Handler切换到主线程
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case BACKGROUND:   //需要在子线程处理事件
            if (isMainThread) {
            	//如果是主线程发布的事件，则放到backgroundPoster队列中，通过线程池处理事件
                backgroundPoster.enqueue(subscription, event);
            } else {
            	//子线程发布的事件可直接在当前线程处理
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:  //单独的线程中调用（非发布线程和主线程），放到asyncPoster队列中
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}

//通过反射调用订阅事件的订阅方法
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```

`postToSubscription()`方法主要根据订阅方法上`Subscribe`注解的线程模式，将事件分情况处理，其中最简单的一种是直接在发布事件`post()`的所在线程中调用`invokeSubscriber()`通过反射执行订阅方法。另一种则需要放到对应的事件队列然后切换线程：

## HandlerPoster

`HandlerPoster`是通过主线程Looper创建的Handler，它维护了一个单向链表队列，遍历队列通过发送消息切换到主线程，然后在`handlerMessage()`中调用`eventBus.invokeSubscriber(pendingPost)`发布事件，其实就是上面通过反射调用订阅方法。就是多了Handler切换主线程的步骤

```Java
final class HandlerPoster extends Handler {
	//PendingPostQueue是一个单向链表结构，跟Handler的MessageQueue差不多，不同的是MessageQueue多了延迟时间排序
    private final PendingPostQueue queue;   
	void enqueue(Subscription subscription, Object event) {
	    PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
	    synchronized (this) {
	    	//将封装的PendingPost放到队列中
	        queue.enqueue(pendingPost);
	        if (!handlerActive) {
	            handlerActive = true;
	            //发送消息
	            if (!sendMessage(obtainMessage())) {
	                throw new EventBusException("Could not send handler message");
	            }
	        }
	    }
	}

	@Override
	public void handleMessage(Message msg) {
		//由于EventBus初始化时创建的HandlerPoster对象使用的是Looper.getMainLooper()主线程的Looper，所以handlerMessage将在主线程中处理消息
		while(true){
			PendingPost pendingPost = queue.poll();
			//...
			//在主线程中调用invokeSubscriber()
			eventBus.invokeSubscriber(pendingPost);
			//...
		}
	}
}
```

## BackgroundPoster && AsyncPoster

`BackgroundPoster`和`AsyncPoster`一样都是一个Runnable对象，他们也维护了一个`PendingPostQueue`事件队列，`enqueue()`方法会将事件放入队列中，然后将this放入到`EventBus`提供的`ExecutorService`线程池中，在run()方法中会遍历事件队列，执行`eventBus.invokeSubscriber(pendingPost)`通过反射调用订阅方法。


# 7. 粘性事件

粘性和普通事件的根本区别就是**事件发布之后注册的订阅者是否能收到事件**，普通事件在事件发布之前已经完成了注册，当事件发布后就能驱动调用订阅方法。而粘性事件发布做了两个动作，一个是调用post()将事件按照普通事件发送出去，对已经注册的订阅者也是可以收到事件的。第二个动作就是将事件保存到`stickyEvents`中，方便之后注册的粘性订阅者能收到事件。下面我们看看源码：

**发布粘性事件**

```Java
#### org.greenrobot.eventbus.EventBus

private final Map<Class<?>, Object> stickyEvents;
//
public void postSticky(Object event) {
    synchronized (stickyEvents) {
    	// 将事件放入stickyEvents中
        stickyEvents.put(event.getClass(), event);
    }
    // 发布普通事件
    post(event);
}
```

**注册粘性订阅方法**

```Java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
	//...
	//省略注册部分（往两个Map中缓存注册信息）
	//...
    //判断如果订阅方法是粘性的（Subscribe注解的sticky属性为true）
    if (subscriberMethod.sticky) {
    	//eventInheritance是否向上查找事件的父类，默认为true
        if (eventInheritance) {
            //遍历stickyEvents保存的粘性事件
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                //订阅方法参数类型eventType与粘性事件类型candidateEventType相同，或者是其父类，则表示该粘性时间应该发布给该订阅方法
                if (eventType.isAssignableFrom(candidateEventType)) {
                	//获取事件对象
                    Object stickyEvent = entry.getValue();
                    //处理粘性事件
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
        	//不遍历事件父类的情况，最后处理事件是一样的
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

粘性订阅者在注册订阅时，能收到订阅之前发布的保存在`stickyEvents`中的事件，`checkPostStickyEventToSubscription()`方法最后调用了`postToSubscription()`分发事件（请看上面第6部分事件分发）

# Subscriber Index

[Subscriber Index](https://greenrobot.org/eventbus/documentation/subscriber-index/)是EventBus3.0推出的订阅注册索引，它是通过编译期注解处理器生成辅助类，在构建时对注解进行处理生成索引表，避免在程序运行期由于反射注册带来性能问题。

**使用**

在模块build.gradle中添加下面的脚本，构建项目生成`eventBusIndex`索引类，然后在Application中使用`EventBus.builder().addIndex(new MyEventBusIndex()).installDefaultEventBus();`将索引类传给EventBus并生成一个单例对象。如果是组件化开发，可多次调用addIndex()添加每个模块的注册索引，所有的索引对象将保存在一个List集合中。

```xml
#### Java ：使用annotationProcessor
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [ eventBusIndex : 'com.example.myapp.MyEventBusIndex' ]
            }
        }
    }
}
 
dependencies {
    def eventbus_version = '3.2.0'
    implementation "org.greenrobot:eventbus:$eventbus_version"
    annotationProcessor "org.greenrobot:eventbus-annotation-processor:$eventbus_version"
}

#### Kotlin：使用kapt
apply plugin: 'kotlin-kapt' // ensure kapt plugin is applied
dependencies {
    def eventbus_version = '3.2.0'
    implementation "org.greenrobot:eventbus:$eventbus_version"
    kapt "org.greenrobot:eventbus-annotation-processor:$eventbus_version"
}
kapt {
    arguments {
        arg('eventBusIndex', 'com.example.myapp.MyEventBusIndex')
    }
}
```

**解析**

```Java
#### 自动生成的注册表类
public class MyEventBusIndex implements SubscriberInfoIndex {
	//SUBSCRIBER_INDEX保存了所有订阅者信息，key为订阅者类，value为SubscriberInfo保存了该类所有订阅方法及父类的订阅方法信息
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;

    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();
        //静态代码块中会添加所有订阅信息（项目中带有Subscribe注解的）到注册表中
        putIndex(new SimpleSubscriberInfo(com.openxu.vedio.ui.MediaPlayerActivity.class, true,
                new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onMessageEvent", java.util.List.class, ThreadMode.MAIN),
        }));

        putIndex(new SimpleSubscriberInfo(com.openxu.vedio.MainActivity.class, true, new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onMessageEvent", java.util.List.class, ThreadMode.MAIN),
        }));

    }
    //私有的put方法，表示这个注册表只能通过注解解析器自动生成
    private static void putIndex(SubscriberInfo info) {
        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
    }
    //提供get方法，根据订阅类找到对应的订阅信息
    @Override
    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
        if (info != null) {
            return info;
        } else {
            return null;
        }
    }
}
```

注解解析器会自动生成注册表，将所有打了Subscribe注解的方法及类根据规则保存在注册表Map中。在3.2讲解注册`register()`的时候讲到`findUsingInfo()`方法，该方法通过索引表获取订阅信息。首先根据订阅类型在所有注册索引集合`subscriberInfoIndexes`中找到对应的`SubscriberInfo`订阅信息，然后就能获取到该类上的所有订阅方法了：

```Java
#### org.greenrobot.eventbus.SubscriberMethodFinder

private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
    	//☆ 根据订阅者类型从注册索引中获取该订阅者所有订阅信息
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
        	//获取订阅者所有订阅方法
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod : array) {
            	//判断是否已经添加过
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
        	//如果没有注册表，还是通过反射的方式获取订阅信息
            findUsingReflectionInSingleClass(findState);
        }
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}

private SubscriberInfo getSubscriberInfo(FindState findState) {
    if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
        SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
        if (findState.clazz == superclassInfo.getSubscriberClass()) {
            return superclassInfo;
        }
    }
    //subscriberInfoIndexes就是初始化EventBus时调用addIndex添加保存注册表的集合List<SubscriberInfoIndex> subscriberInfoIndexes;
    if (subscriberInfoIndexes != null) {
        // 遍历索引类集合
        for (SubscriberInfoIndex index : subscriberInfoIndexes) {
            //查找索引类中对应订阅类的订阅信息
            SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
            if (info != null) {
                return info;
            }
        }
    }
    return null;
}

#### 订阅注册表
@Override
public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
    SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
    if (info != null) {
        return info;
    } else {
        return null;
    }
}
```

# 9. 总结

- `register(this)`通过编译注解器生成的注册索引或者反射方式获取订阅者上所有的订阅方法后缓存到两个Map注册表中
- `post(event)`以event类型为key从Map注册表中获取到所有订阅该类型事件的订阅方法集合，然后遍历集合根据Subscribe注解的线程模式判断是否需要切换线程（主线程通过`Handler`、子线程放入线程池），最后通过反射调用订阅方法处理事件
- `unregister(this)`则根据订阅者类型获取到其所有订阅方法，然后移除掉跟该订阅者相关的缓存在两个Map注册表中的数据
- 粘性事件的发布比普通事件发布多了一个保存在`stickyEvents`中的步骤，当粘性订阅者注册订阅时，会遍历之前发布的粘性事件分发给订阅方法
- Subscriber Index注册索引通过在编译构建时对注解进行解析生成注册索引表，避免项目运行时通过反射获取订阅方法影响性能

```Java

                                                      register(this) 
                                                           |  
                                                           |  使用Subscriber Index注册索引或者反射获取订阅方法信息，缓存到两个Map中
                                                           ⬇
                                        --------------EventBus-----------------
	                                    |                                     |
           post(event1) --------------->|       subscriptionsByEventType      | ------------------->  SubscripClass1.eventMethod1(event1)
										|  Map<eventType, List<Subscription>> |        
           post(event2) --------------->|   key=事件类型    value=订阅方法集合  | ------------------->  SubscripClass1.eventMethod2(event2)
                                        |                                     |   根据eventType找到订阅方法
           post(event3) --------------->|          typesBySubscriber          |   切换线程后反射调用订阅方法
                                        |    Map<Object, List<eventType>>     | ------------------->  SubscripClass2.eventMethod(event3) 
           post(event4) --------------->|   key=订阅类  value=订阅类中事件集合  |
                                        |                                     | --------------------> SubscripClass3.eventMethod(event4)
                                        ---------------------------------------
                                                          ⬆
                                                          |  根据this订阅类删除两个Map中的订阅方法和类
                                                          |
                                                     unregister(this)

```



