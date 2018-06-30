---
title: iOS屏幕旋转终极攻略
date: 2018-02-22 18:44:24
tags: iOS
categories: iOS

---
发现iOS开发的一个很蛋疼的地方就是资料很难搜全，不是过时就是讲不到重点，很难有令我满意的，只能自己动手，丰衣足食喽。本篇主要介绍关于iOS开发中关于屏幕旋转的部分，包括：

- 应对一般的情况。如默认只有竖屏，不支持横屏。
- 某些特殊情况。包括只有某些是横屏且不随设备旋转而旋转，其他是竖屏。包括通过模态跳转和UINavigationController的pushViewController方法跳转到需要强制横屏的界面。
- 界面的自旋转。包括在关闭屏幕旋转(在手机上叫做竖排方向锁定)的情况下监听屏幕方向并让界面随之旋转。

笔者测试的环境是iOS 10.3

<!--more-->

# 关于横竖屏相关的设置
## 全局设置App支持的方向
首先要告诉系统自己的App所支持的方向，可以在以下三个地方设置：
### 项目配置
在项目的`General -> Deployment Info`当中可以设置支持的方向：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-02-22%20%E4%B8%8B%E5%8D%887.45.05.png)
### info.plist
上面介绍的方法和在info.plist里面修改实质上一样的，修改的是名为`Supported interface orientations`的Array。
### UIApplicationDelegate
上面所讲的两种方法都是静态配置，有时需要动态地修改App所支持的方向，这时可以实现AppDelegate中的`- (UIInterfaceOrientationMask)application:(UIApplication *)application 
  supportedInterfaceOrientationsForWindow:(UIWindow *)window;`方法，该方法返回的是App所支持的方向。关于UIInterfaceOrientationMask、UIInterfaceOrientation等等，这种枚举变量的说明网上还是有很多的，而且很靠谱。

  该方法一般使用模式是这样的：

      - (UIInterfaceOrientationMask)application:(UIApplication *)application supportedInterfaceOrientationsForWindow:(UIWindow *)window {
        if (xxx) {
            return 要返回的方向;
        } else {
            return 要返回的方向;
        }
    }
xxx是可以通过外界动态更改的条件。
## 具体ViewController的设置
主要实现的是三个方法：
### shouldAutorotate
当前的UIDevice的UIDeviceOrientation发生改变时，是否要随之旋转。当前Device的UIDeviceOrientation可以通过函数动态地改变，而忽略竖排方向锁定的开启。动态更改方向的代码如下：

    // 先用UIInterfaceOrientationUnknown进行重置
    NSNumber *orientationUnknown = [NSNumber numberWithInt:UIInterfaceOrientationUnknown];
    [[UIDevice currentDevice] setValue:orientationUnknown forKey:@"orientation"];
    
    // 此时设置的是自己想要的设备方向, 笔者选的是UIInterfaceOrientationLandscapeLeft
    NSNumber *orientationTarget = [NSNumber numberWithInt:UIInterfaceOrientationLandscapeLeft];
    [[UIDevice currentDevice] setValue:orientationTarget forKey:@"orientation"];

关于为什么要用UIInterfaceOrientationUnknown进行重置，参见[iOS强制改变物理设备方向的进阶方法](https://www.jianshu.com/p/6c45fa2bb970)。

上述做法的主要目的是调用内部API，应避免直接调用内部API(此处指的是setOrientation方法)。

网上还有一段比较流行的代码：

    if ([[UIDevice currentDevice] respondsToSelector:@selector(setOrientation:)]) {
        SEL selector  = NSSelectorFromString(@"setOrientation:");
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:[UIDevice instanceMethodSignatureForSelector:selector]];
        [invocation setSelector:selector];
        [invocation setTarget:[UIDevice currentDevice]];
        // 从2开始是因为前两个参数已经被selector和target占用
        [invocation setArgument:&orientation atIndex:2];
        [invocation invoke];
    }
这段代码可能在某些情况下是运行正常的，但不正常的情况恰好被我遇到了。情况是这样：我想通过UINavigationController的pushViewController方法跳转到需要强制横屏的界面，第一次是正常的。但是pop掉横屏界面之后，再次push之前需要强制横屏的ViewController就不能横屏了，至于原因，我也不知道……要怪就怪苹果不开源吧。
### supportedInterfaceOrientations
此方法返回的是此ViewController所支持的界面方向。值得一提的是UINavigationController(UITabBarController与之同理)，它可以与其他的ViewController构成一个层次结构。此时外部UINavigationController所支持的方向取决于其内部顶层的ViewController所支持的方向，所以应该重写UINavigationController的三个方法，让其与内部顶层ViewController保持一致：

    - (BOOL)shouldAutorotate {
        return [[self.viewControllers lastObject] shouldAutorotate];
    }

    - (UIInterfaceOrientationMask)supportedInterfaceOrientations {
        return [[self.viewControllers lastObject] supportedInterfaceOrientations];
    }

    - (UIInterfaceOrientation)preferredInterfaceOrientationForPresentation {
        return [[self.viewControllers lastObject] preferredInterfaceOrientationForPresentation];
    }
