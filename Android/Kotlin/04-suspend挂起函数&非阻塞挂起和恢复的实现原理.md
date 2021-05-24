

> 版权声明：本文为openXu原创文章[【openXu的博客】](http://blog.csdn.net/xmxkf)，未经博主允许不得以任何形式转载

@[toc]

上一篇文章中我们了解了协程的一些术语、相关类的作用和协程的基本使用，如果仅仅掌握这些也是可以上手使用协程了，但是如果在使用过程中遇到了一些莫名其妙的问题却找不到原因，不知道怎么解决甚至怀疑从一开始就使用错了，那是因为没有理解协程的实现原理。从这篇文章开始，我们从源码角度深入解读协程，当然源码跟踪只能从一个角度着手，不可能做到全面解读，但是打通了一条路线你会发现其他的分支都是相似的。

## 1. 自定义挂起函数

函数是对为实现某个功能或者计算某个结果的多行代码的封装，挂起函数也是一样，与普通函数不同的是挂起函数"通常"被放到其他线程(异步)，并且能在不阻塞当前线程的情况下同步的得到函数的结果。不阻塞当前线程就是**挂起**，它指的是当协程中调用挂起函数时会记录当前的状态并挂起(暂停)协程的执行（释放当前线程），以达到非阻塞等待异步计算结果的目的。说白了就是不阻塞协程代码块所在的线程但暂停挂起点之后的代码执行，当挂起函数执行完毕再恢复挂起点后的代码执行。比如下面示例中，在主线程开启一个协程，调用挂起函数`delay()`延迟1s后在更新UI，与`Thread.sleep`不同的是delay不会阻塞主线程，这个延迟动作是在子线程中完成的。

```kotlin
CoroutineScope(Dispatchers.Main).launch{
    //UI线程，代码块中的代码按顺序一行行执行
    delay(1000)    //挂起点
    textView.text = "延迟1s" //续体
}
```

### 1.1 为什么需要自定义挂起函数

函数的作用就是对功能的封装，比如从服务器获取用户信息、将数据存在在本地数据库等都可以被封装成一个函数，如果把这个函数定义为普通的函数，在调用这些函数时就会阻塞当前线程（当前线程去执行这个函数就不能干别的事情了）。所以在Android这种UI线程环境中我们通常需要开启子线程来调用这些函数，并在函数执行完毕后手动切回UI线程。如果将这些函数定义为挂起函数，这些步骤就可以让协程自动帮我们完成了，而我们关注的侧重点是函数功能代码的封装。Retrofit http请求客户端和Room数据库等添加了对协程的支持，可以将功能接口定义为挂起函数，而这些挂起函数通俗的说都属于自定义挂起函数（非协程库提供的挂起函数）。

挂起函数的目的是用来挂起协程的执行等待异步计算的结果，所以一个挂起函数通常有两个要点：**挂起**和**异步**，接下来我们一步步来实现自定义挂起函数

### 1.2 suspend到底有什么用？

所有的挂起函数都由`suspend`关键字修饰，是不是有suspend修饰的函数都是挂起函数？答案是NO，比如：

```kotlin
//定义一个User实体类
data class User(val name:String)

//定义一个函数模拟耗时获取User对象
suspend fun getUser():User{
    println("假的挂起函数${Thread.currentThread()}")
    Thread.sleep(1000)
    return User("openXu")
}
```

`getUser()`函数有`suspend`修饰，但是IDE提示`Remove redundant suspend modifier`移除冗余的suspend修饰符，为什么呢？我们先搞清楚suspend到底是什么？它有什么作用？

`suspend`是kotlin中的修饰符，kotlin源码最终都将被编译为java的class执行，而java中并没有这个修饰符，所以`suspend`仅仅在编码和编译阶段起作用：

- 在编码阶段：suspend仅仅作为一个标志，表示这是一个挂起函数，它只能在协程或者其他挂起函数中被调用，如果在普通函数中调用IDE会提示错误；并且它可以调用其他挂起函数
- 在编译阶段：由suspend修饰的函数被编译为class后，函数会被增加一个`Continuation`(续体)类型的参数

> 借助Android Studio-->Tools菜单-->Kotlin-->Show Kotlin Bytecode-->Decompile查看kotlin对应的java源码

上面的`getUser()`方法被编译后对应的java代码如下：

```java
   public static final Object getUser(@NotNull Continuation $completion) {
      String var1 = "假的挂起函数" + Thread.currentThread();
      boolean var2 = false;
      System.out.println(var1);
      Thread.sleep(1000L);
      //假挂起函数根本原因是函数返回值不是COROUTINE_SUSPENDED
      return new User("openXu");
   }
```

对于jvm来说，这就是一个参数为`Continuation`类型的普通函数，这个参数在函数体中并没有被使用，所以是一个多余的参数，而`suspend`的作用就是在编译时增加这个参数，所以`suspend`修饰符就是多余的。

怎样让`suspend`修饰符不多余？就是在函数体类要使用`Continuation`类型的参数，而这个参数是编译器自动添加的，在编码阶段肯定是没办法使用，只能在运行阶段去使用，怎样在运行阶段使用它呢？答案就是调用协程库提供的挂起函数。**要真正实现挂起，必须调用一些协程库中定义的顶层挂起函数**，只有这些库自带的挂起函数才能真正实现协程的挂起，而调用他们的地方才是真正的**挂起点**(真正的挂起操作是这些顶层挂起函数内部调用了`trySuspend()`并返回了COROUTINE_SUSPENDED标志使得当前线程退出执行从而挂起协程)。

### 1.3 不完全挂起函数（组合挂起函数）

为了真正挂起协程就要调用协程库中的挂起函数，协程库的挂起函数很多，是不是随便调用一个就ok呢？比如：

```kotlin
suspend fun getUser():User{
	//调用自带的挂起函数实现挂起
	delay(1000)          //真正的挂起点
	//以下为函数真正的耗时逻辑
    Thread.sleep(1000)        //模拟耗时
    return User("openXu")
}
```

在getUser()中调用了`delay()`，IDE不再提示`suspend`多余(通过查看反编译后的java代码发现Continuation参数确实在函数体中被使用)，但是这个挂起对getUser()并没有意义，我们分析getUser()的执行，首先在挂起作用域中调用这个函数，函数体第一句调用了delay()挂起了协程，协程所在的线程(当前线程)将会停止继续执行(非阻塞)，直到1s延迟完成，协程将恢复当前线程继续执行下面的函数代码，也就是说**函数体一部分耗时计算不是在协程被挂起的状态下执行的，而是直接运行在协程所在的线程(执行式阻塞当前线程)，这种函数称为不完全挂起函数**。

> 协程中并没有关于不完全挂起函数的定义，为了方便大家更好的理解挂起函数，笔者结合实际在定义挂起函数时的问题自创了这个名词，其实它就是一个组合挂起函数（在函数中调用其他挂起函数）

之前项目开发过程中就遇到过不完全挂起函数造成卡顿的问题，项目使用了Retrofit+协程，将接口方法定义为挂起函数：

```java
@GET("tree/json")
suspend fun getTree(): ApiResult<MutableList<Category>>
```

通过`viewModelScope.launch{}`或者`MainScope().launch {}`在UI线程中启动协程，然后直接调用挂起接口函数从服务器请求数据：

```java
viewModelScope.launch {
    try {
        if(showDialog) dialog.value = true  //UI线程，修改DialogLiveData值为true，显示请求对话框
        //×××错误的方式：调用不完全挂起函数
        val category = RetrofitClient.apiService.getTree()

        //√√√正确的方式：将不完全挂起函数的未挂起部分挂起
        /*val category = withContext(Dispatchers.IO){
        	RetrofitClient.apiService.getTree()
        }*/

        if(showDialog) dialog.value = false //UI线程
    } catch (e: Exception) {
        if(showDialog)
            dialog.value = false
        onError(e, showErrorToast)
    }
}
```

每次应用程序启动后第一次调用这个接口请求数据时，请求对话框都会延迟一会儿才能显示或者卡顿一会儿，再次请求这个接口就不会卡了。刚开始以为是项目太大接口太多，或者因为模块化开发导致Retrofit需要做的事情太多了造成卡顿，但是不知道怎么解决，后来研究挂起函数后才明白，**自定义挂起接口方法`getTree()`不就是个不完全挂起函数吗？**调用`getTree()`方法后，Retrofit通过反射创建接口代理对象、解析接口方法注解和参数，创建Call对象等操作都是协程当前线程(UI线程)执行的，只有真正调用call.enqueue()的地方才挂起协程，这就造成了主线程的阻塞；为什么只有第一次卡顿呢？Retrofit将解析后的接口方法`ServiceMethod`缓存到了`serviceMethodCache`的Map中，下次再调用这个接口方法时，就不需要去解析方法注解和参数了，直接从Map中取就可以了。

> Retrofit只有在真正执行请求的时候才调用协程库的挂起函数`suspendCancellableCoroutine()`挂起协程，可在retrofit2.KotlinExtensions.kt文件中查看源码

```java
//ServiceMethod的adapt()中调用call的扩展函数await()，并传入continuation作为参数
//这种调用方式看起来有些奇怪，其实就是java调用kotlin代码
KotlinExtensions.await(call, continuation);

/**Call的扩展方法，被定义在retrofit2.KotlinExtensions.kt文件中*/
suspend fun <T : Any> Call<T>.await(): T {
    return suspendCancellableCoroutine { continuation ->
        ...
        //发起请求：相当于this.enqueue，而扩展方法中的this就是被扩展的类也就是call对象
        enqueue(object : Callback<T> {
            override fun onResponse(call: Call<T>, response: Response<T>) {
                if (response.isSuccessful) {
                    val body = response.body()
                    //恢复协程执行，返回响应结果
                    continuation.resume(body)
                } else {
                	//恢复协程执行，抛出一个异常
                    continuation.resumeWithException(HttpException(response))
                }
            }
            ...
        })
    }
}
```

怎样避免不完全挂起函数造成的线程阻塞(主线程执行了函数的一部分耗时代码)？就是让自定义挂起函数的整个函数体{}都是在协程挂起之后执行，通常将函数体写为Lambda表达式作为参数传递给顶层挂起函数。

### 1.4 真正的、完全的挂起函数

协程库定义了以下顶层挂起函数方便我们自定义挂起函数：

```kotlin
//①. 不常用
public suspend inline fun <T> suspendCoroutine(crossinline block: (Continuation<T>) -> Unit): T
//②. 常用
public suspend inline fun <T> suspendCancellableCoroutine(
    crossinline block: (CancellableContinuation<T>) -> Unit
): T
```

这两个函数的作用是捕获当前的协程的续体对象（下面会讲到的`SuspendLambda`对象）作为参数，其实是`SuspendLambda`中调用到挂起函数时将this作为参数传入的。通常被用于定义自己的挂起函数。它们都调用了另一个顶层挂起函数`suspendCoroutineUninterceptedOrReturn()`用于对参数续体对象进行包装，然后执行作为参数传入的代码块block，在等待恢复信号期间(代码块在未来某一时刻调用续体的resume系列方法)挂起协程的执行。

这两个函数的区别是，`suspendCancellableCoroutine()`函数会用将续体对象拦截包装为一个`CancellableContinuation`类型，`CancellableContinuation`是一个可以`cancel()`取消的续体，用于控制协程的生命周期。尽管协程库提供了不可取消的`suspendCoroutine()`函数，但推荐始终选择使用`suspendCancellableCoroutine()`处理协程作用域的取消，从底层API取消事件传播。

下面我们就通过调用`suspendCancellableCoroutine`改造一下自己的挂起函数：

```kotlin
//调用suspendCancellableCoroutine()，将函数体作为参数传入
suspend fun getUser(): User = suspendCancellableCoroutine {
	//被拦截后的可取消续体对象
    cancellableContinuation ->
    println("挂起函数执行线程${Thread.currentThread()}") //Thread[main,5,main]
    Thread.sleep(3000)
    cancellableContinuation.resume(User("openXu"))
    cancellableContinuation.cancel()
}

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
	//在主线程中开启一个协程
	CoroutineScope(Dispatchers.Main).launch{
	    showProgressDialog()       //UI：显示进度框
	    val user = getUser()       //挂起点
	    tv.text = user.name        //更新UI
	    dismissProgressDialog()    //UI：隐藏进度框
	}
}
```

getUser()函数直接被赋值为协程库提供的挂起函数，函数体是作为参数传入的，这样调用getUser()的地方就相当于调用了suspendCancellableCoroutine()，会立马挂起协程，这样getUser()才是**真正的、完全的挂起函数**

上述示例是在Activity环境中，在UI线程开启一个协程后调用挂起函数`getUser()`，并且在之前和之后显示和隐藏进度圈，运行项目可以观察到进度圈显示后，卡顿了3s然后隐藏。

不是说挂起不会阻塞当前线程吗？为什么还会卡顿？因为我们并没有指定挂起函数执行的线程，默认就在当前UI线程调度了，就相当于在UI线程进行了耗时操作。目前我们的自定义挂起函数只是实现了挂起，但这个挂起并没有太大意义，因为是单线程的，所以为了实现挂起不阻塞主线程，还缺少异步。

> **挂起函数不一定是在子线程执行的**。如果你在其他文章中看到别人说挂起函数是在子线程中执行的，听话：鼠标移到浏览器右上角，看见红色叉叉了吗？叉掉它。

### 1.5 异步挂起函数

我们对`getUser()`函数进行改造，在black代码块中创建一个子线程，使得函数体代码运行在子线程中，运行项目就不会出现卡顿了：

```kotlin
suspend fun getUser(): User = suspendCancellableCoroutine {
    cancellableContinuation ->
    //创建子线程实现异步
    Thread {
        try {
            Thread.sleep(3000)
            when(Random.nextInt(10)%2){ 
                0->{ //10以内随机数如果是偶数返回成功
                    cancellableContinuation.resume(User("openXu"))
                    cancellableContinuation.cancel()
                }
                1-> throw Exception("模拟异常")
            }
        }catch (e:Exception){
        	//通过resumeWithException()用一个异常恢复协程执行
            cancellableContinuation.resumeWithException(e)
        }
    }.start()
}

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
	//在主线程中开启一个协程
	CoroutineScope(Dispatchers.Main).launch{
		showProgressDialog(null)   //UI：显示进度框
        try {
            val user = getUser() //挂起点
      		tv.text = user.name        //更新UI
        }catch (e:Exception){
            FLog.d("挂起异常${e.message}")
        }
        dismissProgressDialog()    //UI：隐藏进度框
	}
}
```

### 1.6 withContext()

上面通过3步我们自定义了一个挂起函数：

- 函数使用`suspend`修饰
- 调用协程库提供的`suspendCoroutine()`或(强制推荐)`suspendCancellableCoroutine()`挂起函数实现真正、完全的挂起
- 开启子线程执行函数体，最后通过`Continuation`续体对象返回函数结果恢复协程执行

步骤是非常清晰，但是代码量和代码清洁度不容乐观，有没有更方便的方式自定义挂起函数呢？协程库提供了`withContext()`函数，严格说起来它并不是用来自定义挂起函数的，而通常用于线程切换，只是它恰好能实现**挂起**和**异步**这两个要素，并且还接受一个函数block作为参数：

```kotlin
suspend fun getUser(): User = withContext(Dispatchers.IO) {
    Thread.sleep(3000)
    when(Random.nextInt(10)%2){ //10以内随机数如果是偶数返回成功
        0->User("openXu")
        else-> throw Exception("模拟异常")
    }
}
```

### 1.7 withContext()和suspendCancellableCoroutine()怎么选？

```java
//1. withContext()
public suspend fun <T> withContext(
    context: CoroutineContext,
    block: suspend CoroutineScope.() -> T
): T {
	...
    return suspendCoroutineUninterceptedOrReturn sc@ { uCont ->
		...
    }
}

//2. suspendCancellableCoroutine()
public suspend inline fun <T> suspendCancellableCoroutine(
    crossinline block: (CancellableContinuation<T>) -> Unit
): T =
    suspendCoroutineUninterceptedOrReturn { uCont ->
        ...
    }
//都是通过suspendCoroutineUninterceptedOrReturn()实现的
public suspend inline fun <T> suspendCoroutineUninterceptedOrReturn(crossinline block: (Continuation<T>) -> Any?): T {
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    ...
}
```

`withContext()`和`suspendCancellableCoroutine()`函数都是协程库提供的顶层挂起函数，发现他们都调用了`suspendCoroutineUninterceptedOrReturn()`来捕获协程的续体对象，并且都接受一个函数类型作为参数，这就为自定义挂起函数提供了可行性。但是究竟该选用哪个函数呢？`withContext()`使用更简洁所以都用它就完事了？

withContext()可接受协程上下文作为参数，这样我们可以传入`Dispatchers`调度器自动切换挂起函数执行的线程；而`suspendCancellableCoroutine()`并不具备切换线程的能力，通常需要我们手动创建线程。所以**当函数体已经具备异步能力的就选`suspendCancellableCoroutine()`，而不具备异步能力(需要手动切线程)的就选`withContext()`**

> 需要注意的是，如果使用`withContext()`则直接通过return返回计算结果或者通过throw抛异常来返回错误从而恢复协程执行；而用`suspendCancellableCoroutine()`的时候则通过续体`continuation`的`resume`系列方法来实现协程恢复

比如Retrofit对挂起函数的支持，在执行请求的时候调用的Call扩展函数await()中就是使用`suspendCancellableCoroutine()`，因为函数体中调用的Call.enqueue()已经具备异步能力，就没必要再切子线程了。Call还有一个同步请求的方法`execute()`，如果我们调用这个方法请求数据的话，就可以通过`withContext()`来实现自定义挂起函数了：

```java

//Retrofit源码
suspend fun <T : Any> Call<T>.await(): T {
  return suspendCancellableCoroutine { continuation ->
    //调用Call的enqueue()进行异步请求
    enqueue(object : Callback<T> {
      override fun onResponse(call: Call<T>, response: Response<T>) {
        if (response.isSuccessful) {
		val body = response.body()
		continuation.resume(body)//恢复协程执行，返回请求结果
        } else {
          continuation.resumeWithException(HttpException(response))//恢复协程执行，抛异常
        }
      }
      ...
    })
  }
}

//使用withContext()改造
suspend fun <T : Any> Call<T>.await(): T {
    return withContext(Dispatchers.IO){
        val response = execute()   //调用同步请求
        if (response.isSuccessful) {
            val body = response.body()
            return@withContext body!! //恢复协程执行，返回请求结果
        } else {
            throw HttpException(response)//恢复协程执行，抛异常
        }
    }
}
```

## 2. 挂起、恢复的实现细节

上面通过示例了解了在开发中应该怎样将一个耗时函数定义为挂起函数，接下来我们继续探索挂起函数底层是怎么实现协程的挂起和恢复的

### 2.1 Continuation续体

`Continuation`是一个接口，它有一个子接口`CancellableContinuation`，从名字上可以看出它表示可以`cancel`的续体，它们的实现类`ContinuationImpl`
和`CancellableContinuationImpl`都是`internal`修饰的(内部使用)，**协程库根本没打算让我们直接创建续体对象，挂起函数的续体对象都是通过`suspendCoroutineUninterceptedOrReturn()`函数自动获取的。**所以目前我们先不去扣它们的实现细节，只通过接口的定义来了解一下续体是什么，以及续体的作用：

Continuation接口：

- 成员`context`: 当前协程的CoroutineContext上下文对象
- `resumeWith(result: Result<T>)`：用一个结果来恢复协程的执行，这个结果(成功or异常)被封装为Result对象内
- 扩展函数`resume()`、`resumeWithException()`：这两个扩展函数是为了更方便我们调用的，毕竟直接调用`resumeWith()`需要我们手动将结果数据或者异常包装为`Result`对象

CancellableContinuation接口：

- 成员`isActive`、`isCompleted`、`isCancelled`：Boolean值，表示当前续体是否是活动状态、是否已完成、是否已取消
- `cancel(cause: Throwable? = null)`：可选的通过一个异常取消当前续体执行，因为可以取消，所以多了上面的3个状态属性

关于续体，我们现在需要了解到的是：**续体是对挂起点之后代码块的封装，表示挂起函数执行完后需要恢复继续执行的代码。同时可以将它当作一个CallBack回调，因为可以调用其resume方法返回挂起点函数的计算结果（成功or失败）**：

```kotlin
suspend fun getUser(): User = suspendCancellableCoroutine{
    continuation ->
    ...
    //调用resume()返回函数结果，并恢复协程执行
    continuation.resume(User("openXu"))
}
fun main() {
    runBlocking {
        //调用挂起函数：将被隐式传入一个Continuation参数
        val user = getUser()    //挂起点
        //简单的理解为下面的代码就是挂起点1的续体（每个挂起点之后的代码，当协程恢复后需要执行的代码）
        println("请求结果$user")
        ...
    }
```

如果觉得上面示例中的注释还不直观，那将上面的代码手动改成如下方式，这样就更加通俗了：

```kotlin
//自定义续体接口
interface Continuation<T>{
	//续体的回调抽象方法，它的实现方法体就是对挂起点之后代码的封装
    fun resume(t:T)   
}
//getUser()函数接受一个续体对象作为参数
fun getUser(continuation: Continuation<User>) {
    ...
    //1. 作为回调，返回函数结果
    continuation.resume(User("openXu"))
}
fun main() {
    //调用getUser()函数，传入一个续体的匿名类对象作为参数
    getUser(object:Continuation<User>{
        override fun resume(user: User) {
            //2. getUser()挂起点之后的代码
            println("请求结果$user")
            ...
        }
    })
}
```

### 2.2 续体传递风格CPS

给一个普通函数加上`suspend`修饰符，函数被编译后会自动增加一个类型为`Continuation`的参数。**每个挂起函数在被编译后都会附加一个`Continuation`类型的参数，在调用时隐式传入续体对象(在挂起函数中可通过续体对象的resume系列函数返回成功值或者异常来恢复协程的执行)。这就是CPS(Continuation-Passing-Style:续体传递风格)。**

比如`getUser()`挂起函数的声明是这样的：

```kotlin
suspend fun getUser(): User
```

经过CPS变换后，将变为：

```kotlin
fun getUser(continuation: Continuation<User>): Any?
对应的java源码：
Object getUser(Continuation $completion)
```

函数的返回类型`User`被移动到符加的参数续体的泛型位置上，而返回值类型变成了`Any?`(Any相当于java中的Object)。为什么返回值类型变成了Any？我们通过源码来解决这个问题：

```kotlin
//①. 自定义挂起函数，获取一个User对象
suspend fun getUser(): User = suspendCancellableCoroutine{
    continuation ->
    //函数体block：偷个懒省略了线程切换的步骤，直接通过续体对象返回一个User对象恢复协程执行
    continuation.resume(User("openXu"))
}

//②. 这个函数上面已经讲过，用于自定义挂起函数的
public suspend inline fun <T> suspendCancellableCoroutine(
    crossinline block: (CancellableContinuation<T>) -> Unit
): T =
	//调用函数③捕获一个当前协程的续体对象uCont
    suspendCoroutineUninterceptedOrReturn { uCont ->
    	//CancellableContinuationImpl拦截续体对象，成为可取消的续体
        val cancellable = CancellableContinuationImpl(uCont.intercepted(), resumeMode = MODE_CANCELLABLE)
        cancellable.initCancellability()   //初始化续体状态
        //调用block代码块(getUser()函数体)，将续体对象作为参数传入
        block(cancellable)    
        //通过续体获取结果值并返回给suspendCoroutineUninterceptedOrReturn()函数
        cancellable.getResult()
    }

//③. 捕获当前协程的Continuation续体实例
public suspend inline fun <T> suspendCoroutineUninterceptedOrReturn(crossinline block: (Continuation<T>) -> Any?): T
```

上面的3个函数调用顺序从上往下，但是执行顺序是从下到上：首先`suspendCoroutineUninterceptedOrReturn()`捕获了当前协程的续体对象`uCont`，然后`uCont`被`CancellableContinuationImpl`拦截包装为一个可取消的续体`cancellable`，然后将续体作为参数调用block代码块（按道理block会在子线程执行），继续调用续体的`getResult()`获取结果作为③的返回值，③的返回值又是②的返回值，②的返回值又是①.getUser()的返回值，所以getUser()返回值类型是什么就要看`cancellable.getResult()`的返回类型是什么了：

```kotlin
//kotlinx.coroutines.CancellableContinuationImpl.kt

internal fun getResult(): Any? {
    ...
    //★挂起协程，如果挂起成功将返回COROUTINE_SUSPENDED标志，这也是挂起的本质（挂起函数返回COROUTINE_SUSPENDED）
    if (trySuspend()) return COROUTINE_SUSPENDED  
    //如果一个挂起函数（伪挂起）没有挂起协程，协程继续执行下面的代码，直接返回一个结果或者抛出一个异常
    val state = this.state  
    if (resumeMode.isCancellableMode) {
            val job = context[Job]
            if (job != null && !job.isActive) {
            	//如果父协程已经cancel，则抛异常
                val cause = job.getCancellationException()
                cancelCompletedResult(state, cause)
                throw recoverStackTrace(cause, this)
            }
        }
    //返回执行成功的结果值
    return getSuccessfulResult(state)
}

//返回成功值，类型为续体的泛型T，也就是Continuation<User>中的User类型
override fun <T> getSuccessfulResult(state: Any?): T =
    when (state) {
        is CompletedContinuation -> state.result as T
        else -> state as T
    }
```

如果协程被挂起，则返回一个特殊标识`COROUTINE_SUSPENDED`，如果没有挂起协程，协程继续执行时，它直接返回一个结果或者抛出一个异常，也就是说getUser()可能返回两种类型的值，所以返回值类型为Any?。其实如果getUser()是真的挂起函数在执行过程中只会返回`COROUTINE_SUSPENDED`，真实的返回值类型User对象是通过续体回调的（稍后讲协程执行逻辑会讲到）。

都到这会儿了，我们干脆继续看看`COROUTINE_SUSPENDED`是什么东西：

```kotlin
@SinceKotlin("1.3")
public val COROUTINE_SUSPENDED: Any get() = CoroutineSingletons.COROUTINE_SUSPENDED

@SinceKotlin("1.3")
//协程状态枚举：协程被挂起了、未确定状态、协程恢复执行了
internal enum class CoroutineSingletons { COROUTINE_SUSPENDED, UNDECIDED, RESUMED }
```

`COROUTINE_SUSPENDED`是一个枚举类型的值，没有什么具体的含义，仅仅表示协程已经被挂起了，其实它是什么类型根本不重要，重要的是它是单例唯一的，可以作为协程挂起后状态的标志。在协程库1.3版本以前，它就是一个Any对象，大家都称它为白板。

### ★★★2.3 SuspendLambda

我们先了解一下相关的类：

```kotlin
package kotlin.coroutines
/**①. 续体接口：代表协程下一步应该执行的代码*/
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}

//kotlin-stdlib.jar
package kotlin.coroutines.jvm.internal 
/**②. 续体的基本实现类，实现了resumeWith()*/
internal abstract class BaseContinuationImpl(
        public val completion: Continuation<Any?>?
) : Continuation<Any?>,... {
    //实现续体的resumeWith()函数，并且是final的子类不能覆盖
    public final override fun resumeWith(result: Result<Any?>) { ... }
    //★定义了一个invokeSuspend()抽象函数，这个函数的函数体实现就是协程代码块中的代码，由kotlin编译器自动生成实现
    protected abstract fun invokeSuspend(result: Result<Any?>): Any?
}

/**③. 续体实现类，继承自BaseContinuationImpl，增加了拦截器intercepted()功能，实现线程调度等*/
internal abstract class ContinuationImpl(
        completion: Continuation<Any?>?,
        private val _context: CoroutineContext?
) : BaseContinuationImpl(completion) {
      private var intercepted: Continuation<Any?>? = null
	 public fun intercepted(): Continuation<Any?> =
        intercepted?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }
}

/**③. 协程构建器launch{}传入的挂起Lambda表达式的封装抽象类，同时它又是一个续体*/
internal abstract class SuspendLambda(
        public override val arity: Int,
        completion: Continuation<Any?>?
) : ContinuationImpl(completion), FunctionBase<Any?>, SuspendFunction {}
```

下面更加清晰的展示了它们的继承关系：

```xml
Continuation 续体：代表协程下一步应该执行的代码
  |
  |-- BaseContinuationImpl 续体基本实现：主要实现了resumeWith(Result)函数，以控制续体状态机的执行流程，定义了抽象方法invokeSuspend(Result)
        |	
        |-- ContinuationImpl 续体实现：增加了拦截器intercepted()功能，实现线程调度等等
               |
               |-- SuspendLambda 挂起lambda表达式：是对协程代码块中代码的封装
                     |
                     |-- kotlin编译为协程生成的匿名子类：将协程代码块中的代码分为n个状态后作为invokeSuspend(Result)的函数体从而实现该函数
```

`SuspendLambda`非常重要，它是协程的核心，我们所说的续体、状态机、挂起、恢复等都是通过它来实现的。虽然协程库中有一个`AbstractCoroutine`类代表着协程(下篇文章会讲到)，但我更倾向于将`SuspendLambda`看作协程，而`AbstractCoroutine`的主要作用就是保存和传递上下文对象给`SuspendLambda`。

kotlin在编译时会为每个协程生成一个`SuspendLambda`的匿名子类，并创建其对象，我是怎么知道的呢？看看下面的代码：

```kotlin
//Kotlin代码：Suspend.kt
fun main() {
    GlobalScope.launch {
        delay(1000)      
        println("延迟1s")
    }
    Thread.sleep(5000)
}
```
在编译为class后的字节码指令：

```java
//SuspendKt类
public final class runable/SuspendKt {
	//main函数
	public final static main()V
	   L0
	    LINENUMBER 14 L0
	    //从GlobalScope类中获取静态字段INSTANCE
	    GETSTATIC kotlinx/coroutines/GlobalScope.INSTANCE : Lkotlinx/coroutines/GlobalScope;
	    ...
	    //new一个SuspendKt$main$1的对象
	    NEW runable/SuspendKt$main$1
	    ...
	    //调用CoroutineScope的launch()方法，传入的最后一个参数就是上面new的SuspendKt$main$1对象
	    INVOKESTATIC kotlinx/coroutines/BuildersKt.launch$default (Lkotlinx/coroutines/CoroutineScope;Lkotlin/coroutines/CoroutineContext;Lkotlinx/coroutines/CoroutineStart;Lkotlin/jvm/functions/Function2;ILjava/lang/Object;)Lkotlinx/coroutines/Job;
	    POP
    ...
}
/**★ 编译期自动成的类，发现它继承了SuspendLambda，其主要目的就是实现invokeSuspend()函数*/
final class runable/SuspendKt$main$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2 {
    public final invokeSuspend(Ljava/lang/Object;)Ljava/lang/Object;
	...
}
```

kotlin在编译时会为每个协程生成一个`SuspendLambda`的匿名子类，并创建其对象，这个对象有2层含义：

- 对创建协程时传入的挂起Lambda表达式中代码块的封装，将代码块中的代码以调用挂起函数为分割点分为多个部分后填充到`invokeSuspend()`函数中从而实现`SuspendLambda`
- 实现了`Continuation`，本身就是一个续体对象，`resumeWith()`具有恢复协程执行的能力

**每个协程（父子嵌套协程视为多个协程）都会生成一个对应的`SuspendLambda`的匿名子类，这个匿名子类对象就是对协程Lambda代码块的封装，同时又是当前协程的续体对象。**

### ★★★2.4 状态机

续体是对挂起点之后代码块的封装，调用了多少个挂起函数（多少个挂起点）那就需要多少个续体对象。然而协程在实现的时候考虑到性能问题，要尽可能少创建类和对象，比如为一个协程创建的`SuspendLambda`的匿名子类同时充当协程的续体(少创建类)，而一个协程也只会创建一个`SuspendLambda`续体对象（少创建对象）。不是一个挂起点对应一个续体对象吗？如果在协程中调用了多个挂起函数通过一个续体对象是怎么实现的？答案就是通过状态机，说白了就是将多个续体融合到一个`SuspendLambda`对象中，通过**维护一个Int类型的状态值，标记续体执行到哪个步骤了，这个状态值和对应的switch语句就是状态机**（详情看下一步的逻辑讲解）。使用状态机使得无论挂起lambda表达式体内有多少挂起点，编译器也只创建一个`SuspendLambda`子类和对象。

比如下面的示例中创建了一个协程，在挂起Lambda表达式中调用了3个挂起函数，整个lambda表达式就被划分为4个部分，对应着4个状态值，除了初始状态(协程启动后最先需要执行的续体)，一个挂起点之后到下一个挂起点(或者表达式末尾)是一个新的状态：

```kotlin
//Suspend.kt
data class User(val name:String)

fun main() {
    GlobalScope.launch {
        //--------------------初始状态0----------------------
        println("状态0")
        delay(1000)         //挂起点1
        //--------------------状态1----------------------
        println("状态1")
        var user= getUser()  //挂起点2
        //--------------------状态2----------------------
        println("状态2 $user")
        delay(1000)          //挂起点3
        //--------------------状态3----------------------
        println("状态3")
    }
    Thread.sleep(5000)
}

suspend fun getUser(): User = suspendCancellableCoroutine{
    continuation ->
    Thread{
        Thread.sleep(2000)
        continuation.resume(User("openXu"))
    }.start()
}
```

### 2.5 挂起、恢复实现原理源码解读

根据上面对`Continuation`续体和`SuspendLambda`挂起lambda表达式类的讲解，我们知道要搞清楚协程的执行原理，只需要搞懂`resumeWith(Result)`和`invokeSuspend()`函数即可。`invokeSuspend()`函数就是对协程代码块中代码的封装，只是加入了状态机将代码分为多个部分，当协程开始执行时会首先调用一次`invokeSuspend()`函数触发协程代码块初始状态的代码执行，当调用到挂起函数时，挂起函数会返回一个`COROUTINE_SUSPENDED`标志，导致`invokeSuspend()`的renturn停止剩下代码的执行，这就是协程的**非阻塞挂起**。当挂起函数执行完成会调用续体的`resumeWith()`函数以返回函数结果或者异常，而`resumeWith()`中又调用了`invokeSuspend()`根据状态机的状态值恢复执行协程代码块下一个状态的代码，这就是**协程的恢复**。所以**协程的执行实际上就是n+1次（n==协程代码块中调用的挂起函数数量）调用invokeSuspend()，整个流程是通过resumeWith()和invokeSuspend()中的状态机switch来控制的**。

我们借助Android Studio的Show Kotlin Bytecode查看示例中kotlin反编译后的java源码，我将代码执行的顺序通过`☆index`标注出来了，大家根据标注和注释梳理一下执行流程，重点关注`resumeWith()`和`invokeSuspend()`是怎么搭配着干完协程代码块的执行的：

```java
public final class SuspendKt {
    public static final void main() {
        //☆1. 启动协程
        BuildersKt.launch$default((CoroutineScope)GlobalScope.INSTANCE, (CoroutineContext)null, (CoroutineStart)null,
                //创建一个匿名对象，这个匿名类的类型是SuspendKt$main$1 extends SuspendLambda implements Function2
                (Function2)(new Function2((Continuation)null) {
            private CoroutineScope p$;  //协程作用域引用
            Object L$0;
            Object L$1;
            int label;   //★ 状态码，是SuspendLambda匿名子类的成员变量（续体的成员变量）

            //协程的执行就是多次调用invokeSuspend()函数，直到协程代码块中的所有代码执行完毕
            public final Object invokeSuspend(@NotNull Object $result) {
                boolean var5;
                String var8;
                label26: {
                    Object var10000;
                    CoroutineScope $this$launch;
                    User user;
                    Object var6;
                    label25: {
                        var6 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
                        boolean var4;
                        String var7;
                        switch(this.label) {
                            case 0: //☆4. 初始状态0，协程开启后立马执行的代码
                                //--------------------初始状态----------------------
                                //检测结果是否有异常，第一次调用该函数传递的实参是Unit，不是Result.Failure类型，不会抛异常
                                ResultKt.throwOnFailure($result);
                                $this$launch = this.p$;
                                var7 = "状态0";
                                var4 = false;
                                System.out.println(var7);   //☆5. println("状态0")
                                this.L$0 = $this$launch;    //保存当前协程的协程作用域对象
                                this.label = 1;   //初始状态的续体代码已经执行了，将状态值改为1
                                /**
                                 * ☆6. delay(1000)挂起点1： this作为续体传递，方便在未来某一个恢复协程
                                 * delay()函数会return一个OROUTINE_SUSPENDED标志
                                 * 这个if判断var6和OROUTINE_SUSPENDED是相等的，所以这里会直接return使协程停止执行，从而实现挂起
                                 *
                                 * delay()函数会在1s后调用续体(this)的resumeWith(Unit)，而resumeWith()中会触发调用invokeSuspend()从而恢复续体下一个状态下的代码执行
                                 */
                                if (DelayKt.delay(1000L, this) == var6)
                                    return var6;
                                break;
                            case 1:    //☆7. 挂起点1 delay()恢复后，将执行状态码为1的case
                                $this$launch = (CoroutineScope)this.L$0;
                                //每次续体的恢复都会检测上一个挂起点是否执行正常，如果resumeWith一个异常，这里将会throw抛出异常，注意这个异常是在协程所在线程抛出的
                                ResultKt.throwOnFailure($result);
                                break; //跳出switch，执行第二个续体状态的代码
                            case 2:    //☆10. 挂起点2 getUser()恢复
                                $this$launch = (CoroutineScope)this.L$0;
                                ResultKt.throwOnFailure($result);
                                var10000 = $result;
                                break label25; //跳出标签label25的位置
                            case 3:    //☆13. 挂起点3 delay()恢复
                                user = (User)this.L$1;
                                $this$launch = (CoroutineScope)this.L$0;
                                ResultKt.throwOnFailure($result);
                                break label26;   //跳出标签label26的位置
                            default:
                                throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
                        }
                        //--------------------状态1----------------------
                        var7 = "状态1";
                        var4 = false;
                        System.out.println(var7);//☆8. println("状态1")
                        this.L$0 = $this$launch;
                        this.label = 2;   //将状态值改为2
                        //☆9. var user= getUser() 挂起点2：返回挂起标志OROUTINE_SUSPENDED，然后return挂起协程，等待下次rusumeWith恢复协程执行
                        var10000 = runable.SuspendKt.getUser(this);
                        if (var10000 == var6)
                            return var6;
                    }
                    //--------------------状态2----------------------
                    user = (User)var10000;
                    var8 = "状态2 " + user;
                    var5 = false;
                    System.out.println(var8);      //☆11. println("状态2 $user")
                    this.L$0 = $this$launch;
                    this.L$1 = user;
                    this.label = 3;  //状态值改为3
                    //☆12. delay(1000)挂起点3： 返回挂起标志OROUTINE_SUSPENDED，然后return挂起协程，等待下次rusumeWith恢复协程执行
                    if (DelayKt.delay(1000L, this) == var6)
                        return var6;
                }
                //--------------------状态3----------------------
                var8 = "状态3";
                var5 = false;
                System.out.println(var8);   //☆14. println("状态3")
                return Unit.INSTANCE;    //☆15. 协程代码块中所有代码都执行完毕，返回一个Unit.INSTANCE作为协程返回值，resumeWith()拿到这个返回值后会设置协程状态为完成
            }

            //☆3. 创建当前SuspendLambda匿名子类对象，并调用它的invokeSuspend()开始执行初始状态的代码
            public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
                //new一个匿名类对象，也就是当前对象
                Function2 var3 = new <anonymous constructor>(completion);
                var3.p$ = (CoroutineScope)value;
                return var3;
            }
            //☆2. 调用Function2的对象会执行invoke()函数，invoke()是kotlin中的调用操作符重载函数
            public final Object invoke(Object var1, Object var2) {
                //重新创建对象后调用其invokeSuspend()
                return this.create(var1, (Continuation)var2).invokeSuspend(Unit.INSTANCE);
            }

            /**
             * ★★★resumeWith()的实现来自BaseContinuationImpl，SuspendLambda继承了BaseContinuationImpl，
             * 所以挂起函数中调用续体的resumeWith()就是这个方法逻辑，我将这个方法的实现贴在这里，方便查看
             */
		    public final override fun resumeWith(result: Result<Any?>) {
		        var current = this
		        var param = result
                 //这个while true死循环就是个纸老虎，因为当invokeSuspend()中调用到挂起函数时就会return并不会死循环
		        while (true) {   
		            probeCoroutineResumed(current)
		            with(current) {
		                val completion = completion!! 
		                val outcome: Result<Any?> =
		                        try {
		                            //★★★调用invokeSuspend()，执行续体下一个状态码对应的代码（协程恢复执行）
		                            val outcome = invokeSuspend(param)
		                            //如果下一部分代码又调用了挂起函数挂起了协程，直接return，等待挂起函数继续调用续体的resumeWith
		                            if (outcome === COROUTINE_SUSPENDED) return
		                            /**
		                             * 否则当前协程的Lambda表达式代码都执行完毕了，invokeSuspend()最终会返回当前协程的返回值
		                             * 比如示例中只有一个协程，这个协程并没有返回值，所以将返回一个Unit.INSTANCE作为结果，相当于java中的Void
		                             *
		                             * 如果存在嵌套协程，比如：
		                                GlobalScope.launch {   //父协程
		                                    val user = withContext(Dispatchers.IO){  //子协程
		                                        getUser()
		                                    }
		                                    println("获取User $user")
		                                }
		                             * 这种情况下，子协程的SuspendLambda.invokeSuspend()最终将返回getUser()函数的返回值也就是一个User对象
		                             */
		                            //将Lambda表达式的结果值封装到Result对象中
		                            Result.success(outcome)
		                        } catch (exception: Throwable) {
                                    //如果挂起函数resume恢复了异常对象 或者 协程代码块中throw了异常，将异常封装到Result中
		                            Result.failure(exception)  
		                        }
		                releaseIntercepted() // this state machine instance is terminating
                         //completion是BaseContinuationImpl的属性，它是对当前协程对象的引用，在创建SuspendLambda子类对象时传入的AbstractCoroutine子类对象
		                if (completion is BaseContinuationImpl) {   //这种情况我还没遇到过
		                    current = completion
		                    param = outcome
		                } else {  
                            //completion一般都是AbstractCoroutine的子类对象，所以这里将调用AbstractCoroutine类的resumeWith(outcome)表示协程执行完毕，做一些收尾工作（比如修改协程状态等）
		                    completion.resumeWith(outcome)
		                    return
		                }
		            }
		        }
		    }

        }), 3, (Object)null);
        Thread.sleep(5000L);
    }
    ...

    //getUser()的代码就是将suspendCoroutineUninterceptedOrReturn()和getUser函数体打平了
    public static final Object getUser(@NotNull Continuation $completion) {
        //拦截续体(也就是上面SuspendLambda的子类对象)，包装为一个可取消的续体
        CancellableContinuationImpl cancellable$iv = new CancellableContinuationImpl(IntrinsicsKt.intercepted($completion), 1);
        cancellable$iv.initCancellability();
        CancellableContinuation continuation = (CancellableContinuation)cancellable$iv;
        int var6 = false;
        //开启子线程执行真正的函数体
        (new Thread((Runnable)(new SuspendKt$getUser$2$1(continuation)))).start();
        //getResult()中会调用trySuspend()并返回COROUTINE_SUSPENDED标志
        Object var10000 = cancellable$iv.getResult();
        if (var10000 == IntrinsicsKt.getCOROUTINE_SUSPENDED())
            DebugProbesKt.probeCoroutineSuspended($completion);
        //getUser()将COROUTINE_SUSPENDED标志返回从而挂起协程（暂停协程代码执行）
        return var10000;  
    }
    //getUser()子线程的Runnable
    final class SuspendKt$getUser$2$1 implements Runnable {
        final CancellableContinuation $continuation;
        public final void run() {
            Thread.sleep(2000L);
            Continuation var1 = (Continuation)this.$continuation;
            User var2 = new User("openXu");
            ...
            //挂起函数getUser()通过续体对象调用resumeWith()返回结果值，再次触发invokeSuspend()恢复协程剩余代码的执行
            var1.resumeWith(Result.constructor-impl(var2)); 
        }
        SuspendKt$getUser$2$1(CancellableContinuation var1) {
            this.$continuation = var1;
        }
    }
}
```

虽然反编译后的java代码比较简单，但是由于代码比较长，可读性较差，我们把上面的代码简化成下面的样子估计就看的更明白了：

```java
final class runable/SuspendKt$main$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2 {
	...
	int label = 0;   //★ 状态码
    //调用续体的resumeWith()会触发invokeSuspend()，所以这里将invokeSuspend()直接换成resumeWith()
    public final void resumeWith(@NotNull Object result) {
        //每次调用都会根据状态值跳转到不同的代码块，从而执行不同的状态代码
        if (label == 0) goto L0
        if (label == 1) goto L1
        if (label == 2) goto L2
        if (label == 3) goto L3
        else throw IllegalStateException()
        L0: {  //--------------------初始状态0----------------------
            println("状态0")
            label = 1
            result = delay(1000, this)  //挂起点1，this作为续体传递
            //挂起协程，通过return跳出方法停止执行来实现
            if (result == COROUTINE_SUSPENDED) return
        }
        L1: {  //--------------------状态1----------------------
            println("状态1")
            label = 2
            result = getUser(this)    //挂起点2，this作为续体传递
            if (result == COROUTINE_SUSPENDED) return
        }
        L2: {  //--------------------状态2----------------------
            Object user = result   //getUser()返回的User对象
            println("状态2 $user")
            label = 3
            result = delay(1000, this)   //挂起点3，this作为续体传递
            if (result == COROUTINE_SUSPENDED) return
        }
        L3: {  //--------------------状态3----------------------
            println("状态3")
            return   //lambda表达式执行完成（续体执行完毕），直接return
        }
    }
	...
}
```

> **思考：**main()函数中调用launch()的时候已经通过`(Function2)(new Function2((Continuation)null)`构造了一个`SuspendKt$main$1 extends SuspendLambda implements Function2`类型的对象，按道理启动协程就直接通过这个对象调用`invokeSuspend()`就完事了，为什么还需要☆2`invoke()`、☆3`create()`这两个函数创建新对象？其实☆2和☆3不一定会执行，可能直接就调用`invokeSuspend()`了，也可以只调用☆3创建新对象后再调用`invokeSuspend()`，这涉及到不同的协程构建方式，用不同的方式构建协程（包括外层协程或者其他子协程）就会有不同的执行流程，如果需要执行☆2或者☆3那就说明第一次创建的对象缺少点什么，需要再次创建一个完整的对象。其实缺的就是构造函数中传递给SuspendLambda的`Continuation`，通过launch()构建协程时，你会发现第一次`new Function2((Continuation)null)`传递的是一个null参数，这个对象被强转为`Function2`就是因为它不完整，还不具备`SuspendLambda`的一些功能，所以需要在后面的某个时段通过`invoke()`或者`create()`重新构建对象传递非null的Continuation类型实参。在下一篇文章中会有相应的讲解


### 2.6 嵌套协程执行逻辑

上面的示例中只有一个协程，目的是简化代码嵌套，更加直观的理解状态机和其执行逻辑。接下来我们看一个带有嵌套子协程的示例。示例中通过`runBlocking{}`开启了一个阻塞的父协程，父协程中只调用了一个挂起函数withContext()，所以它的状态机有两个状态，withContext()会启动一个子协程，子协程也只调用了一个挂起函数`delay(1000)`，所以子协程也只有两个状态：

```kotlin
fun main() {
    runBlocking{
        //--------------------父 初始状态0----------------------
        println("父协程 $this")
        //调用挂起函数withContext()开启子协程
        val result = withContext(Dispatchers.IO){
            //--------------------子 初始状态0----------------------
            println("子协程 $this")
            delay(1000)
            //--------------------子 状态1----------------------
            val str = "ABC"
            println("挂起函数返回值$str")
            str
        }
        //--------------------父 状态1----------------------
        println(result)
    }
}
```

上面说过，kotlin编译后会为每个协程(包括子协程)生成一个`SuspendLambda`的子类对象用于封装协程的挂起Lambda表达式代码，同时这个对象也充当该协程的续体实例，下面是对应的class指令，可以很直观的佐证这个结论：

```java
public final class runable/SuspendKt {
    public final static main()V
            ...
            //创建 挂起Lambda表达式1 的对象
            INVOKESPECIAL runable/SuspendKt$main$1.<init> (Lkotlin/coroutines/Continuation;)V
            CHECKCAST kotlin/jvm/functions/Function2 //检查上面创建的对象并转换为Function2
            //★ 开启父协程（将Function2作为参数传入）
            INVOKESTATIC kotlinx/coroutines/BuildersKt.runBlocking$default (Lkotlin/coroutines/CoroutineContext;Lkotlin/jvm/functions/Function2;ILjava/lang/Object;)Ljava/lang/Object;
            ...

    // 1. 父协程的挂起Lambda表达式封装类 runable/SuspendKt$main$1.class
    final class runable/SuspendKt$main$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2{
        private Lkotlinx/coroutines/CoroutineScope;p$   //父协程的作用域
        public final invokeSuspend(Ljava/lang/Object;)Ljava/lang/Object;
            ...
            //创建 挂起Lambda表达式2 的对象
            INVOKESPECIAL runable/SuspendKt$main$1$result$1.<init> (Lkotlin/coroutines/Continuation;)V
            CHECKCAST kotlin/jvm/functions/Function2
            //★ 调用withContext()开启子协程（将Function2作为参数传入）
            INVOKESTATIC kotlinx/coroutines/BuildersKt.withContext(Lkotlin/coroutines/CoroutineContext;Lkotlin/jvm/functions/Function2;Lkotlin/coroutines/Continuation;)Ljava/lang/Object;
    }
    // 2. 子协程的挂起Lambda表达式封装类 runable/SuspendKt$main$1$result$1.class
    final class runable/SuspendKt$main$1$result$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2{
        private Lkotlinx/coroutines/CoroutineScope;p$   //子协程的作用域
        public final invokeSuspend(Ljava/lang/Object;)Ljava/lang/Object;
    }
}
```

查看反编译后的java代码就是为了理清`invokeSuspend()`方法中代码的执行顺序，但是代码比较长可读性差，这里就不粘贴反编译的java代码了，而是尝试自己对照kotlin源码和class指令将`SuspendLambda`手动反编译出来。这个看起来比较复杂但是只要弄清楚状态机就一点也不难了，大家可以自己编写一个示例后做这个尝试，如果能写出来那你就可以通过java语言来实现协程了。

下面是手动反编译的思路和执行流程：

- 两个协程就对应两个SuspendLambda的子类，因为`resumeWith()`的目的是调用`invokeSuspend()`方法，这里直接换成了`resumeWith()`更加直观
- 父协程有两个状态码（0，1），那就对应两个代码块L0和L1，将kotlin源码中初始状态的代码放到L0中，由于调用了`withContext()`挂起函数，状态值lable要+1，withContext()返回了`COROUTINE_SUSPENDED`表示需要挂起协程，所以直接return挂起，停止执行父协程余下的代码
- withContext()开启的子协程和父协程一样也是两个状态，将子协程初始状态代码放到L0中，L0调用了delay()挂起了子协程，状态码+1并return；1s后恢复执行，此时状态码为1则执行L1代码块，返回了字符串“ABC”作为withContext()的返回值恢复父协程的执行
- 父协程恢复后发现父状态码为1，需要执行L1代码块，L1中打印了withContext的结果值，由于父协程整改表达式都执行完了，直接return
- 需要注意的是所有的挂起函数调用都将this作为续体参数传入

```java
//1 父协程的SuspendLambda封装
final class runable/SuspendKt$main$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2 {
	...
	int label = 0;   //★ 父状态码
    public final void resumeWith(@NotNull Object result) {  //main()所在的主线程执行
        if (label == 0) goto L0
        if (label == 1) goto L1
        else throw IllegalStateException()
        L0: {  //--------------------父 初始状态0----------------------
            println("父协程 $this")
            label = 1
            result = withContext(Dispatchers.IO, this){}  //挂起点1
            //挂起协程，return
            if (result == COROUTINE_SUSPENDED) return
        }
        L1: {  //--------------------状态1----------------------
           println(result)
           return   //lambda表达式执行完成（续体执行完毕），直接return
        }
    }
	...
}

//2 子协程的SuspendLambda封装
final class runable/SuspendKt$main$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2 {
	...
	int label = 0;   //★ 子状态码
    public final void resumeWith(@NotNull Object result) {  //Dispatchers.IO子线程中执行
        if (label == 0) goto L0
        if (label == 1) goto L1
        else throw IllegalStateException()
        L0: {  //--------------------子 初始状态0----------------------
            println("子协程 $this")
            label = 1
            result = delay(1000, this) //挂起点1
            if (result == COROUTINE_SUSPENDED) return  //挂起协程，return
        }
        L1: {  //--------------------状态1----------------------
           val str = "ABC"
           println("挂起函数返回值$str")
           return str       //返回结果值
        }
    }
	...
}
```

## 3. 总结

这篇文章讲解了应该怎样编写自己的挂起函数，并通过查看反编译后的源码讲解了协程代码块的运行机制，弄清楚了挂起和恢复的本质，**协程的return式非阻塞挂起和resumeWith()恢复的交替保证了协程代码块中代码的顺序执行，所以在协程代码块中可以将异步逻辑编写为同步的上下顺序结构**。

> 挂起函数就是在一个函数中调用协程库提供的顶层挂起函数使得函数的返回值为`COROUTINE_SUSPENDED`标志(也可以手动使一个函数返回COROUTINE_SUSPENDED从而实现挂起，但是这样的话就没办法恢复了，因为代码编写阶段没办法拿到续体对象)，从而实现return式的挂起，而真正的函数返回值是通过续体`resumeWith()`回调的。协程真正的挂起点就是一个挂起函数返回了`COROUTINE_SUSPENDED`标志。

但是遗留了两个问题:

- 协程的执行就是n+1次调用`invokeSuspend()`函数执行对应状态下的续体代码，而第一次调用该函数(执行初始状态代码块)肯定就是协程启动了，那第一次invokeSuspend()是什么时候触发的？

- 协程挂起Lmabda表达式代码是在协程当前线程执行的，当调用挂起函数时，挂起函数是怎样通过上下文自动切到子线程执行的？当挂起函数执行完后在子线程中调用续体的resumeWith()，为什么恢复协程执行后又回到了协程所在的线程？

这两个问题分别通过接下来的两篇文章解答







