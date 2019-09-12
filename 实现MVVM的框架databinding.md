## 						MVVM框架-DataBinding

​		DataBinding是google 推出的一个实现MVVM模式的框架，帮助开发者从View显示和实际数据的同步中，解放出来。其主要思想是把data和view绑定在一起，当view显示数据发生改变时，实际data也随着改变；同理，当实际data发生改变时，界面View的显示也跟着改变。或许大家此时还有不少疑问：DataBinding是如何实现这些的？这就是我接下来要详细说明的

### 首先-先了解MVC、MVP、MVVM

​		DataBinding是一个实现了MVVM的框架，那么要了解DataBinding之前，就要先了解一下什么是MVVM及其先驱MVP和MVP的先驱-MVC

#### 1.MVC

[MVC](https://baike.baidu.com/item/MVC)全名是Model View Controller，是模型(model)－视图(view)－控制器(controller)的缩写，一种软件设计典范，用一种业务逻辑、数据、界面显示分离的方法组织代码，将业务逻辑聚集到一个部件里面，在改进和个性化定制界面及用户交互的同时，不需要重新编写业务逻辑。MVC被独特的发展起来用于映射传统的输入、处理和输出功能在一个逻辑的图形化用户界面的结构中。——来自百度百科

​    	view-负责界面

​		controller-控制逻辑操作

​		model-负责数据存取

​		view响应到用户事件（比如点击拉取用户信息），把具体的下载逻辑交由control去执行，view自身并不管如何去拉取用户信息；control去获取用户信息（例如从内存、本地数据库、网络等），获取到信息后，交由model，本身不管数据如何处理；model收到数据后，进行相应处理（如存入本地等），然后通知view获取到的信息，并由view更新显示

​		三者各司其职，不需要知道其他模块怎么实现，只需要内聚自己的逻辑即可。这就方便项目维护、扩展等。

#### 2.MVP

​		MVP是MVC的一种衍生，针对MVC的一些不足进行的改变。就如现在的Android，如果需要更新view，通常都需要Activity来进行操作。在经典的MVC模式中，当model要更新view时，就需要获得相应的Activity引用，如此做的话，耦合度会比较深，当项目需要扩展、修改、复用时，涉及的模块会比较广。

​		所以，MVP模式衍生了出来，在MVP模式中，P（Presenter）代替了MVC的control，作为一个主导的存在：当view需要数据时，去通知P，由P进行主导，调度model（此时的model也已经进行升级，不再单单为数据库，而是泛指获取到数据，至于是从缓存、数据库、网络中获取，都是model内部的实现）获取数据，然后再通知view，所获得的数据。

​		此时的Presenter，就是一个居中调度的身份，view需要什么，由P去调度model获取，然后再交由view显示，在这之中view和model已经基本没有联系，当view、和model发生改变时，都不会影响对方，相互之间彻底解耦。

#### 3.MVVM

​		同理，由上可得，MVVM是MVP的一种升级版，不过MVVM不是在模式上进行改进，而是在使用上进行改进。

​		MVP在使用的时候，P获取到数据之后，需要手动去通知View