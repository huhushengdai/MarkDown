## 也许你需要LiveData

——有没有觉得子线程做数据处理，然后切换主线程做UI更新很麻烦？

——有没有activity、fragment生命周期在destroy之后，子线程才处理完数据的情况？此时多一个判断，根据生命周期情况来更新UI，不过分吧？

——听说你用了RxJava，既优雅的处理了线程切换问题，又解决了生命周期问题，那么有没有觉得每次都自己绑定RxJava生命周期很麻烦？业务代码和View显示代码有没有因为被耦合到一起了？

……

或许还有很多开发的问题，虽然不难解决，但是很繁琐。**这种时候不妨试试LiveData**。
*(PS.别问LiveData是什么，问就只能往下看)*

引子——

在Android开发中，为了避免ANR，通常耗时的操作（网络请求、数据库操作）都会在子线程中执行，执行完成后，以接口回调的方式去传递数据。这种情况下就比较容易出现一种内存泄漏的情况——内部类和外部模块的引用。具体什么情况，可以边看代码边说明：

```
public class HttpActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_http);
        HttpUtils.loadInfo(new CallBack() {
            //CallBack内部类创建时，会隐式的持有外部类的引用
            //如果activity在finish()后，这个内部类还没有执行完（或者说没有被释放）
            //会导致activity对象也无法释放，无法被gc回收（还有其他对象，持有activity的引用）
            //造成内存泄漏
            @Override
            public void onSuccess(String info) {
                //切换到主线程，更新UI
                runOnUiThread(() -> {
                    //进行更新UI
                });
            }
        });
    }
}

//demo http工具类
public class HttpUtils {
    public static void loadInfo(CallBack callBack){
        //耗时操作，需要开启子线程去执行
        AsyncTask.execute(() -> {
            //...
            //经历了非常耗时操作
            callBack.onSuccess("获取到数据");
        });
    }
}
```

上面代码进行了几个个简单操作：

1.开启一个子线程执行耗时操作，获取数据

2.获取数据通过CallBack进行接口回调，把数据传回给activity

3.activity获取到数据后，切换到主线程更新UI

以上3个操作，应该算是Android开发中最常见的情况之一了，异步获取数据，然后回调，切换到主线程中更新。然而就这么简单的事，很容易造成内存泄漏（代码注释中已大概说明）。

**虽然看起来很容易造成内存泄漏，但实际影响可能并没有想象中的大：一个原因是当子线程执行完之后，会释放callBack对象，也就是释放了activity的引用，使得activity也可以被gc回收了（当然如果因为某些原因，线程没有执行完，会导致activity一直得不到释放）；另一个原因是activity对象，不会在短时间内，被大量创建**

这只是一般情况，如果子线程阻塞了，怎么办？或者获取数据回调时，activity、fragment生命周期已经destroy了，怎么办？

LiveData，就是来解决这些问题的。

先来看看使用LIveData是什么情况，再看看LiveData是如何避免这些情况发生：

```
public class HttpActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_http);
        HttpUtils.loadInfo().observe(this, new Observer<String>() {
            //画重点，这里也是内部类，同样会隐式持有activity的引用
            //那么会造成内存泄漏么？
            @Override
            public void onChanged(@Nullable String s) {
                //进行更新UI
                //不需要切换到主线程
            }
        });
    }
}

public class HttpUtils {
    public static LiveData<String> loadInfo(){
        MutableLiveData<String> data = new MutableLiveData<>();
        //耗时操作，需要开启子线程去执行
        AsyncTask.execute(() -> {
            //...
            //经历了非常耗时操作
            data.postValue("获取到数据");
        });
        //这里虽然返回了LiveData
        //但是并不一定能获取到数据
        return data;
    }
}
```

先来看下具体做了些什么：

1.activity调用loadInfo()方法返回了一个LiveData<String>

2.activity通过LiveData.observe()方法，提供了一个接口回调对象Observer，观察是否有获取到数据

3.在AsyncTask.execute()中，执行耗时操作，获取到数据后，调用了LiveData.postValue()方法

4.activity在onChanged()方法中，更新UI

画两个重点：

1.在获取到LiveData对象时，并不一定立刻获取到数据，当LiveData.postValue()时，才会获取到数据，回调在Observer.onChanged()方法中，同时这里不需要切换到主线程。

2.new Observer()时，这也是个内部类，同样持有activity引用，但是**没有内存泄漏隐患、没有内存泄漏隐患、没有内存泄漏隐患**

```
* If the owner moves to the {@link Lifecycle.State#DESTROYED} state, the observer will
* automatically be removed.
```

源码中有这么一段说明：当Lifecycle.State变成DESTROYED时，observer将会被自动移除。

顾名思义：就是当activity的生命周期变成destroy时，LiveData会释放observer对象，是的observer对象可以被回收，同时activity对象也可以被回收了。

很方便吧，不用顾虑内存泄漏问题，当然，这仅仅是其中一点，还有其他方便的地方

接下来就介绍一下LiveData的几个优点：

**1.避免内存泄漏**

原因和栗子，都在上面了

**2.自动切换到主线程**

作为Google的亲儿子，自然会解决一个主要问题：更新UI时必须在主线程中。所以onChanged()方法是被切换到主线程中执行的。

```
liveData.setValue("可以在主线程中调用");
liveData.postValue("可以在子线程中调用，然后切换到主线程中通知");
```

