### 作用：

跨线程通信。

例如子线程获取数据，切换到主线程更新UI。

### 过程：

1.handler 把消息（执行任务）加入消息队列

2.Looper一个一个的从消息队列中读取消息，进行执行

### 注意点：

1.handler创建实例时，需要传入一个Looper实例，如果没有传，会默认获取当前线程的Looper

2.Looper创建实例是当前线程调用Looper.prepare();方法创建的，创建后会通过ThreadLocal set进当前线程中

实际上就是，每一个线程只能有一个Looper，并且需要通过Looper.prepare()创建了才会有，所以如果要做子线程的Handler，必须在子线程中调用Looper.prepare(HandlerThread 不需要，因为该类内部已经调用过了)

3.Looper实例被创建后，会调用Looper.loop()方法去循环（死循环）遍历MessageQueue队列，读取到消息后执行。同事MessageQueue队列是native层阻塞式的，队列中没有消息时，会进入休眠，释放CPU资源，不会让Looper因为死循环，一直浪费cpu资源。