参考：

[扔物线（朱凯）Kotlin 的协程用力瞥一眼](https://kaixue.io/kotlin-coroutines-1/)
[Kotlin协程设计提案](https://github.com/Kotlin-zh/KEEP/blob/master/proposals/coroutines.md)
[Kotlin协程指南](https://www.kotlincn.net/docs/reference/coroutines/coroutines-guide.html)
[Kotlin协程文档](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/index.html)
[Android上的Kotlin协程](https://developer.android.google.cn/kotlin/coroutines)




> 版权声明：本文为openXu原创文章[【openXu的博客】](http://blog.csdn.net/xmxkf)，未经博主允许不得以任何形式转载

@[toc]


公司项目中使用协程已经有一段时间了，刚开始接触协程的时候也和其他做Android的小伙伴有相同的心理活动：Google爸爸，球球你不要再出新东西了，学不动了。不管当初我是怎样拒绝它的，但是现在我只能说：嗯  协程真香！接触协程的时候在网上搜索了一些文章，有的文章是很早以前的，估计作者写文章的时候也并没有真正理解协程，还有一些文章通俗易懂但是不够深入。所以笔者打算通过几篇文章来记录本人通过kotlin源码深入协程的过程，希望对想了解协程的人有所帮助。

# 1. 什么是协程

虽然对于一些人来说比如刚开始的我，**协程(Coroutines)**是一个新的概念，但是协程这个术语早在1958年就被提出并用于构建汇编程序，协程是一种编程思想，并不局限于特定的语言，就像Rx也是一种思想，并不局限于使用Java实现的RxJava。不同语言实现的协程库可能名称或者使用上有所不同，但它们的设计思想是有相似之处的。

`kotlinx.coroutines`是由JetBrains开发的kotlin协程库，可以把它简单的理解为一个**线程框架**。但是协程不是线程，也不是新版本的线程，它是基于线程封装的一套更上层工具库，我们可以使用kotlin协程库提供的api方便的灵活的指定协程中代码执行的线程、切换线程，但是不需要接触线程Thread类。说到这里，大家可能就会想到Android的`AsyncTask`或者RxJava的`Schedulers`，没错，从某种意义上来说它们和协程是相通的，都解决了异步线程切换的问题，然而**协程最重要的是通过非阻塞挂起和恢复实现了异步代码的同步编写方式，把原本运行在不同线程的代码写在一个代码块{}里，看起来就像是同步代码。**

```kotlin
MainScope().launch(){     //在UI线程中启动协程
    //下面的代码看起来会以同步的方式一行行执行（异步代码同步获取结果）
    val token = apiService.getToken()   // 网络请求：IO线程，获取用户token
    val user = apiService.getUser(token)// 网络请求：IO线程，获取用户信息
    nameTv.text = user.name             // 更新 UI：主线程，展示用户名
    val articles = apiService.getUserArticle(user.id)// 网络请求：IO线程，根据用户id获取用户写的文章
    articleTv.text = "用户${user.name}总共写了${articles.size}篇文章"   // 更新 UI：主线程
}
```

协程并不是从操作系统层面创立的新的运行方式，代码是运行在线程中的，线程又是运行在进程中的，协程也是运行在线程中的，所以才说它是基于线程封装的库。然而有人会拿协程与线程比较，问协程是不是比线程效率更高？如果理解了协程是基于线程封装就应该知道，协程并没有改变代码运行在线程中的原则，单线程中的协程执行时间并不会比不用协程少，它们之间没有可比性，因为它们根本不属于同一类事物；协程也不是为了线程而生的，它是为了解决因为多线程带来的编码上的不便的问题而出现的。

然而说到这里估计有些人还是云里雾里的，下面还是通过庸俗的示例来丝滑的认识协程

# 2. 协程有什么作用

在Android开发中，通常会将耗时操作放到子线程中，然后通过回调的方式将结果返回后切换主线程更新UI，但是实际开发过程中可能遇到很多奇怪而合理的需求，它们可能是：

1. 一个页面需要同时并发请求多个接口，当所有接口都请求完成需要做一些合并处理然后更新UI

- 按照惯例，我们可能会为每个接口请求设置一个boolean标志，每当一个接口请求完将对应的boolean值改为true，当最后一个接口请求完成发现所有标志都为true再更新UI，这样就能达到并发请求的目的，然而管理这么多boolean值累不累？优雅不优雅？
- 初级程序员可能干脆来个单线程，一个接口请求完成后，再请求另一个接口，直到最后一个接口返回数据，玩暴力美学啊，本来能同时干的事情非得一件件干，你让用户浪费他宝贵的时间合适吗？浪费用户高配的性能过瘾吗？
- 高级一点的可能就上RxJava了，通过RxJava的zip操作符，达到发射一次，将结果合并处理的目的，但是说实话到现在还有很多人不会用RxJava

2. 先调用接口1获取数据，然后拿到接口1的结果作为参数调用接口2，然后将接口2的数据展示出来

- 按照惯例，我们可能会调用接口1，然后在接口1的回调中获取数据再嵌套调用接口2
- 高级一点的可能就上RxJava了

......都是一样的套路，没有一点新花样吗？/(ㄒoㄒ)/~~还真没有，那就以第2个需求为栗子继续吧

## 2.1 回调地狱

需求：页面上需要展示文章列表，而获取文章列表之前要先获取到文章树结构，拿到树下的第一个分支id，然后通过分支id获取分支下的第一页文章作为默认展示内容。

通过Retrofit请求的逻辑如下：

> 此处借用[wanandroid开放的接口](https://www.wanandroid.com/blog/show/2)，只是简单的实现这个业务逻辑，对框架封装并不完善，比如错误处理等等，只需要领会嵌套回调即可

```kotlin
/**RetrofitClient单例*/
object RetrofitClient {
    /**log**/
    private val logger = HttpLoggingInterceptor.Logger {
        FLog.i(this::class.simpleName, it)
    }
    private val logInterceptor = HttpLoggingInterceptor(logger).apply {
        level = HttpLoggingInterceptor.Level.BODY
    }

    /**OkhttpClient*/
    private val okHttpClient = OkHttpClient.Builder()
            .callTimeout(10, TimeUnit.SECONDS)
            .addNetworkInterceptor(logInterceptor)
            .build()
    /**Retrofit*/
    private val retrofit = Retrofit.Builder()
        .client(okHttpClient)
        .baseUrl(ApiService.BASE_URL)
        .addCallAdapterFactory(RxJava3CallAdapterFactory.create())
        .addConverterFactory(GsonConverterFactory.create())
        .build()
    /**ApiService*/
    val apiService: ApiService = retrofit.create(ApiService::class.java)
}

/**接口定义*/
interface ApiService {
	companion object {
        const val BASE_URL = "https://www.wanandroid.com"
    }
    /*获取文章树结构*/
    @GET("tree/json")
    fun getTree(): Call<ApiResult<MutableList<Tree>>>

    /*根据数结构下某个分支id，获取分支下的文章*/
    @GET("article/list/{page}/json")
    fun getArticleList(
            @Path("page") page: Int,
            @Query("cid") cid: Int
    ): Call<ApiResult<Pagination<Article>>>
}

/**ViewModel*/
class SystemViewModel : BaseViewModel(){
    private val remoteRepository : SystemRemoteRepository by lazy { SystemRemoteRepository() }

    val page = MutableLiveData<Pagination<Article>>()

    fun getArticleList() {
       remoteRepository.getArticleList(){
           page.value = it
       }
   }
}

/**数据仓库*/
class SystemRemoteRepository{
	/**
	 * 1. 展示回调嵌套，回调地狱
	 */
	fun getArticleList(responseBack: (result: Pagination<Article>?) -> Unit) {
	        /**1. 获取文章树结构*/
	        val call:Call<ApiResult<MutableList<Tree>>> = RetrofitClient.apiService.getTree()
	        //同步（需要自己手动切换线程）
	        //val response : Response<ApiResult<MutableList<Tree>>> = call.execute()
	        //异步回调
	        call.enqueue(object : Callback<ApiResult<MutableList<Tree>>> {
	            override fun onFailure(call: Call<ApiResult<MutableList<Tree>>>, t: Throwable) {
	            }
	            override fun onResponse(call: Call<ApiResult<MutableList<Tree>>>, response: Response<ApiResult<MutableList<Tree>>>) {
	                FLog.v("请求文章树结构成功："+response.body())

	                /**2. 获取分支id下的第一页文章*/
	                val treeid = response.body()?.data?.get(0)?.id
	                //当treeid不为null执行
	                treeid?.let {
	                    RetrofitClient.apiService.getArticleList(0, treeid)
	                            .enqueue(object : Callback<ApiResult<Pagination<Article>>> {
	                                override fun onFailure(call: Call<ApiResult<Pagination<Article>>>, t: Throwable) {
	                                }
	                                override fun onResponse(call: Call<ApiResult<Pagination<Article>>>, response: Response<ApiResult<Pagination<Article>>>) {
	                                    //返回获取的文章列表
	                                    responseBack(response.body()?.data)
	                                }
	                            })
	                }

	            }
	        })
	    }
}
```

其实上面数据仓库中的方法是应该拆分为2个方法的，第二个接口请求拆分为方法后还可以复用（分页获取），这里仅仅为了展示嵌套回调的需求。可以看到仅仅是两层回调嵌套，可读性就已经很差了，然而这种需求并非不常见的甚至会出现更多层嵌套，这时候就会写出**深>形**的代码，非常不雅观而且不易于代码复用及后期维护。

有人会说将上面的嵌套回调拆分为2个方法，在第一个接口请求完成之后再调用另一个方法请求文章列表，这不就消除了嵌套回调了吗？从视觉上来说确实是消除了，但是从逻辑上来说嵌套依然存在，而且这种方式会让两个方法之间形成很强的业务关联，对代码维护带来的挑战不比嵌套回调小。代码就不展示了，就是简单的将两个请求拆分。

## 2.2 Rx解决回调地狱

还有人说，使用Retrofit+RxJava组合通过Rx的链式调用就能消除嵌套回调：

```kotlin
interface ApiService {
	...
    /**RxJava方式*/
    @GET("tree/json")
    fun getTreeByRx(): Observable<ApiResult<MutableList<Tree>>>

    @GET("article/list/{page}/json")
    fun getArticleListByRx(
        @Path("page") page: Int,
        @Query("cid") cid: Int
    ): Observable<ApiResult<Pagination<Article>>>
}

class SystemRemoteRepository{
	/**
	 * 2. Retrofit+RxJava消除回调嵌套
	 */
	fun getArticleListByRx(responseBack: (result: Pagination<Article>?) -> Unit) {
	    /**1. 获取文章树结构*/
	    val observable1: Observable<ApiResult<MutableList<Tree>>> = RetrofitClient.apiService.getTreeByRx()
	    observable1.subscribeOn(Schedulers.io())
	            .observeOn(AndroidSchedulers.mainThread())
	            //使用当前Observable发出的值调用给定的Consumer，然后将其转发给下游
	            .doOnNext({
	                FLog.v("1请求文章树成功，切换到主线程处理数据：${Thread.currentThread()}")
	            })
	            .observeOn(Schedulers.io())
	            .flatMap({
	                FLog.v("2请求文章树成功，IO线程中获取接口1的数据，然后将被观察者变换为接口2的Observable：${Thread.currentThread()}")
	                if(it?.errorCode == 0){
	                    //当treeid不为null执行
	                    it?.data?.get(0)?.id?.let { it1 -> RetrofitClient.apiService.getArticleListByRx(0, it1) }
	                }else{
	                    //请求错误的情况，发射一个Error
	                    Observable.error({
	                        Throwable("获取文章树失败：${it.errorCode}:${it.errorMsg}")
	                    })
	                }
	            })
	            .observeOn(AndroidSchedulers.mainThread())
	            .subscribe(object: Observer<ApiResult<Pagination<Article>>> {
	                override fun onComplete() {}
	                override fun onSubscribe(d: Disposable?) {}
	                override fun onNext(t: ApiResult<Pagination<Article>>?) {
	                    FLog.v("3请求文章列表成功：${t?.data}")
	                    responseBack(t?.data)
	                }
	                override fun onError(e: Throwable?) {
	                    FLog.e("3请求失败：${e?.message}")
	                }
	            })
	}
}
```

Retrofit+RxJava确实消除了回调的嵌套，但是还是避免不了回调（Observer观察者可看作是回调），链式调用处理异步数据流确实比传统的嵌套回调好了太多，但是代码量不减反增，而且我们需要在正确的位置准确的插入不同的操作符用来处理异步数据，对于不熟悉Rx的同学来说也是很头痛的。**虽然链式的处理异步数据真的很优雅（堪称艺术），但是对于大部分编码者来说可能称不上痛快。**

**怎样才痛快？怎样才痛快？怎样才痛快？**

## 2.3 同步调用

我调用接口方法请求数据，方法直接给我返回数据，我不关心线程切换，拿到数据就展示，这才符合常人思维，才痛快：

```kotlin
/**
 * 此处调用RetrofitClient.apiService的接口返回的是Retrofit的Call实例，调用Call.execute()方法同步请求数据
 */
fun getArticleList() {
    //同步调用接口1，直接得到返回数据
    val response1 : Response<ApiResult<MutableList<Tree>>> = RetrofitClient.apiService.getTree().execute()
    val cid = response1.body()?.data?.get(0)?.id
    if(cid!=null){
        //同步调用接口2，直接得到返回数据
        val response2 : Response<ApiResult<Pagination<Article>>> = RetrofitClient.apiService.getArticleList(0, cid).execute()
        page.value = response2.body()?.data!!
    }
}

/**接口定义*/
interface ApiService {
	companion object {
        const val BASE_URL = "https://www.wanandroid.com"
    }
    /*获取文章树结构*/
    @GET("tree/json")
    fun getTree(): Call<ApiResult<MutableList<Tree>>>

    /*根据数结构下某个分支id，获取分支下的文章*/
    @GET("article/list/{page}/json")
    fun getArticleList(
            @Path("page") page: Int,
            @Query("cid") cid: Int
    ): Call<ApiResult<Pagination<Article>>>
}
```

看看上面的`getArticleList()`方法，使用同步的方式请求接口，直接拿到返回数据，爽不爽？很爽，你们个个都是人才，异步是不可能异步的，线程切换又不会，只有这样写才能维持得了生活的这样子...。好吧，运行项目：bong!!!

```xml
2021-03-05 13:43:07.078 12508-12508/com.openxu.android E/AndroidRuntime: FATAL EXCEPTION: main
    Process: com.openxu.android, PID: 12508
    java.lang.RuntimeException: Unable to resume activity {com.openxu.android/com.openxu.android.MainActivity}: android.os.NetworkOnMainThreadException
    ...
```

我相信这种同步调用的写法可以说是所有开发者梦寐以求的，然而Android不允许在主线程中耗时等待，我们只能忍气吞声的处理各种回调，kotlin协程库的出现终于让它成为现实。

## 2.4 协程--用同步的方式编写异步代码

以往我们编写异步代码，需要关注：

- 子线程什么时候执行结束？因为结束之后需要在对应的回调中获取数据
- 子线程中的数据怎样发送到主线程？用传统Handler？还是各种事件总线如EventBus？LiveData(抱歉，LiveData并不支持线程切换，只能在UI线程中setValue)？
- 多个线程的管理问题

使用协程就可以让我们摆脱因为多线程带来的各种编码上的不便：

```kotlin
class SystemViewModel : BaseViewModel(){
    private val remoteRepository : SystemRemoteRepository by lazy { SystemRemoteRepository() }
    val page = MutableLiveData<Pagination<Article>>()

    fun getArticleList() {
        viewModelScope.launch {  //主线程开启一个协程
        	// 网络请求：IO线程(需要注意的是OkHttp请求的部分是子线程，Retrofit部分是主线程，这部分内容请看后续关于挂起函数的讲解)
            val tree : ApiResult<MutableList<Tree>> = RetrofitClient.apiService.getTreeByCoroutines()
            // 主线程
            val cid = tree?.data?.get(0)?.id
            if(cid!=null){
            	// 网络请求：IO线程
                val pageResult : ApiResult<Pagination<Article>> = RetrofitClient.apiService.getArticleListByCoroutines(0, cid)
                // 主线程
                page.value = pageResult.data!!
            }
        }
    }
}

/**接口定义，Retrofit从2.6.0版本开始支持协程*/
interface ApiService {
	/*获取文章树结构*/
	@GET("tree/json")
	suspend fun getTreeByCoroutines(): ApiResult<MutableList<Tree>>
	/*根据数结构下某个分支id，获取分支下的文章*/
	@GET("article/list/{page}/json")
	suspend fun getArticleListByCoroutines(
	    @Path("page") page: Int,
	    @Query("cid") cid: Int
	): ApiResult<Pagination<Article>>
}
```

运行上面的代码，尽然成功了，刚刚发生了什么？这不就是同步调用吗？跟上面的同步有什么不一样吗？看起来差不多啊，确实差不多，就是在定义接口时，方法前加了个`suspend`关键字，调用接口的时候用`viewModelScope.launch {}`包裹。既然可以运行成功，就说明请求接口并不是在主线程中进行的，然而有的同学不信，他在`getArticleList()`方法中的任意位置通过`Thread.currentThread()`打印的结果都是`Thread[main,5,main]`，这不就是在主线程中调用的吗？上述协程中的代码是在主线程执行没错，但是接口请求的方法却是在子线程中执行的。

原因就在于我们定义接口的时候使用了`suspend`关键字，它的意思是挂起、暂停，函数被加了这个标记就称它为挂起函数，需要注意的是，`suspend`关键字并没有其他重要的作用，它仅仅标识某个函数是挂起函数，可以在函数中调用其他挂起函数，但是只能在协程中调用它。所以上面两个接口都被定义为挂起函数了，挂起函数只能在协程中调用，那谁是协程？

其实在kotlin协程库中是有一个类`AbstractCoroutine`来表示协程的，这个抽象类有很多子类代表不同的协程，但是这些子类都是private的，并没有暴露给我们，所以你在其他文章中看到别人说`viewModelScope.launch{}`包裹起来的闭包(代码块)就是协程也没问题，但这个代码块的真正意义是协程需要执行的代码。当在协程中调用到挂起函数时，协程就会在当前线程（主线程）中被挂起，这就是协程中著名的**非阻塞式挂起**，主线程暂时停止执行这个协程中剩余的代码，注意：暂停并不是阻塞等待（否则会ANR），而是**主线程暂时从这个协程中被释放出来去处理其他Handler消息**，比如响应用户操作、绘制View等等。

那挂起函数谁执行？这得看挂起函数内部是否有切换线程，如果没有切换线程当然就是主线程执行了，所以**挂起函数不一定就是在子线程中执行的**，但是通常在定义挂起函数时都会为它指定其他线程，这样挂起才有意义。比如上面定义的suspend的请求接口，Retorift在执行请求的时候就切换到了子线程并挂起主线程，当请求完成（挂起函数执行完毕）返回结果时，会通知主线程：我该干的都干完了，下面的事你接着干吧，主线程接到通知后就会拿到挂起函数返回的结果继续执行协程里面剩余的代码，这叫做**协程恢复(resume)**。如果又遇到挂起函数就会重复这个过程，直到协程中的代码被执行完。

通过协程可以彻底去除回调，**使用同步的方式编写异步代码**。什么是同步调用？调用一个方法能直接拿到方法的返回值，尽管这个方法是耗时的、在其他线程执行的，也能直接得到它的返回值，然后再执行下面的代码，协程不是通过等待的方式实现同步，而是通过非阻塞挂起实现看起来同步的效果。

## 2.5 协程只是为了消灭回调？

协程爽吗？很爽，但是协程就是为了解决嵌套回调吗？并非如此，消灭回调只是利用协程实现的一个很小的部分 。其实上面第一部分讲协程是什么的时候就已经说了，协程是为了让我们**更方便的写并发代码，把原本运行在不同线程的代码写在一个代码块{}里，看起来就像是同步代码。**，它的侧重点是**多线程并发**和**看起来同步**。所以对于串行（上述嵌套）或者并行（多个接口并行请求）的代码都可以用协程来"同步"调用的，**不管业务有多复杂，都可以用协程写出清晰的上下行代码结构**。


# 3. kotlin协程出现的原因

上面的示例演示了使用协程解决回调地狱、自由切换线程、大大减少代码量等优势，协程使用起来真的很爽，这些都是体现在使用层面的。但是kotlin协程没有出现时，代码也照样能跑的啊，为什么kotlin非得搞协程出来呢？就是为了增强我们：google爸爸，我们学不动了 的感觉吗？很显然这不是本意，kotlin相对于java的一方面优势就是简洁，而各种回调嵌套显然是kotlin不能容忍的，再者kotlin支持高阶函数这就为kotlin协程设计带来了便利，有了这些原由支撑，kotlin协程就诞生了。

概括起来其实就是使用kotlin协程让代码更简洁。根本原因是**解决了异步任务非阻塞同步返回数据的问题，达到简化代码逻辑、所见即所得的目的**。所以我们要先搞懂什么是异步任务、以及异步任务数据返回

**异步计算**：这个不用做多介绍了吧，异步是多线程的概念，说简单点就是将一部分计算放到另外的线程执行。比如Android中主线程(UI线程)是为了响应绘制、用户交互的，如果一些耗时任务(网络请求、数据库操作等)让主线程执行就会造成UI卡顿，所以将这些耗时操作放到其他线程执行，相对于主线程来说，这些耗时任务就是异步计算的。

通常情况下，我们执行一个异步任务(一个方法)是为了获取一个计算结果，异步计算是怎么返回结果数据的呢？有以下几种方式：

## 3.1 CallBack异步回调

```koltin
/**耗时获取一个整数的平方值*/
fun square(input:Int):Int{
    println("耗时任务在子线程 ${Thread.currentThread()}")
    Thread.sleep(2000)
    return input * input
}

//模拟一个耗时的异步（子线程）任务，通过回调获取结果
fun asynCallBack(onComplete : (result:Int)->Unit){
    Thread {
        val result = square(10)
        onComplete(result)
    }.start()
}
fun main() {
    println("主线程：${Thread.currentThread()}")
    //调用异步计算方法，传入回调函数获取计算结果
    asynCallBack (onComplete = {
        println("${Thread.currentThread()}异步耗时任务计算结果 $it")
    })

    Thread.sleep(5000)  //主线程休眠5s避免jvm退出
}
```

执行结果：

```xml
主线程：Thread[main,5,main]
耗时任务在子线程：Thread[Thread-0,5,main]
Thread[Thread-0,5,main]异步耗时任务计算结果 100

Process finished with exit code 0
```

上面的示例中，main函数所在线程是不会等待异步线程执行完成的，它所做的就是调用了asynCallBack这个函数，并在这个函数中开启了一个子线程，主线程的任务就完成了，如果不通过`Thread.sleep(5000)`让主线程休眠，是不会观察到任何结果的，因为主线程执行完jvm就退出了。

这个示例就展示了异步任务通过回调的方式获取计算结果，异步计算的结果并没有返回到主线程中，如果是在Android中，通常需要使用`Activity.runOnUiThread()`将结果转移回UI线程展示。

## 3.2 Future阻塞获取异步计算结果

严格的说来，通过回调的方式并不能看作是异步计算返回执行结果，因为返回结果通常看起来是`val result = asynCallBack()`这样的，但是Java中通过Thread没有办法获取异步任务执行结果，在JDK5中引入了`Future`，Future表示异步计算的结果，提供了`cancel()`取消任务、`get()`阻塞等待获取结果等方法，相信很多人对它都不了解，因为在Android开发中基本不会接触到这个类，因为它是线程阻塞的。下面使用Future改写示例：

```kotlin
fun asynFuture() : Future<Int> {
    //将一个Callable对象提交到线程池中，返回一个Future对象表示异步计算结果
    return Executors.newSingleThreadExecutor()
            .submit(Callable<Int> {
                square(10)
            })
}
fun main() {
    println("主线程：${Thread.currentThread()}")
    val future = asynFuture()
    //阻塞等待获取异步计算结果
    val result = future.get()
    //异步计算执行完毕才会执行下面的代码，主线程就像是被阻塞劫持了
    println("阻塞主线程等待异步计算执行结果：$result")
}
```

执行结果：

```kotlin
主线程：Thread[main,5,main]
耗时任务在子线程：Thread[pool-1-thread-1,5,main]
阻塞主线程等待异步计算执行结果：100
```

Future通常用于计算密集、处理大量数据等场景，可以直接通过它获取异步计算的结果，这跟kotlin协程看起来差不多了，最关键的区别是它是线程阻塞等待的，如果是在UI应用程序(比如Android)中，阻塞UI线程线程是不被允许的，所以Future对于我们来说并不常用

## 3.3 协程非阻塞挂起

```kotlin
suspend fun asynCoroutine() : Int = withContext(Dispatchers.IO){
    square(10)
}

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    println("主线程：${Thread.currentThread()}")
    //开启协程，将异步耗时任务包裹在协程执行体代码块中
    CoroutineScope(Dispatchers.Main).launch{
        // 1. 异步执行挂起，挂起是非阻塞的，会释放主线程去做别的事情
        val result = asynCoroutine()
        // 2. 异步执行完继续执行，更新UI
        tvResult.text = "异步任务执行结果：$result"
    }
}
```

这个示例是在Android Activity环境中，这样更好的理解挂起函数的非阻塞挂起。`asynCoroutine()`是一个挂起函数，这个函数将在子线程中被调度，在UI线程中开启一个协程，并在协程代码块中调用挂起函数(注释1)，挂起函数之后的代码将停止执行，直到挂起函数执行完毕，这看起来就像是在阻塞等待挂起函数执行结果，但是并没有阻塞主线程，主线程虽然没有继续执行挂起函数之后的代码，但是去干了别的事情（比如绘制UI、响应用户交互）。调用挂起函数就像是在调用处打了个标记，告诉主线程我要去干一件大事可能要很长时间，你先去忙别的事情吧，我执行完再通知你。当挂起函数执行完，就会拿着执行结果通知主线程恢复继续执行标记点之后的代码。这就是协程的非阻塞挂起，也就是说协程可以**实现非阻塞获取异步计算结果**，这对于各种UI环境开发来说简直是太好用了，因为可以**用看起来同步的方式编写异步代码**


# 4. 总结

这篇文章只是带大家初步认识了协程，简单了解kotlin协程的来由以及其优势，关于协程的使用和其原理请看接下来的几篇文章，喜欢的朋友加个关注点个赞，一键三联比个心🤞








## suspend
## 取消与超时
## 组合挂起函数
## 协程上下文与调度器
## 异步流
## 通道
## 异常处理与监督
## 共享的可变状态与并发
## Select表达时

okhttp+协程，拦截器抛出的异常无法捕获的问题

https://stackoverflow.com/questions/58697459/handle-exceptions-thrown-by-a-custom-okhttp-interceptor-in-kotlin-coroutines

okhttp两种添加拦截器方法的区别

//    Okhttp 的addInterceptor 和 addNetworkInterceptor 的区别？
//    addInterceptor（应用拦截器）：
//    1，不需要担心中间过程的响应,如重定向和重试.
//    2，总是只调用一次,即使HTTP响应是从缓存中获取.
//    3，观察应用程序的初衷. 不关心OkHttp注入的头信息如: If-None-Match.
//    4，允许短路而不调用 Chain.proceed(),即中止调用.
//    5，允许重试,使 Chain.proceed()调用多次.
//    addNetworkInterceptor（网络拦截器）：
//    1，能够操作中间过程的响应,如重定向和重试.
//    2，当网络短路而返回缓存响应时不被调用.
//    3，只观察在网络上传输的数据.
//    4，携带请求来访问连接.













