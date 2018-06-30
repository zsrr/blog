---
title: iOS触摸事件
date: 2018-06-25 21:01:38
tags: [iOS,UIKit]
categories: iOS

---
本篇开始进行iOS的进阶学习，大致分成四部曲：

- 基础API的使用，包括UIKit和CoreAnimation等等，主要看一下文档，总结即可。
- OC进阶。
- iOS系统进阶。
- 几个主要的库的源代码看一下。

以下内容参考：

- [iOS事件响应链中Hit-Test View的应用](https://www.jianshu.com/p/d8512dff2b3e)
- [Using Responders and the Responder Chain to Handle Events](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/using_responders_and_the_responder_chain_to_handle_events?language=objc)
- [iOS手势识别详解-GestureRecognizer部分](https://www.jianshu.com/p/0da1761abb03)

# Responder Chain
首先要介绍一下UIResponder Chain
## UIResponder 
这个不必多说，UIView, UIViewController, UIApplication都是UIResponder的子类，都可以响应交互事件。

下面的这幅图，摘自官方文档，这些类实例组成一个树形结构，构成了一个完整的Responder Chain。

![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-06-25%20%E4%B8%8B%E5%8D%8810.17.23.png)

## First Responder
关于如何找到第一响应者，这里有一个关键的函数：

    - (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event;

当用户点击屏幕(不管是点击还是什么动作，能产生UITouch即可)时，找到当前点击区域内在Responder Chain中最顶层的UIView，这个View就是hit-test view。找到hit-test view之后，这个View将会选择是否处理此Event，如果不想处理的话将会交给父级UIView进行处理。

在执行hitTest方法时，会先判断此event是否在自己的bounds之内(默认实现)，如果是再进行递归调用，不是的话则返回nil，判断UIEvent是否发生在自己的bounds内部的实现函数为：

    - (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event;

hitTest的逻辑大致是这样的:

    - (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
        if (!self.isUserInteractionEnabled || self.isHidden || self.alpha <= 0.01) {
            return nil;
        }
        if ([self pointInside:point withEvent:event]) {
            for (UIView *subview in [self.subviews reverseObjectEnumerator]) {
                CGPoint convertedPoint = [subview convertPoint:point fromView:self];
                UIView *hitTestView = [subview hitTest:convertedPoint withEvent:event];
                if (hitTestView) {
                    return hitTestView;
                }
            }
            return self;
        }
        return nil;
    }

### 增大UIButton热响应区域
下面介绍一个简单的应用。UIButton官方推荐的响应区域大小至少是44x44，但是在实际开发过程中我们会遇到很多小图，这时候需要自己扩展响应区域。解决这个问题很简单，只需要重写UIButton的pointInside:withEvent方法即可：

    - (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event {
        return CGRectContainsPoint(HitTestingBounds(self.bounds, self.minimumHitTestWidth, self.minimumHitTestHeight), point);
    }

    CGRect HitTestingBounds(CGRect bounds, CGFloat minimumHitTestWidth, CGFloat minimumHitTestHeight) {
        CGRect hitTestingBounds = bounds;
        if (minimumHitTestWidth > bounds.size.width) {
            hitTestingBounds.size.width = minimumHitTestWidth;
            hitTestingBounds.origin.x -= (hitTestingBounds.size.width - bounds.size.width)/2;
        }
        if (minimumHitTestHeight > bounds.size.height) {
            hitTestingBounds.size.height = minimumHitTestHeight;
            hitTestingBounds.origin.y -= (hitTestingBounds.size.height - bounds.size.height)/2;
        }
        return hitTestingBounds;
    }
## Next Responder
当选中的First Responder选择不响应当前触摸事件的时候，此时会把事件传到nextResponder，nextResponder的寻找规则为(下方摘自官方文档)：

- UIView objects. If the view is the root view of a view controller, the next responder is the view controller; otherwise, the next responder is the view’s superview.

- UIViewController objects. If the view controller’s view is the root view of a window, the next responder is the window object.If the view controller was presented by another view controller, the next responder is the presenting view controller.

- UIWindow objects. The window's next responder is the UIApplication object.

- UIApplication object. The next responder is the app delegate, but only if the app delegate is an instance of UIResponder and is not a view, view controller, or the app object itself.

# Gesture Recognizer
在这里不想多说用法，详情见官方文档，Gesture Recognizer不参与Responder Chain。如果手势识别成功的话，那么会调用View的touchesCancelled:forEvent:，同时其他关联的GestureRecognizer失败。识别失败则后续的UITouch和UIEvent继续传给对应的View，前提条件是View实现了对应的方法。

这里需要注意的一件事是，无论一个UIView是否是最终响应事件的Responder，如果它有附加的Recognizer，那么这个Recognizer的手势识别过程是一定有的。也就是说如果一个父级View有一个GestureRecognizer，点击子级View，这时候即便子级View是最终实际响应者，手势识别过程也是存在的，这个时候父级的GestureRecognizer识别完成之后，会调用子View的touchesCancelled方法。曾经遇到过一个很苦恼的问题，一个点赞按钮，点击似乎有延迟，最后发现是父层View加了一个双击手势识别的GestureRecognizer，这个还是有一点耗时的。

# 自己处理触摸事件
也可以自己处理相应的触摸事件，需要选择性地重写4个方法：

    - (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event;
    - (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event;
    - (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event;
    - (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event;

上面的几个方法很好理解。UIEvent就是代表着一个完整的触摸流程；UITouch是记录着手指的移动，有可能不止一个手指，这里参数为NSSet类型是因为有可能开启了多点触控。UITouch对象记录着位置，力度大小，半径等等信息。

四个方法需选择性地重写，对于一个手势，当UIKit发现没有重写对应方法，也没有对应的Recognizer可以识别，那对应的UITouch和UIEvent就会传到nextResponder。


