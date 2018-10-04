---
title: 亲身动手实验得出anchorPoint和point的关系
date: 2018-10-04 22:54:36
tags: iOS
categories: iOS

---

最近准备读一本很晦涩的书：[iOS Core Animation Advanced Techniques](https://github.com/AttackOnDobby/iOS-Core-Animation-Advanced-Techniques), 里面简单的提到了anchorPoint和point的概念，上网搜到了一篇比较不错的文章：[这将是你最后一次纠结position与anchorPoint](http://kittenyang.com/anchorpoint/)，但是理论比较多，我偏向于看效果。自己动手，丰衣足食。

本篇主要探索的内容是：

- anchorPoint，point和frame三者的关系。
- anchorPoint存在的意义。

<!--more-->

# 实验代码
测试的逻辑是这样的，采用**控制变量法**：

- 更改anchorPoint，查看对frame和point的影响。
- 更改point，查看对frame和anchorPoint的影响。
- 更改frame，查看对point和anchorPoint的影响。

下面是主要的代码：

    #import "MainViewController.h"

    @interface MainViewController ()

    @property (nonatomic, weak) CALayer *targetLayer;

    @end

    @implementation MainViewController

    - (void)viewDidLoad {
        [super viewDidLoad];
        
        self.view.backgroundColor = [UIColor blueColor];
        
        [self reset];
        
        UIButton *test1Btn = [[UIButton alloc] initWithFrame:CGRectMake(10, 10, 60, 80)];
        test1Btn.tag = 1;
        [test1Btn addTarget:self action:@selector(onTestBtnClicked:) forControlEvents:UIControlEventTouchUpInside];
        [test1Btn setTitle:@"测试1" forState:UIControlStateNormal];
        [self.view addSubview:test1Btn];
        
        UIButton *test2Btn = [[UIButton alloc] initWithFrame:CGRectMake(80, 10, 60, 80)];
        test2Btn.tag = 2;
        [test2Btn addTarget:self action:@selector(onTestBtnClicked:) forControlEvents:UIControlEventTouchUpInside];
        [test2Btn setTitle:@"测试2" forState:UIControlStateNormal];
        [self.view addSubview:test2Btn];
        
        UIButton *test3Btn = [[UIButton alloc] initWithFrame:CGRectMake(150, 10, 60, 80)];
        test3Btn.tag = 3;
        [test3Btn addTarget:self action:@selector(onTestBtnClicked:) forControlEvents:UIControlEventTouchUpInside];
        [test3Btn setTitle:@"测试3" forState:UIControlStateNormal];
        [self.view addSubview:test3Btn];
        
        UIButton *resetBtn = [[UIButton alloc] initWithFrame:CGRectMake(220, 10, 60, 80)];
        resetBtn.tag = 4;
        [resetBtn addTarget:self action:@selector(onTestBtnClicked:) forControlEvents:UIControlEventTouchUpInside];
        [resetBtn setTitle:@"重置" forState:UIControlStateNormal];
        [self.view addSubview:resetBtn];
    }

    - (void)onTestBtnClicked:(UIButton *)sender {
        switch (sender.tag) {
            case 1: {
                // 递减0.1
                __strong CALayer *sublayer = _targetLayer;
                sublayer.anchorPoint = CGPointMake(sublayer.anchorPoint.x - 0.1, sublayer.anchorPoint.y - 0.1);
                NSLog(@"AnchorPoint changed! Frame is : %@ Point is : %@", [NSValue valueWithCGRect:sublayer.frame], [NSValue valueWithCGPoint:sublayer.position]);
                break;
            }
            
            case 2: {
                __strong CALayer *sublayer = _targetLayer;
                // 递减20
                sublayer.position = CGPointMake(sublayer.position.x - 20, sublayer.position.y - 20);
                NSLog(@"Position changed! Frame is : %@ AnchorPoint is : %@", [NSValue valueWithCGRect:sublayer.frame], [NSValue valueWithCGPoint:sublayer.anchorPoint]);
                break;
            }
            
            case 3: {
                __strong CALayer *sublayer = _targetLayer;
                // 递减20
                sublayer.frame = CGRectMake(sublayer.frame.origin.x - 20, sublayer.frame.origin.y - 20, sublayer.frame.size.width, sublayer.frame.size.height);
                NSLog(@"Frame changed! AnchorPoint is : %@ Position is : %@", [NSValue valueWithCGPoint:sublayer.anchorPoint], [NSValue valueWithCGPoint:sublayer.position]);
                break;
            }
                
            default:
                [self reset];
                break;
        }
    }

    - (void)reset {
        [_targetLayer removeFromSuperlayer];
        CALayer *sublayer = [CALayer new];
        sublayer.backgroundColor = [UIColor redColor].CGColor;
        sublayer.frame = CGRectMake(0, 0, 200, 200);
        [self.view.layer addSublayer:sublayer];
        // 居中处理
        sublayer.position = self.view.center;
        
        self.targetLayer = sublayer;
        
        NSLog(@"Inital anchorPoint is : %@, frame is : %@, position is : %@", [NSValue valueWithCGPoint:sublayer.anchorPoint], [NSValue valueWithCGRect:sublayer.frame], [NSValue valueWithCGPoint:sublayer.position]);
    }

    @end

# 实验效果

测试1：

![测试1](http://ok34fi9ya.bkt.clouddn.com/%E6%B5%8B%E8%AF%951.gif)

输出日志：

    2018-10-04 23:58:44.427771+0800 Beginner[39499:14491751] Inital anchorPoint is : NSPoint: {0.5, 0.5}, frame is : NSRect: {{87.5, 306}, {200, 200}}, position is : NSPoint: {187.5, 406}
    2018-10-04 23:58:49.877996+0800 Beginner[39499:14491751] AnchorPoint changed! Frame is : NSRect: {{107.5, 326}, {200, 200}} Position is : NSPoint: {187.5, 406}
    2018-10-04 23:58:50.403453+0800 Beginner[39499:14491751] AnchorPoint changed! Frame is : NSRect: {{127.49999999999999, 346}, {200, 200}} Position is : NSPoint: {187.5, 406}
    2018-10-04 23:58:50.986726+0800 Beginner[39499:14491751] AnchorPoint changed! Frame is : NSRect: {{147.5, 366}, {200, 200}} Position is : NSPoint: {187.5, 406}

测试2：

![测试2](http://ok34fi9ya.bkt.clouddn.com/%E6%B5%8B%E8%AF%952.gif)

输出日志：

    2018-10-05 00:00:45.085862+0800 Beginner[39499:14491751] Inital anchorPoint is : NSPoint: {0.5, 0.5}, frame is : NSRect: {{87.5, 306}, {200, 200}}, position is : NSPoint: {187.5, 406}
    2018-10-05 00:00:50.069012+0800 Beginner[39499:14491751] Position changed! Frame is : NSRect: {{67.5, 286}, {200, 200}} AnchorPoint is : NSPoint: {0.5, 0.5}
    2018-10-05 00:00:50.703337+0800 Beginner[39499:14491751] Position changed! Frame is : NSRect: {{47.5, 266}, {200, 200}} AnchorPoint is : NSPoint: {0.5, 0.5}
    2018-10-05 00:00:51.130704+0800 Beginner[39499:14491751] Position changed! Frame is : NSRect: {{27.5, 246}, {200, 200}} AnchorPoint is : NSPoint: {0.5, 0.5}

测试3：

![测试3](http://ok34fi9ya.bkt.clouddn.com/%E6%B5%8B%E8%AF%953.gif)

输出日志：

    2018-10-05 00:01:30.949345+0800 Beginner[39499:14491751] Inital anchorPoint is : NSPoint: {0.5, 0.5}, frame is : NSRect: {{87.5, 306}, {200, 200}}, position is : NSPoint: {187.5, 406}
    2018-10-05 00:01:32.690235+0800 Beginner[39499:14491751] Frame changed! AnchorPoint is : NSPoint: {0.5, 0.5} Position is : NSPoint: {167.5, 386}
    2018-10-05 00:01:33.234414+0800 Beginner[39499:14491751] Frame changed! AnchorPoint is : NSPoint: {0.5, 0.5} Position is : NSPoint: {147.5, 366}
    2018-10-05 00:01:33.784964+0800 Beginner[39499:14491751] Frame changed! AnchorPoint is : NSPoint: {0.5, 0.5} Position is : NSPoint: {127.5, 346}

# 结论
现在根据上述三个实验得出的结论有：

- anchorPoint不会影响position, 但是会影响frame。
- position不会影响anchorPoint, 但是会影响frame。
- frame不会影响anchorPoint, 但是会影响position。

开头给出的文章中得出的结论大致是对的, 现在贴一个万金油公式：

frame.origin.x = position.x - anchorPoint.x * bounds.size.width；  
frame.origin.y = position.y - anchorPoint.y * bounds.size.height；

position实际上就是anchorPoint在superLayer中的位置。

那么anchorPoint的存在意义是什么呢？

先看一下官方文档的描述吧：

```You specify the value for this property using the unit coordinate space. The default value of this property is (0.5, 0.5), which represents the center of the layer’s bounds rectangle. All geometric manipulations to the view occur about the specified point. For example, applying a rotation transform to a layer with the default anchor point causes the layer to rotate around its center. Changing the anchor point to a different location would cause the layer to rotate around that new point.```

大致的意思是之后做出的几何特性的改变都以anchorPoint作为中心点，例如想要围绕某点进行旋转就可以设置anchorPoint属性。