### preferredInterfaceOrientationForPresentation
通过模态进入时默认的方向。
# 一般情况
若应用只支持竖屏或者横屏，当然可以在项目配置中只勾选一种方向，但谁也保证不了App之后一定没有横屏需求，所以还是不要更改项目的配置，可以在BaseViewController当中写下面的代码：

    - (BOOL)shouldAutorotate {
        return NO;
    }

    - (UIInterfaceOrientationMask)supportedInterfaceOrientations {
        return UIInterfaceOrientationMaskPortrait;
    }

    - (UIInterfaceOrientation)preferredInterfaceOrientationForPresentation {
        return UIInterfaceOrientationPortrait;
    }
之后若需要有横屏的ViewController，重写三个方法即可。
# 强制横屏
下面分成两种情况加以讨论，下面讨论的两种情况均可忽略竖排方向锁定设置：
## 模态跳转
此种情况是通过模态跳转到需要强制横屏的ViewController，此时重写强制横屏的ViewController的三个方法即可：

    - (BOOL)shouldAutorotate {
        return NO;
    }

    - (UIInterfaceOrientationMask)supportedInterfaceOrientations {
        return UIInterfaceOrientationMaskLandscapeLeft;
    }

    - (UIInterfaceOrientation)preferredInterfaceOrientationForPresentation {
        return UIInterfaceOrientationLandscapeLeft;
    }
运行正常的前提条件是App支持横屏所需的方向。
## pushViewController
若是通过UINavigationController的pushViewController方法来进入到一个需要强制横屏的ViewController，除了上述所说需要重写UINavigationController的三个方法让其与顶部ViewController保持一致之外，还需要在需要强制横屏的ViewController当中添加如下代码：

    // 必须为YES
    - (BOOL)shouldAutorotate {
        return YES;
    }

    - (UIInterfaceOrientationMask)supportedInterfaceOrientations {
        return UIInterfaceOrientationMaskLandscapeLeft;
    }

    - (UIInterfaceOrientation)preferredInterfaceOrientationForPresentation {
        return UIInterfaceOrientationLandscapeLeft;
    }

    - (void)viewWillAppear:(BOOL)animated {
        [super viewWillAppear:animated];
        
        NSNumber *orientationUnknown = [NSNumber numberWithInt:UIInterfaceOrientationUnknown];
        [[UIDevice currentDevice] setValue:orientationUnknown forKey:@"orientation"];
        
        NSNumber *orientationTarget = [NSNumber numberWithInt:UIInterfaceOrientationLandscapeLeft];
        [[UIDevice currentDevice] setValue:orientationTarget forKey:@"orientation"];
    }
