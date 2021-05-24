> 转载请标明出处： 
https://openxu.blog.csdn.net/article/details/113999613
本文出自:[【openXu的博客】](http://blog.csdn.net/xmxkf)


## 问题背景

项目中使用LiveEventBus时遇到一个非常隐蔽的问题，花了很多时间才定位到原因，我觉得这个场景使用情况还是很多的，在此记录该过程。

首先介绍一下使用场景，app主页面采用TabHost+Fragment构成下方的4个Tab，其中第一个Tab的Fragment中采用了TabLayout+ViewPager+Fragment，可以看到首页中有综合、电、水等Fragment。

![](01-image\scene.jpg)

首页的每个子Fragment在请求统计数据之前，需要先选择上方的Spinner作为参数，而Spinner的数据也是从服务器获取的。其中电Fragment需要获取“行业分类”和“用电类别”，水、气、热等需要获取“行业分类”，为了避免在每个子Fragment中都请求相同的“行业分类”，干脆把获取Spinner数据的请求放在首页HomeFragment中，这样就只需要请求一次数据，当HomeFragment获取到Spinner的数据后，其子Fragment通过`LiveEventBus.get(xxx).observeSticky(owner, observer)`黏性订阅来获取HomeFragment请求的数据。

**HomeFragmentVM**

HomeFragment的ViewModule获取数据后通过LiveEventBus发送事件：

```java
public class HomeFragmentVM extends BaseViewModel {

    public HomeFragmentVM(@NonNull Application application) {
        super(application);
    }

    /*行业列表*/
    public void getIndustryType(Map<String, String> params) {
        NetworkManager.getInstance().newBuilder()
                .method(NetworkManager.Method.GET)
                .viewModel(this)
                .url(ServerApi.EM_INDUSTRYTYPE_GETLIST)
                .putParams(params)
                .build(new ResponseCallback(){
                    @Override
                    public void onSuccess(String msg, FpcDataSource data) throws Exception{
                        ArrayList<PublicIndustryType> publicIndustryTypes = ParseNetData.parseData(data.getTables().get(0), PublicIndustryType.class);
                        LiveEventBus.get(LiveEventKey.IndustryTypeList)
                                .post(publicIndustryTypes);
                    }
                });
    }

    /*用电类型*/
    public void getElectricityType(Map<String, String> params) {
        NetworkManager.getInstance().newBuilder()
                .method(NetworkManager.Method.GET)
                .viewModel(this)
                .url(ServerApi.CMDS_DICTIONARYITEM_GETLIST)
                .putParams(params)
                .build(new ResponseCallback(){
                    @Override
                    public void onSuccess(String msg, FpcDataSource data) throws Exception{
                        ArrayList<PublicElectricityType> publicElectricityTypes = ParseNetData.parseData(data.getTables().get(0), PublicElectricityType.class);
                        LiveEventBus.get(LiveEventKey.ElectricityTypeList)
                                .post(publicElectricityTypes);
                    }
                });
    }
}
```

**子Fragment**

电Fragment中粘性订阅“行业类别”和“用电类型”，其他水、气、热等Fragment中只需要订阅“行业类别”，而不需要“用电类型”

```java
//所有子Fragment中共同订阅 行业类别
LiveEventBus.get(LiveEventKey.IndustryTypeList)
        .observeSticky(this, o->{
            if(o==null){
                binding.pieChart.setData(null);
                binding.pieChart.setCenterText(null);
                return;
            }
            ArrayList<PublicIndustryType> list = (ArrayList<PublicIndustryType>)o;
            publicIndustryTypes.add(new PublicIndustryType("", "所有行业"));
            ((Spinner.MySpinnerAdapter)spinnerIndustryTypes.getAdapter()).setDataList(publicIndustryTypes);
        });
//只有电的Fragment中订阅 用电类型
LiveEventBus.get(LiveEventKey.ElectricityTypeList)
        .observeSticky(this, o->{
            if(o==null){
                binding.pieChart.setData(null);
                binding.pieChart.setCenterText(null);
                return;
            }
            ArrayList<PublicElectricityType> list = (ArrayList<PublicElectricityType>)o;
            publicElectricityTypes.add(new PublicElectricityType("", "所有用电类别"));
            publicElectricityTypes.addAll(list);
            ((Spinner.MySpinnerAdapter)spinnerElectricityTypes.getAdapter()).setDataList(publicElectricityTypes);
        });
```


## 问题

现在出现的问题是，应用程序启动后，第一次切换到电Fragment时，可以收到“行业类型”和“用电类别”，当切换到其他页面（导致电Fragment被移除）再次切回电时，就只能收到“行业分类”。

这个问题是在项目完成后优化阶段出现的，导致的原因是优化了ViewPager+Fragment，使用了懒加载，避免ViewPager的预加载机制导致应用程序初次加载过多数据（造成程序加载数据时间过程和内存浪费的问题），只有当子Fragment对用户可见时才去请求数据。但是这并不会直接导致`LiveEventBus`粘性事件只能部分接收到一次的问题，下面我们一步步按项目进展流程分析。

## ViewPager缓存所有页面

项目最开始阶段为了赶进度，没有设计懒加载机制，并且直接让ViewPager缓存所有页面`binding!!.viewpager.offscreenPageLimit = tabs.size`，当HomeFragment被加载后，就会初始化其所有的子Fragment，并且完成数据请求，这一阶段没有出现上述问题，但是留下了隐患，就是应用程序第一次启动会比较慢，因为请求的数据太多，就会看到加载框转了很久，所以客户不满意了

## ViewPager默认缓存

这个方案其实跟缓存所有页面的性值是一样的，只是一次最多只需要缓存3个子Fragment(ViewPager默认的offscreenPageLimit==1)，但是还是提前加载了可能用不到的数据，没从根本上解决问题，所以继续

## Fragment懒加载

在基类Fragment中设计懒加载机制，只有当Fragment真正对用户可见（可交互）时才去加载数据。

```java
public abstract class BaseLazyFragment<V extends ViewDataBinding, VM extends BaseViewModel>
        extends Fragment  {
    protected V binding;
    protected VM viewModel;
    private boolean isViewCreated = false;   //View是否创建完毕
    private boolean isFirstVisib = true;     //是否第一次可见
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        FLog.v(getClass().getSimpleName()+"-->onCreateView() "+binding);
        if(binding==null)
            initViewDataBinding(inflater, container, savedInstanceState);
        return binding.getRoot();
    }
    @Override
    public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        initView();
        registObserve();
        isViewCreated = true;
        //补充调用setUserVisibleHint(true),完成数据加载
        if(getUserVisibleHint())
            setUserVisibleHint(true);
    }
    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {
        super.setUserVisibleHint(isVisibleToUser);
        lazyLoad();
    }
    private void lazyLoad(){
        if(getUserVisibleHint() && isFirstVisib && isViewCreated){
            initData(isFirstVisib);
            isFirstVisib = false;
        }
    }
    /**
     * 懒加载数据
     * @param isFirstVisib 是否是第一次对用户可见
     */
    protected void initData(boolean isFirstVisib) {
    }
    @Override
    public void onDestroy() {
        super.onDestroy();
        getLifecycle().removeObserver(viewModel);
        if (binding != null)
            binding.unbind();
    }

    ...
}
```

让所有子Fragment继承BaseLazyFragment，只有当ViewPager真正选中的Fragment才会加载数据。同时这个BaseLazyFragment是可以被当成普通Fragment使用的，当不需要懒加载时，只需要在onResume()中编写加载数据的代码，当需要懒加载时重写`initData(boolean isFirstVisib)`，并且参数isFirstVisib标识了是否是第一次可见，比如当Fragment已经可见并且加载数据了，这时候跳转到另外的页面，再次返回让Fragment可见时还会调用initData，但是isFirstVisib的值是false，我们可以选择性的去处理。当然我的项目中只需要加载一次，所以只有当isFirstVisib为true时才加载。

同时这个懒加载机制有一点点不完善的地方，就是没有加入当Fragment由可见 到不可见的时候应该阻断数据加载，由于我的项目中这种场景需求性不强就不弄那么复杂了。

## ViewPager+FragmentPagerAdapter+懒加载

其实在加入懒加载后，并没有出现那个问题，原因是HomeFragment中的ViewPager设置的Adapter是`FragmentPagerAdapter`。

`FragmentPagerAdapter`的`destroyItem()`方法中在销毁Fragment时，只是将Fragment从Activity上detach()，并没有真正销毁回收Fragment对象（不会回调`onDestroy()`方法），当再次切换该Fragment时，会直接复用并attach该Fragment对象。

这个方案没有出现问题的根本原因是每个子Fragment只会被初始化一次，并且只会加载一次数据，后面ViewPager切换时都是切换的缓存的Fragment对象。但是考虑到首页Fragment是比较多的，所以为了节省内存，应该使用`FragmentStatePagerAdapter`

## ViewPager+FragmentStatePagerAdapter+懒加载

`FragmentStatePagerAdapter`的`destroyItem()`方法中在销毁Fragment时直接`mCurTransaction.remove(fragment)`将fragment对象移除了，这回导致fragment被彻底销毁并回收，但是它会保存fragment的状态，方便数据恢复。

采用这种组合模式会导致ViewPager切换时，电Fragment被移除（比如选中气Fragment，这时候只会缓存水、气、热这3个Fragment）后，再次切换到电Fragment时，会重新构建一个全新的Fragment对象，这时候就会再走一遍生命周期方法和懒加载方法，这时候问题出现了，发现只能监听到“行业类型”的粘性事件，而不能监听到“用电类型”。

## 问题根本原因以及解决方案

刚开始出现这个问题时，怀疑跟Fragment销毁和重建有关，但是被蒙蔽了，两个Spinner获取数据的代码逻辑是完全一样的，为什么有一个可以再次监听到粘性事件？另一个就只能第一次能监听到呢？怀疑过LiveEventBus框架本生的问题，但最终都不了了之，做了大量测试和日志对比，并没有找到问题根本原因。但是发现将`LiveEventBus.get(LiveEventKey.ElectricityTypeList).observeSticky()`换成`observeStickyForever()`就能再次监听到“用电类型”，说明要一直监听才能再次收到？可是当Fragment重建时又是一个全新的观察者，跟之前的有什么关系？

认真阅读查看了[LiveEventBus](https://github.com/JeremyLiao/LiveEventBus)的使用说明，也没有发现使用上的错误。 也考虑过电Fragment和其他Fragment的区别，其中“行业类型”是在每个子Fragment中都有注册观察者，而“用电类型”只在电Fragment中注册了观察者，但是这又能说明什么问题呢？直到读到[LiveEventBus的配置](https://github.com/JeremyLiao/LiveEventBus/blob/master/docs/config.md)，其中有一个配置**autoClear**，用于配置在没有Observer关联的时候是否自动清除LiveEvent以释放内存（默认值false），突然一下就恍然大悟了。

为什么“用电类型”只能监听到第一次？第一次是肯定能被监听到的，当电Fragment中监听粘性事件时就会收到数据，但是当电Fragment被销毁后，这个事件的Observer会因为生命周期感知而被移除，所以“用电类型”这个事件就没有Observer了，而我在Application中配置了`autoClear`为true，所以这个事件就会被清除；而当再次切换到电Fragment时，即使再去订阅这个事件，这个事件是一个全新的事件，并且数据是空的，所以就监听不到数据了。

为什么“行业类别”可以每次切换都监听到？因为“行业类别”这个事件始终都有Observer观察着（电水气热Fragment中都监听了它），所以这个事件永远不会被清除。

### 方案探索

使用`FragmentPagerAdapter`代替`FragmentStatePagerAdapter`，让电Fragment对象不会被彻底销毁，这样第一次监听到“用电类型”后就被缓存到Fragment中了，切换时都是复用缓存的fragment对象。

但是我还是想用`FragmentStatePagerAdapter`，那就把LiveEventBus框架的`autoClear`配置为false，这样当电Fragment被销毁时，事件并不会因为没有Observer而被清除。但是`autoClear`还是设置为true比较好，随着层级页面打开的越多，如果事件不被清除，被销毁的页面的数据会占用大量内存。

那怎样让“用电类型”事件的Observer不被移除呢？那就只能将`LiveEventBus.get(LiveEventKey.ElectricityTypeList).observeSticky()`换成`observeStickyForever()`，这样的话 `FragmentElectric` 在需要被销毁时并不能被真正回收，造成内存泄漏，因为FragmentElectric中创建了匿名内部类Observer实例去订阅事件，而内部类持有外部类的引用，Observer不能随着生命周期感知取消订阅就会永驻内存，这样FragmentElectric也就不能被回收。通过`FLog.i("订阅日志 "+Console.getInfo());`在控制台打印日志如下，看发现随着`FragmentElectric`的多次切换会导致key为`ElectricityTypeList`事件的Observer数量不停增长。

```xml
*********Base info*********
    lifecycleObserverAlwaysActive: true
    autoClear: true
    logger enable: true
    logger: com.jeremyliao.liveeventbus.logger.DefaultLogger@960ad9c
    Receiver register: true
    Application: com.hk.operator.HkApplication@41a0e88
    *********Event info*********
    Event name: ElectricityTypeList
    	version: 0
    	hasActiveObservers: true
    	hasObservers: true
    	ActiveCount: 3
    	ObserverCount: 3
    	Observers: 
    		[com.jeremyliao.liveeventbus.core.LiveEventBusCore$ObserverWrapper@a5c3d11=androidx.lifecycle.LiveData$AlwaysActiveObserver@9ab237a, com.jeremyliao.liveeventbus.core.LiveEventBusCore$ObserverWrapper@e3b536a=androidx.lifecycle.LiveData$AlwaysActiveObserver@cce4684, com.jeremyliao.liveeventbus.core.LiveEventBusCore$ObserverWrapper@4782537=androidx.lifecycle.LiveData$AlwaysActiveObserver@5712d8f]
    Event name: IndustryTypeList
    	version: 0
    	hasActiveObservers: true
    	hasObservers: true
    	ActiveCount: 1
    	ObserverCount: 1
    	Observers: 
    		[com.jeremyliao.liveeventbus.core.LiveEventBusCore$ObserverWrapper@3db36f8=androidx.lifecycle.ExternalLiveData$ExternalLifecycleBoundObserver@6a1fe5b]
```

为了解决内存泄漏问题，LiveEventBus文档中说明了在使用Forever模式订阅消息时，需要调用removeObserver取消订阅。于是我将Observer作为成员变量，并在FragmentElectric的onDestroy()方法中取消订阅，结果还是跟以前一样，问题又出现了，这不就跟生命周期感知订阅一回事吗？

```java
    @Override
    public void onDestroy() {
        super.onDestroy();
        LiveEventBus.get(LiveEventKey.ElectricityTypeList).removeObserver(observer);
    }
```

那应该在什么时机去释放掉之前的Observer呢？只能在下次FragmentElectric重新构建后订阅事件之后再取消之前的Observer对象，那问题就来了，我应该怎样在下一个FragmentElectric对象中拿到上一个FragmentElectric对象中的Observer对象呢？有一种办法就是将这个Observer对象作为一个全局对象，当然这种方式就不太优雅了，而且内存还是泄漏了，只是有回收的机会，并且不会一直泄漏：

```java
Observer observer = new Observer<Object>() {
    @Override
    public void onChanged(Object o) {
        if(o==null){
            binding.pieChart.setData(null);
            binding.pieChart.setCenterText(null);
            return;
        }
        ArrayList<PublicElectricityType> list = (ArrayList<PublicElectricityType>)o;
        publicElectricityTypes.add(new PublicElectricityType("", "所有用电类别"));
        publicElectricityTypes.addAll(list);
        FLog.i(this+"收到用电类型列表：" + publicElectricityTypes.size());
        ((Spinner.MySpinnerAdapter)spinnerElectricityTypes.getAdapter()).setDataList(publicElectricityTypes);
    }
};
//新的订阅
LiveEventBus.get(LiveEventKey.ElectricityTypeList)
        .observeStickyForever(observer);
//取消之前的订阅
if(HomeFragment.observer!=null)
    LiveEventBus.get(LiveEventKey.ElectricityTypeList).removeObserver(HomeFragment.observer);
//将observer作为全局静态变量
HomeFragment.observer = observer;
```

### 最终方案

目前采用的方案是HomeFragment中使用ViewPager+FragmentPagerAdapter，避免Fragment被真正销毁，这样做的话就能避免LiveEventBus因为生命周期感知（onDestroty）自动销毁Observer，也就不会出现事件因为没有Observer订阅被清除的情况。订阅时还是采用生命周期感知订阅

```java

LiveEventBus.get(LiveEventKey.ElectricityTypeList)
    .observeSticky(this, observer);
```

而且这样做还能避免因为频繁切换ViewPager导致的数据多次加载，利用空间换时间，目前Fragment也不算太多，可以接受。

























