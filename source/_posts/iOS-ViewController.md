---
title: iOS ViewController
date: 2017-02-01 18:56:10
tags: iOS
categories: iOS开发

---
本篇主要记录iOS中的Controller，简记
<!--more-->

## UIViewController简介
官方文档：  
>The UIViewController class provides the infrastructure for managing the views of your iOS apps. A view controller manages a set of views that make up a portion of your app’s user interface. It is responsible for loading and disposing of those views, for managing interactions with those views, and for coordinating responses with any appropriate data objects. View controllers also coordinate their efforts with other controller objects—including other view controllers—and help manage your app’s overall interface.

用通俗的话来讲，他就是mvc层的c(Controller)，用于View层和Model层的通信，跟Android里面的Activity十分类似，但是比其要灵活不少，貌似还不是很臃肿……

## UIViewController生命周期
主要是这几个部分：  
1. 初始化  
2. 如果是通过其他ViewController通过segue导航而来的话，那就根据segue的要求进行初始化  
3. Outlet设置  
4. viewDidLoad()
5. viewWillAppear()  
6. viewWillLayoutSubviews()  
7. viewDidLayoutSubviews()  
8. (6,7步多次调用之后) viewDidAppear()  
9. (界面离开屏幕)viewWillDisappear()  
10. (6,7步多次调用)viewDidDisappear()  
这几个函数的顺序我就不验证了，懒，值得注意的地方在于第二步和第三步的顺序不能颠倒，也就是说，如果在prepare()阶段通过segue的destination来修改目标Controller的Outlet标签成员的话，会崩溃的。  
## UIViewController的主要职责
**管理views**  
![](http://ok34fi9ya.bkt.clouddn.com/ViewController1.png)
UIViewController有一个view属性，便是上图中的RootView，RootView的子视图也可以交给Controller来进行管理，例如Outlet标签标记的视图。  

**管理子Controller**  
![](http://ok34fi9ya.bkt.clouddn.com/ViewController2.png)
上图中，SplitViewController管理着两个子Controller，分别是master controller和detail controller，这两个Controller的内容是包裹在父Controller的Container里面的（只能在iPad上面看到全部的内容）。  

**处理与用户交互有关的事件**  
Controller可以处理与其管理的views有关的交互事件（触摸，点击，滑动……）实现的方式有两种。一种就是直接从Storyboard按住ctrl拖(既不优雅也没效率)，实现一个标记IAction属性的方法，这种方法通常是用来处理单一的例如Touch up inside的单一事件。另一种实现方式是在管理的view上面添加GestureRecogniser，iOS sdk自己给出的几个recogniser真是太好用了，这里给出一段示例代码：

	- (void)viewDidLoad {
     	[super viewDidLoad];
 
     	// 创建TabGestureRecogniser
     	UITapGestureRecognizer *tapRecognizer = [[UITapGestureRecognizer alloc]
          initWithTarget:self action:@selector(respondToTapGesture:)];
 
     	// 配置TabGestureRecogniser
     	tapRecognizer.numberOfTapsRequired = 1;
 
     	// 为view添加GestureRecogniser
     	[self.view addGestureRecognizer:tapRecognizer];
	}
接下来Controller只需要实现selector里面的方法就可以了，个人感觉这真是太优雅了，比安卓提供的丰富多彩很多（安卓好像就几个ClickListener）  

**作为管理的view的委托类**  
一个典型示例便是UITableViewController，它的内部持有一个tableView，在这个view加载的时候，便调用其delegate属性的一系列方法，例如获取sections的个数，获取每一个section有多少行，每行的高度是多少等等。而管理它的Controller便实现了UITableView的委托协议，这样我们就可以在Controller内部实现相应的方法，进而管理tableView。