利用API来使UIDevice的方向改变上面已经提到了，此时因为是手动改变，所以即便是设置了竖排方向锁定也是无所谓的。shouldAutorotate必须设置为YES，这样才能在Device的方向改变之后随之旋转。
# 监听屏幕方向
以上所讲的，已经能解决绝大部分问题了。下面所说的是为了应对某些蛋疼的需求。如何监听屏幕方向的改变？很容易想到通过UIDevice提供的API来获得方向，但是在竖排方向锁定打开的情况下是行不通的，因此我不打算介绍UIDevice所提供的API，我要介绍一种通用的方法来获得屏幕的方向，这需要用到传感器来实现，原文地址(不容易搜到):[iOS中检测当前设备的旋转方向（关闭屏幕旋转)](http://blog.csdn.net/sunflowerinrain/article/details/78847740)。下面我通过这篇文章所带来的基本想法封装一个OrientationManager:

头文件：

    #import <DOSingleton/DOSingleton.h>
    #import <UIKit/UIKit.h>

    extern NSString *const OrientationManagerOrientationChangedKey;

    @interface OrientationManager : DOSingleton

    // 当前的方向
    @property (atomic, assign, readonly) UIDeviceOrientation orientation;

    - (void)startMonitoring;

    @end
所用到的DOSingleton是为了简化单例模式的步骤引入的第三方库，感兴趣的可以看github主页。

实现：

    #import "OrientationManager.h"
    #import <ReactiveCocoa/ReactiveCocoa.h>

    @import CoreMotion;

    NSString *const OrientationManagerOrientationChangedKey = @"OrientationChanged";

    @interface OrientationManager()

    @property (nonatomic, strong) CMMotionManager *manager;

    @property (atomic, assign, readwrite) UIDeviceOrientation orientation;

    @property (nonatomic, assign, getter=isStarted) BOOL started;

    @end

    @implementation OrientationManager

    - (instancetype)init {
        if (!self.isInitialized) {
            self = [super init];
            self.orientation = UIDeviceOrientationUnknown;
        }
        return self;
    }

    - (void)dealloc {
        [_manager stopDeviceMotionUpdates];
    }

    - (void)startMonitoring {
        if (!self.isStarted) {
            NSOperationQueue *queue = [[NSOperationQueue alloc] init];
            self.manager = [[CMMotionManager alloc] init];
            // 1s中取样15次，可自行设定
            _manager.accelerometerUpdateInterval = 1.0 / 15.0;
            if ([_manager isDeviceMotionAvailable]) {
                @weakify(self);
                [_manager startDeviceMotionUpdatesToQueue:queue withHandler:^(CMDeviceMotion * _Nullable motion, NSError * _Nullable error) {
                    if (!error && motion) {
                        @strongify(self);
                        [self handleWithDeviceMotion:motion];
                    }
                }];
                self.started = YES;
            }
        }
    }

    - (void)handleWithDeviceMotion:(CMDeviceMotion *)motion {
        UIDeviceOrientation newOrientation;
        
        double x = motion.gravity.x;
        double y = motion.gravity.y;
        double z = motion.gravity.z;
        
        if (fabs(z) >= 0.75) {
            // 认为是平置的，因为只考虑横竖屏，所以只考虑x, y轴，忽略z轴
            return;
        }
        
        
        if (fabs(y) >= fabs(x)) {
            if (y >= 0){
                newOrientation = UIDeviceOrientationPortraitUpsideDown;
            } else {
                newOrientation = UIDeviceOrientationPortrait;
            }
        } else {
            if (x >= 0) {
                newOrientation = UIDeviceOrientationLandscapeRight;
            } else {
                newOrientation = UIDeviceOrientationLandscapeLeft;
            }
        }
        
        if (self.orientation == newOrientation) {
            return;
        } else {
            // 更新，并且发送通知
            self.orientation = newOrientation;
            dispatch_async(dispatch_get_main_queue(), ^{
                // 放在主线程发送通知
                [[NSNotificationCenter defaultCenter] postNotificationName:OrientationManagerOrientationChangedKey object:nil];
            });
        }
    }

    @end

在ViewController里面简单使用一下：

    - (void)viewDidLoad {
        [super viewDidLoad];
        [[OrientationManager sharedInstance] startMonitoring];
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(onOrientationChanged) name:OrientationManagerOrientationChangedKey object:nil];
    }

    - (void)dealloc {
        [[NSNotificationCenter defaultCenter] removeObserver:self];
    }

    - (void)onOrientationChanged {
        switch ([OrientationManager sharedInstance].orientation) {
            case UIDeviceOrientationLandscapeLeft:
                NSLog(@"Landscape left");
                break;
            case UIDeviceOrientationLandscapeRight:
                NSLog(@"Landscape right");
                break;
            case UIDeviceOrientationPortrait:
                NSLog(@"Portrait");
                break;
            case UIDeviceOrientationPortraitUpsideDown:
                NSLog(@"Portrait upside down");
                break;
            default:
                NSLog(@"Unknown");
                break;
        }
    }
看一下运行情况：

    2018-02-22 23:32:36.976260+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:32:41.594187+0800 Demo[6183:1150263] Landscape left
    2018-02-22 23:32:50.197738+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:32:51.828630+0800 Demo[6183:1150263] Landscape right
    2018-02-22 23:32:53.507366+0800 Demo[6183:1150263] Portrait upside down
    2018-02-22 23:32:57.110077+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:32:58.152403+0800 Demo[6183:1150263] Landscape left
    2018-02-22 23:32:58.902556+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:33:03.097224+0800 Demo[6183:1150263] Landscape left
    2018-02-22 23:33:03.847134+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:33:04.206700+0800 Demo[6183:1150263] Landscape right
    2018-02-22 23:33:04.821790+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:33:21.216680+0800 Demo[6183:1150263] Landscape right
    2018-02-22 23:33:23.231701+0800 Demo[6183:1150263] Landscape left
    2018-02-22 23:33:24.158209+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:33:25.599591+0800 Demo[6183:1150263] Portrait upside down
    2018-02-22 23:33:26.815896+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:35:06.726776+0800 Demo[6183:1150263] Landscape right
    2018-02-22 23:35:08.311154+0800 Demo[6183:1150263] Landscape left
    2018-02-22 23:35:09.626317+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:35:23.933861+0800 Demo[6183:1150263] Portrait upside down
    2018-02-22 23:35:25.501846+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:35:28.884317+0800 Demo[6183:1150263] Landscape right
    2018-02-22 23:35:36.545293+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:35:36.545943+0800 Demo[6183:1150263] Landscape left
    2018-02-22 23:35:37.145841+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:35:38.886784+0800 Demo[6183:1150263] Landscape left
    2018-02-22 23:35:39.608811+0800 Demo[6183:1150263] Portrait
    2018-02-22 23:35:39.609253+0800 Demo[6183:1150263] Landscape right
    2018-02-22 23:35:40.293250+0800 Demo[6183:1150263] Portrait
嗯，完美！

这时如果要旋转界面的话，一种做法是利用UIView的transform属性，这种方法比较灵活，可以选择性的旋转想要旋转的View；另一种是上面提到的强制横屏的做法，就是发现改变之后利用API更改UIDevice的方向，这样可以忽略竖排方向锁定带来的影响。


