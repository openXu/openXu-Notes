
## 总结



## 1. 外观模式(门面模式)

kotlin协程库从协程的配置、创建到启动都是通过`CoroutineScope`协程作用域，我们不需要关心协程是怎样创建的，怎样启动的，只需要配置上下文和启动模式就能开启一个协程，从而隐藏了协程复杂的实现细节。所以`CoroutineScope`就是协程库提供的门面API

```kotlin
    //配置上下文，启动协程
    CoroutineScope(Dispatchers.Main).launch (
            EmptyCoroutineContext,
            CoroutineStart.DEFAULT   //配置启动模式
    ){
        //协程挂起Lambda表达式
    }
```
## 2. 单例模式

`GlobalScope`是一个单例对象：

```kotlin
public object GlobalScope : CoroutineScope {
}
```


## 3. 工厂模式

### 3.1  简单工厂模式CoroutineScope()

`CoroutineScope`接口只暴露了`GlobalScope`这一个子类对象，它是用于构建全局的协程，在实际开发中并不推荐使用（生命周期与应用程序同步，可能造成内存泄漏）。另一个子类`ContextScope`是`internal`修饰的，所以没办法直接创建子类对象，只能通过库提供的简单工厂函数`CoroutineScope()`，不同的协程作用域类型的根本原因是其维护的协程上下文对象不同，而这个简单工厂函数通过传递不同的上下文对象就可以创建无数种作用域实例，一个工厂生产多种产品，所以它是简单工厂模式

```kotlin
//简单工厂函数，一个工厂函数创建各种类型的作用域对象
public fun CoroutineScope(context: CoroutineContext): CoroutineScope =
    ContextScope(if (context[Job] != null) context else context + Job())

```

### 3.2 工厂模式MainScope()

`CoroutineScope()`用于生产各种类型的作用域对象，基本适用所有的场景，但是针对特殊的常用场景，协程库还给我们提供了特殊的作用域工厂，比如：

- `MainScope()`: 用于构建一个调度在UI线程的作用域，适用于带UI线程的场景
- `viewModelScope`: 创建一个生命周期与ViewModel同步的UI线程调度的作用域，适用于Android中的MVVM模式

```kotlin
//用于创建在UI线程调度的作用域工厂
public fun MainScope(): CoroutineScope = ContextScope(SupervisorJob() + Dispatchers.Main)

//用于创建一个在UI线程调度，且生命周期与ViewModel同步的协程作用域对象
public val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        if (scope != null) {
            return scope
        }
        return setTagIfAbsent(
            JOB_KEY,
            CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
        )
    }

internal class CloseableCoroutineScope(context: CoroutineContext) : Closeable, CoroutineScope {
    override val coroutineContext: CoroutineContext = context

    override fun close() {
        coroutineContext.cancel()
    }
}
```

虽然直接通过简单工厂函数`CoroutineScope()`也能创建出以上两种作用域对象，但是每次都需要`ContextScope(Dispatchers.Main)`传递上下文，如果使用的地方多了就比较麻烦，所以**通过工厂模式为常用的场景创建对应的作用域对象，简化作用域的创建。**我们还可以参照`MainScope()`根据项目需求自己扩展作用域工厂函数，比如我想创建调度在主线程、带有具体名称协程的作用域对象：

```kotlin
//ContextScope是internal修饰的，不能直接使用，但是可以借助简单工厂函数CoroutineScope()
//public fun MyScopeFactory(): CoroutineScope = ContextScope(Dispatchers.Main+CoroutineName("openXu"))
//借助简单工厂函数CoroutineScope()实现自定义工厂函数
public fun MyScopeFactory(): CoroutineScope = CoroutineScope(Dispatchers.Main+CoroutineName("openXu"))
```

### 3.3 抽象工厂模式

上面两种工厂是用于创建CoroutineScope协程作用域对象的，


## 4. 拦截器



## 5. Contination 装饰模式  代理模式






















