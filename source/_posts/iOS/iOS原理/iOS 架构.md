---
title: iOS 架构
categories: iOS原理
---


## 一、MVC

> MVC 全名 Model View Controller，是模型(model)－视图(view)－控制器(controller)的缩写，一种软件设计典范，用一种业务逻辑、数据、界面显示分离的方法组织代码，将业务逻辑聚集到一个部件里面，在改进和个性化定制界面及用户交互的同时，不需要重新编写业务逻辑。MVC 被独特的发展起来用于映射传统的输入、处理和输出功能在一个逻辑的图形化用户界面的结构中。

组成 MVC 的三个模式分别是：

1. 组合模式
2. 策咯模式
3. 观察者模式

MVC 在软件开发中发挥的威力，最终离不开这三个模式的默契配合

1. View 层，单独实现了组合模式
2. Model 层和 View 层，实现了观察者模式
3. View 层和 Controller 层，实现了策咯模式

MVC 要实现的目标是将软件用户界面和业务逻辑分离以使代码可扩展性、可复用性、可维护性、灵活性加强.

<center>
![](http://dzliving.com/iOSStructure_0.png)
</center>

#### 1.1 减重 

实际开发 ViewController 往往会非常之重，动不动几百行，几千行代码！那么是一些什么东西在里面？

* 繁重的网络层
* 复杂的 UI 层
* 难受的代理
* 啰嗦的业务逻辑
* 还有一些其他功能

这里建议：

1. 繁重的网络层：封装到业务逻辑管理者比如：present、viewModel
2. 复杂的 UI 层：将 UI 代码直接剥离出 VC
	
	```
	- (void)loadView
	{
		self.homeView = [[HomeView alloc] init];
		self.view     = self.homeView;
	}
	```

3. 难受的代理：可以封装一个功能类比如我们常写的tableview collectionView的代理 我们就可以抽取出来封装为一个公共模块，一些特定的逻辑就可以利用适配器设计模式，根据相应的 model 消息转发


#### 1.2 解耦

推荐使用不直接依赖 model 利用发送消息的方式传递。


## 二、MVP

> MVP 全称 Model-View-Presenter。MVP 是从经典的模式 MVC 演变而来，它们的基本思想有相通的地方：Controller/Presenter 负责逻辑的处理，Model 提供数据，View 负责显示。

MVP 是面向协议编程的思想。


MVP 能够解决：

* 代码思路清晰
* 耦合度降低显著
* 通讯还算比较简单

缺点：

* 需要写很多关于代理相关的代码
* 视图和 Presenter 的交互会过于频繁
* 如果 Presenter 过多地渲染了视图，往往会使得它与特定的视图的联系过于紧密。一旦视图需要变更，那么 Presenter 也需要变更了


## 三、MVVM

> MVVM 全称 Model-View-ViewModel，本质上就是 MVC 的改进版。MVVM 将其中的 View 的状态和行为抽象化，让我们将视图 UI 和业务逻辑分开，放在 ViewModel 中处理，viewModel 可以取出 Model 的数据同时帮忙处理 View 中由于需要展示内容而涉及的业务逻辑。

MVVM 的特色：双向绑定

<center>
![](http://dzliving.com/iOSStructure_1.png)
</center>