## LiveData解决内存泄漏的神奇组件

Activity，Android四大组件之一，主要用于显示界面的，在Android中是使用最多的组件，没有之一。但是，这样一个最常用的组件，会比较容易出现一个问题——内存泄漏。

举个例子，以MVP为例：

![图1](E:\doc\MarkDown\LiveData1.png)

如图，当第一步View（Activity）去请求数据后，在获取数据之前就关闭了Activity，但是此时Activity的内存还并不会被回收，因为Activity的引用还被Presenter还持有，要等待获取数据后更新UI的回调，如果此时拉取数据非常耗时的话，就会造成Activity的内存无法释放，导致了内存泄漏。

内存泄漏可以**粗糙、粗糙、粗糙**的总结为一句话：**当对象需要释放时候，还有地方有持有该对象的引用**（*由于造成内存细节，涉及GC回收机制，比较复杂，所以不在这里详述，有兴趣的朋友可以自己去查阅相关资料*）

*PS.有些同学会不大关注内存泄漏问题，毕竟大多数情况不会有太多影响——比如造成oom，导致app崩掉。但是...老生常谈的“但是”就不说那么多了。*

针对这类的内存泄漏，解决方案也比较多：

**1.使用弱引用。**presenter使用弱引用的方式，去持有View(Activity)的对象，然后在获取到数据回调View的时候，判断View（Activity）对象是否还存在。

**2.通知其持有者去释放。**当Activity生命周期发生变化时候，去通知Presenter，释放Activity的引用。

这里要介绍的LiveData，就是第二种实现思想。

**LiveData实现的核心思想就是与获取数据方（Activity）相互观察。**当获取数据方（Activity）需要数据时，LiveData就是被观察者，当数据发生改变时，就去通知Activity；同时，LiveData也是观察者，观察Activity的生命周期变化，**当LiveData观察到activity的生命周期结束时，LiveData会释放Activity的持有**，同时，也会取消观察activity（即通知Activity释放自己的所引）。也就避免了Activity内存泄漏的可能性。

避免Activity泄漏的方式很多，为什么这里会特别介绍LiveData的呢？这里就来说说LiveData的优点：

**1.兼容性好。**作为Google推出的组件，自然会优解决兼容性问题，在Android SDK 28及其以上的版本中，Activity、Fragment都已经实现了LifecycleOwner接口，使用LiveData时，就不需要去关心Activity、Fragment等生命周期变化问题；如果项目SDK版本还不能升级上去的，也没关系，只需要在自家项目中的BaseActivity、BaseFragment中，实现LifecycleOwner，并且在onCreate、onDestroy等生命周期的地方，调用一下通知方法即可。当然，要是自家项目中没有建BaseActivity、BaseFragment这种基础类......

**2.使用简单。**LiveData已经封装非常完善，使用者只需要关注获取到数据，不需要关心生命周期变化了要如何处理。比如：

```
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    loadData().observe(this, new Observer<String>() {
        @Override
        public void onChanged(@Nullable String s) {
            //这里获取到数据，更新UI
        }
    });
}

public LiveData<String> loadData(){
   final MutableLiveData<String> liveData = new MutableLiveData<>();
    //异步任务
    AsyncTask.execute(new Runnable() {
        @Override
        public void run() {
            //耗时的加载数据操作
            liveData.postValue("获取到的数据");
        }
    });
    return liveData;
}
```

上面代码中，可以看出，跟LiveData相关的只有两个地方：

①.在onCreate（）中，观察数据变化

②.在loadData（）方法中，获取到数据了，然后调用postValue方法通知数据发生改变

这其中可以看出，并不需要关心其他花里胡哨的东西，让我们可以更专注于业务。

**3.可以自动切换到主线程。**作为Google的亲儿子，自然会解决一个主要问题：更新UI时必须在主线程中。所以onChanged()方法是被切换到主线程中执行的。

```
liveData.setValue("可以在主线程中调用");
liveData.postValue("可以在子线程中调用，然后切换到主线程中通知");
```

LiveData提供了两个通知方法，setValue和postValue：

①.setValue方法调用后，是立即通知观察者，数据发生了改变。必须在主线程调用的，如果在子线程中调用，会报出`IllegalStateException: Cannot invoke setValue on a background thread`异常

②.postValue。这个就是可以在子线程调用，调用之后，会切换到主线程，然后再通知观察者数据发生改变

**4.黏性设计**。通常的观察者模式，如果在已经通知数据发生改变之后，再注册的话，是不会收到数据改变的通知。但是LiveData有些不同，**LiveData通知数据更新后，才注册观察LiveData的，也可以获取到最新一次数据发生变化的通知**。这个设计可以很好的避免了重复请求数据的操作（*PS.如果结合Google的另一个组件ViewModel使用，效果更佳*）

看到这里是否已经意识到了LiveData不仅仅只是处理内存泄漏的问题，还有其他很实用的功能。

