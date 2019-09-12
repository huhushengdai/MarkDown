binder是什么：

1.Android跨进程通信的一种方式

C/S架构

内存映射 mmap函数，内存只复制一次

有pid作为进程标识，提高安全性

binder如何实现跨进程通信：

service 通过binder 驱动，获取ServiceManager的binder，进行把自己名称和binder注册到ServiceManager里面

client 通过binder驱动，获取ServiceManager的binder，然后找到需要的service 的binder

此时client获取到的service binder 并不是实际的service binder对象，而是binder驱动的代理，当client进行操作时，binder驱动会调用实际的service binder 对象进行相应的操作，从而实现了进程间通信



为什么要使用binder这种方式去实现跨进程通信



service如何启动的



activity如何启动