LiveData提供了两个通知方法，setValue和postValue：

①.setValue方法调用后，是立即通知观察者，数据发生了改变。必须在主线程调用的，如果在子线程中调用，会报出`IllegalStateException: Cannot invoke setValue on a background thread`异常

②.postValue。这个就是可以在子线程调用，调用之后，会切换到主线程，然后再通知观察者数据发生改变

**3.根据生命周期变化，动态获取数据**

```
* The observer will only receive events if the owner is in {@link Lifecycle.State#STARTED}
* or {@link Lifecycle.State#RESUMED} state (active).
```

源码说明：只有在STARTED或者RESUMED状态，observer才会接收到数据发生改变的通知。

试想这么一个场景：activity处于onPause或者onStop状态，此时有十几个数据变化的通知过来，我们是不是得更新十几次UI，但是此时频繁更新UI对我们来说没有意义，使用者看不见此时的activity。

**所以，只有在STARTED或者RESUMED状态，observer才会接收到数据改变通知很有必要，可以节省很多资源（不需要去处理那么多次不需要的UI更新）**

在onPause或者onStop时，LiveData即使postValue()十几次，observer都不会接收到通知，只有当activity回到STARTED或者RESUMED状态，才会获取到最后一次数据变更的通知

这里还有一个优点，在调用LiveData.postValue()方法之后，才调用LiveData.observer()，可以立即获取到数据，不需要重新去执行获取数据的操作。

**4.Transformations（转换）**。Transformations提供了两种转换LiveData的方式

①.Transformations.map(source,func)。有时获取到的数据类型，并不是观察者所直接需要的，需要进行一个转换。

```
public LiveData<Order> getOrder() {
        //耗时操作，去加载获取的信息
        LiveData<Goods> goodsData = getGoods();
        //把Goods 转换成Order
        return Transformations.map(goodsData, new Function<Goods, Order>() {
            @Override
            public Order apply(Goods input) {
                Order order = new Order();
                order.goodsName = input.name;
                order.profit = input.salePrice - input.buyPrice;
                return order;
            }
        });
    }
```

上面举例情况是：

(1).观察者需要Order数据

(2).并没有直接获取Order的方法，但是由Goods数据运算得到的Order

(3).有加载`LiveData<Goods>`的方法

所以要获取`LiveData<Order>`，就是通过Transformations.map()方法，把Goods转换成Order返回给观察者。这里稍微可以注意的是`LiveData<Goods> goodsData = getGoods();`只是获取到LiveData，可能数据还并没有立即获得（别忘了LiveData的机制）。

解释一下Transformations方法：

**Transformations.map(source，func）是一个转换方法，source是源数据（被转换的数据），func是一个回调接口，source数据发生变化时，func把source的数据转换成Transformations.map（）的返回类型**

②Transformations.switchMap()。想下这么一个情景：第一个请求返回的数据，是第二个请求的参数，而第二个请求返回的数据，才是真正所需要的。（有没有人会想，在activity监听到数据后，再去启动第二个请求？）

```
public LiveData<User> queryUser(long userId){
    MutableLiveData userData =  new MutableLiveData<>();
    //开启子线程执行任务
    AsyncTask.execute(() -> {
            //...中间根据userId去做查询User
        });
    return userData;
}

public LiveData<User> getUser(){
    //第一个请求，去获取用户id
    LiveData<Long> idData = getUserId();
    return Transformations.switchMap(idData, new Function<Long, LiveData<User>>() {
        @Override
        public LiveData<User> apply(Long input) {
            //当获取到用户id后，根据用户id进行第二个请求
            return queryUser(input);
        }
    });
}
```

LiveData的优点和基本用法，已经大致说完了，是否觉得非常简短？越是精简，越是显得Google的牛逼，把具体实现都隐藏了，提供了非常简单的使用方法，让开发者把精力集中在具体业务开发上，完美诠释了开闭原则。Google如此厉害的操作，深深值得我们学习，所以接下来就再聊聊LiveData的设计和实现。

**LiveData的设计中心思想很简单，主要就是使用观察者模式。**

observe观察LiveData的数据变化，LiveData数据发生变化，就通知给observe；

LiveData去观察LifecycleOwner的生命周期变化，当生命周期到DESTROYED时，移除observe；

```
//LiveData observe 方法
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        //生命周期已经结束，不需要添加进去
        return;
    }
    LifecycleBoundObserver  wrapper = new LifecycleBoundObserver(owner, observer);
    //mObservers 是一个map集合，这里用来存放observer这个观察者
    //当LiveData数据有变化时，就会遍历mObservers，通知observer
    ObserverWrapper existing = mObservers .putIfAbsent(observer, wrapper);
    //一个observer只能被加入到一个LifecycleOwner中
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    //加入LifecycleOwner的观察队列
    //当生命周期到DESTROYED时，LifecycleBoundObserver会把自身从mObservers集合中移除
    owner.getLifecycle().addObserver(wrapper);
}
```

LiveData的介绍到这里结束了，记住LiveData的两个特点，会在开发中最常用到的：
**1.自动切换到主线程**

**2.跟生命周期绑定，自动解绑，不需要开发者做解绑处理（也就是不用担心内存泄漏）**

LiveData优点有很多，可以解决很多开发遇到的问题。如果配合Google推出的另一个亲儿子ViewModel，会解决更多问题，想知道更多，更具体的么？——Coming soon