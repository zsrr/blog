---
title: iOS UIView
date: 2017-02-01 20:37:56
tags: iOS
categories: iOS开发

---
## UIView简介
官方文档：  
>The UIView class defines a rectangular area on the screen and the interfaces for managing the content in that area.  

说白了就是一块展现内容还能处理交互的矩形区域，比如一个按钮，一个输入框，重要性不再多说。  
## 重点
**设置外观**  
这个没什么好说的，常见的有backgroundColor(背景色)，isHidden(是否隐藏)，alpha(透明度)等等。一个比较重要的属性是layer，这个稍后再谈。  

**frame, bounds**  
先贴上一篇讲的非常好的博客：  
[frame和bounds的区别](http://www.jianshu.com/p/964313cfbdaa)  
通俗的讲，frame是针对superview来说的，bounds是针对subview来说的，下面来写一个小程序验证一下有什么作用:  
代码如下：  

	private var subView: UIView?
    private var subViewsSubView: UIView?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setUp()
    }

    private func setUp() {
        setUpControllButtons()
        setUpSubviews()
    } 
        private func setUp() {
        setUpControllButtons()
        setUpSubviews()
    }
    
    private func setUpSubviews() {
        subView = UIView(frame: CGRect(x: 40.0, y: 40.0, width: 200, height: 300))
        subViewsSubView = UIView(frame: CGRect(x: 20.0, y: 20.0, width: 50, height: 50))
        subViewsSubView?.backgroundColor = UIColor.yellow
        subView?.backgroundColor = UIColor.blue
        view.addSubview(subView!)
        subView?.addSubview(subViewsSubView!)
    }
    
    private func setUpControllButtons() {
        let cb1 = UIButton(frame: CGRect(x: 50.0, y: 500.0, width: 100, height: 100))
        let cb2 = UIButton(frame: CGRect(x: 250.0, y: 500.0, width: 100, height: 100))
        
        ......
        
        cb1.addGestureRecognizer(UITapGestureRecognizer(target: self, action: #selector(ViewController.onCb1Click)))
        cb2.addGestureRecognizer(UITapGestureRecognizer(target: self, action: #selector(ViewController.onCb2Click)))
        
        view.addSubview(cb1)
        view.addSubview(cb2)
    }
    
    func onCb1Click() {
        if let subView = self.subView {
            subView.frame = CGRect(x: subView.frame.minX, y: subView.frame.minY, width: subView.frame.width + 10, height: subView.frame.height + 10)
        }
    }
    
    func onCb2Click() {
        if let subView = self.subView {
            subView.bounds = CGRect(x: subView.bounds.minX + 10, y: subView.bounds.minY + 10, width: subView.bounds.width, height: subView.bounds.height)
        }
    }
程序很简单，先是创建一个subView，设置其在父view(即Controller的RootView)的坐标(40.0, 40.0)，宽200，高300。再往subView里面添加子视图subViewsSubView，其相对于subView的坐标为(20.0, 20.0)。两个控制的按钮，一个title为frame，点击的效果是将subView的frame宽高都加10，效果如下：
![](http://ok34fi9ya.bkt.clouddn.com/screen1.gif)
可见frame的宽和高便是控件实际的宽高，而且改变之后，整个矩形区域的左上角的位置不发生改变，宽和高加长，现在修改onCb1Click()方法：

    func onCb1Click() {
        if let subView = self.subView {
            subView.frame = CGRect(x: subView.frame.minX + 10, y: subView.frame.minY + 10, width: subView.frame.width, height: subView.frame.height)
        }
    }
运行效果如下：
![](http://ok34fi9ya.bkt.clouddn.com/screen2.gif)
可见frame的minX, minY是矩形区域左上角在父view的坐标，当其改变时，它在父view的位置便发生了改变。  
下面来看bounds按钮点击的效果：
![](http://ok34fi9ya.bkt.clouddn.com/screen3.gif)
当bounds的宽高改变的时候，view的宽高也会改变，不同的是frame的宽高改变的时候不动点在左上角，而bounds的宽高改变的时候不动点在view初始的center。改变onCb2Click()方法如下：

    func onCb2Click() {
        if let subView = self.subView {
            subView.bounds = CGRect(x: subView.bounds.minX - 10, y: subView.bounds.minY - 10, width: subView.bounds.width, height: subView.bounds.height)
        }
    }
运行效果如图：
![](http://ok34fi9ya.bkt.clouddn.com/screen4.gif)
这里应该这么想，初始的时候bounds的minX, minY都是0(坐标原点)，现在将其都剪去10，那么左上角变成了(-10, -10)，坐标原点向右下角移动，子view的位置是根据坐标原点的位置确定的，所以子view也跟着右下角移动了。

**处理交互事件**  
UIView继承于UIResponder，若想自定义的view按照自己的逻辑处理交互事件的话，需要重写四个方法：  

	func touchesBegan(Set<UITouch>, with: UIEvent?)
	func touchesMoved(Set<UITouch>, with: UIEvent?)
	func touchesEnded(Set<UITouch>, with: UIEvent?)
	func touchesCancelled(Set<UITouch>, with: UIEvent?)
这四个具体怎么用请看官方文档（我就是懒）  
还有一种简单一点的办法，是在view初始化的时候给它注册GestureRecogniser，这时候target是自己就可以了。  

**layer属性**  
这是个比较重要的属性，可以控制view的外观，像圆角，阴影，渐变背景色都可以通过它来设置（搞不懂为什么view可以直接设置背景色向外暴露backgroundColor属性，不优雅），不过我认为它并不如安卓那样通过xml来控制简单方便。它是一个CALayer类，几个常用的属性：  

	var cornerRadius: CGFloat { get set } //设置圆角
	var borderWidth: CGFloat { get set } //边框的宽度
	var borderColor: CGColor? { get set } //边框的颜色
	var backgroundColor: CGColor? { get set } //背景色
此外，它还有一系列的子类，例如CAGradientLayer(设置渐变色)，CATextLayer, CAShapeLayer。下面画一个拥有渐变色背景的Button吧！ 代码如下：  

    override func viewDidLoad() {
        super.viewDidLoad()
        setUp()
    }
    
    private func setUp() {
        let button = UIButton()
        button.frame = CGRect(x: 40.0, y: 40.0, width: 300, height: 200)
        view.addSubview(button)
        
        let backgroundLayer = CAGradientLayer()
        //设置frame，不要忽略这一步！
        backgroundLayer.frame = button.layer.bounds
        //设置起点和终点, 范围是(0,0) -> (1, 1)
        backgroundLayer.startPoint = CGPoint(x: 0.5, y: 1.0)
        backgroundLayer.endPoint = CGPoint(x: 0.5, y: 0.0)
        //渐变颜色数组
        backgroundLayer.colors = [ UIColor.blue.cgColor, UIColor.red.cgColor ]
        button.layer.addSublayer(backgroundLayer)
        
        button.layer.cornerRadius = 10
    }
运行效果如下：  
![](http://ok34fi9ya.bkt.clouddn.com/IMG_2159.PNG)