---
layout:     post
title:      RunLoop的使用场景
date:       2018-07-09
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Obj-C
    - RunLoop
    - iOS
--- 

# RunLoop的使用场景

## 1.保证线程的长时间存活

在iOS开发过程中，有时候我们不希望一些花费时间比较长的操作阻塞主线程，导致界面卡顿，那么我们就会创建一个子线程，然后把这些花费时间比较长的操作放在子线程中来处理。可是当子线程中的任务执行完毕后，子线程就会被销毁掉。 

当子线程中的任务执行完毕后，线程就被立刻销毁了。如果程序中，需要经常在子线程中执行任务，频繁的创建和销毁线程，会造成资源的浪费。这时候我们就可以使用RunLoop来让该线程长时间存活而不被销毁。

``` objectivec

/** 子线程启动后，启动runloop */
- (void)subThreadEntryPoint
{
    NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
    //如果注释了下面这一行，子线程中的任务并不能正常执行
    [runLoop addPort:[NSMachPort port] forMode:NSRunLoopCommonModes];
    NSLog(@"启动RunLoop前--%@",runLoop.currentMode);
    [runLoop run];
}

/** 子线程任务 */
- (void)subThreadOpetion
{
    NSLog(@"启动RunLoop后--%@",[NSRunLoop currentRunLoop].currentMode);
    NSLog(@"%@----子线程任务开始",[NSThread currentThread]);
    [NSThread sleepForTimeInterval:3.0];
    NSLog(@"%@----子线程任务结束",[NSThread currentThread]);
}

```
> 有几点需要注意： 
 - 1.获取RunLoop只能使用 [NSRunLoop currentRunLoop] 或 [NSRunLoop mainRunLoop]; 
 - 2.即使RunLoop开始运行，如果RunLoop 中的 modes 为空，或者要执行的mode里没有item，那么RunLoop会直接在当前loop中返回，并进入睡眠状态。 
 - 3.自己创建的Thread中的任务是在kCFRunLoopDefaultMode这个mode中执行的。

## 2.保证NSTimer在视图滑动时，依然能正常运转

1.我们经常会在应用中看到tableView 的header 上是一个横向ScrollView，一般我们使用NSTimer，每隔几秒切换一张图片。可是当我们滑动tableView的时候，顶部的scollView并不会切换图片，这可怎么办呢？ 
2.界面上除了有tableView，还有显示倒计时的Label，当我们在滑动tableView时，倒计时就停止了，这又该怎么办呢？
> 原因是当我们滑动scrollView时，主线程的RunLoop 会切换到UITrackingRunLoopMode这个Mode，执行的也是UITrackingRunLoopMode下的任务（Mode中的item），而timer 是添加在NSDefaultRunLoopMode下的，所以timer任务并不会执行，只有当UITrackingRunLoopMode的任务执行完毕，runloop切换到NSDefaultRunLoopMode后，才会继续执行timer。
解决方法很简单，我们只需要在添加timer 时，将mode 设置为NSRunLoopCommonModes即可。

``` objectivec

    - (void)timerTest {
        // 第一种写法
        NSTimer *timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(timerUpdate) userInfo:nil repeats:YES];
        [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
        [timer fire];
        // 第二种写法，因为是固定添加到defaultMode中，就不要用了
    }

```

###其他一些关于timer的坑

我们在子线程中使用timer，也可以解决上面的问题，但是需要注意的是把timer加入到当前runloop后，必须让runloop 运行起来，否则timer仅执行一次。

这就是多线程与runloop的关系了，每一个线程都有一个与之关联的RunLoop，而每一个RunLoop可能会有多个Mode。CPU会在多个线程间切换来执行任务，呈现出多个线程同时执行的效果。执行的任务其实就是RunLoop去各个Mode里执行各个item。因为RunLoop是独立的两个，相互不会影响，所以在子线程添加timer，滑动视图时，timer能正常运行。

## 3.使用RunLoop监测主线程的卡顿,并将卡顿时的线程堆栈信息保存下来

### 原理

官方文档说明了RunLoop的执行顺序：
``` bash
1. Notify observers that the run loop has been entered.
2. Notify observers that any ready timers are about to fire.
3. Notify observers that any input sources that are not port based are about to fire.
4. Fire any non-port-based input sources that are ready to fire.
5. If a port-based input source is ready and waiting to fire, process the event immediately. Go to step 9.
6. Notify observers that the thread is about to sleep.
7. Put the thread to sleep until one of the following events occurs:
 * An event arrives for a port-based input source.
 * A timer fires.
 * The timeout value set for the run loop expires.
 * The run loop is explicitly woken up.
8. Notify observers that the thread just woke up.
9. Process the pending event.
 * If a user-defined timer fired, process the timer event and restart the loop. Go to step 2.
 * If an input source fired, deliver the event.
 * If the run loop was explicitly woken up but has not yet timed out, restart the loop. Go to step 2.
10. Notify observers that the run loop has exited.
```
用伪代码来实现就是这样的：

``` cpp
{
    /// 1. 通知Observers，即将进入RunLoop
    /// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
    do {

        /// 2. 通知 Observers: 即将触发 Timer 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
        /// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);

        /// 4. 触发 Source0 (非基于port的) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);

        /// 6. 通知Observers，即将进入休眠
        /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);

        /// 7. sleep to wait msg.
        mach_msg() -> mach_msg_trap();


        /// 8. 通知Observers，线程被唤醒
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);

        /// 9. 如果是被Timer唤醒的，回调Timer
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);

        /// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);

        /// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);


    } while (...);

    /// 10. 通知Observers，即将退出RunLoop
    /// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}
```

主线程的RunLoop是在应用启动时自动开启的，也没有超时时间，所以正常情况下，主线程的RunLoop 只会在 2—9 之间无限循环下去。 
那么，我们只需要在主线程的RunLoop中添加一个observer，检测从 kCFRunLoopBeforeSources 到 kCFRunLoopBeforeWaiting 花费的时间 是否过长。如果花费的时间大于某一个阙值，我们就认为有卡顿，并把当前的线程堆栈转储到文件中，并在以后某个合适的时间，将卡顿信息文件上传到服务器。

**实现步骤**
在看了上面的两个监测卡顿的示例Demo后，我按照上面讲述的思路写了一个Demo，应该更容易理解吧。 
第一步，创建一个子线程，在线程启动时，启动其RunLoop。
``` objectivec
+ (instancetype)shareMonitor
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[[self class] alloc] init];
        instance.monitorThread = [[NSThread alloc] initWithTarget:self selector:@selector(monitorThreadEntryPoint) object:nil];
        [instance.monitorThread start];
    });

    return instance;
}

+ (void)monitorThreadEntryPoint
{
    [[NSThread currentThread] setName:@"FluencyMonitor"];
    NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
    [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
    [runLoop run];
}
```
第二步，在开始监测时，往主线程的RunLoop中添加一个observer，并往子线程中添加一个定时器，每0.5秒检测一次耗时的时长。
``` objectivec

- (void)start
{
    if (_observer) {
        return;
    }

    // 1.创建observer
    CFRunLoopObserverContext context = {0,(__bridge void*)self, NULL, NULL, NULL};
    _observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                              kCFRunLoopAllActivities,
                                              YES,
                                              0,
                                              &runLoopObserverCallBack,
                                              &context);
    // 2.将observer添加到主线程的RunLoop中
    CFRunLoopAddObserver(CFRunLoopGetMain(), _observer, kCFRunLoopCommonModes);

    // 3.创建一个timer，并添加到子线程的RunLoop中
    [self performSelector:@selector(addTimerToMonitorThread) onThread:self.monitorThread withObject:nil waitUntilDone:NO modes:@[NSRunLoopCommonModes]];
}

- (void)addTimerToMonitorThread
{
    if (_timer) {
        return;
    }
    // 创建一个timer
    CFRunLoopRef currentRunLoop = CFRunLoopGetCurrent();
    CFRunLoopTimerContext context = {0, (__bridge void*)self, NULL, NULL, NULL};
    _timer = CFRunLoopTimerCreate(kCFAllocatorDefault, 0.1, 0.01, 0, 0,
                                                   &runLoopTimerCallBack, &context);
    // 添加到子线程的RunLoop中
    CFRunLoopAddTimer(currentRunLoop, _timer, kCFRunLoopCommonModes);
}

```

第三步，补充观察者回调处理

``` objectivec

static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info){
    FluencyMonitor *monitor = (__bridge FluencyMonitor*)info;
    NSLog(@"MainRunLoop---%@",[NSThread currentThread]);
    switch (activity) {
        case kCFRunLoopEntry:
            NSLog(@"kCFRunLoopEntry");
            break;
        case kCFRunLoopBeforeTimers:
            NSLog(@"kCFRunLoopBeforeTimers");
            break;
        case kCFRunLoopBeforeSources:
            NSLog(@"kCFRunLoopBeforeSources");
            monitor.startDate = [NSDate date];
            monitor.excuting = YES;
            break;
        case kCFRunLoopBeforeWaiting:
            NSLog(@"kCFRunLoopBeforeWaiting");
            monitor.excuting = NO;
            break;
        case kCFRunLoopAfterWaiting:
            NSLog(@"kCFRunLoopAfterWaiting");
            break;
        case kCFRunLoopExit:
            NSLog(@"kCFRunLoopExit");
            break;
        default:
            break;
    }
}

```

从打印信息来看，RunLoop进入睡眠状态的时间可能会非常短，有时候只有1毫秒，有时候甚至1毫秒都不到，静止不动时，则会长时间进入睡觉状态

因为主线程中的block、交互事件、以及其他任务都是在kCFRunLoopBeforeSources 到 kCFRunLoopBeforeWaiting 之前执行，所以我在即将开始执行Sources 时，记录一下时间，并把正在执行任务的标记置为YES，将要进入睡眠状态时，将正在执行任务的标记置为NO。

第四步，补充timer 的回调处理

``` objectivec

static void runLoopTimerCallBack(CFRunLoopTimerRef timer, void *info)
{
    FluencyMonitor *monitor = (__bridge FluencyMonitor*)info;
    if (!monitor.excuting) {
        return;
    }

    // 如果主线程正在执行任务，并且这一次loop 执行到 现在还没执行完，那就需要计算时间差
    NSTimeInterval excuteTime = [[NSDate date] timeIntervalSinceDate:monitor.startDate];
    NSLog(@"定时器---%@",[NSThread currentThread]);
    NSLog(@"主线程执行了---%f秒",excuteTime);

    if (excuteTime >= 0.01) {
        NSLog(@"线程卡顿了%f秒",excuteTime);
        [monitor handleStackInfo];
    }
}

```

剩下的工作就是将字符串保存进文件，以及上传到服务器了。

我们不能将卡顿的阙值定的太小，也不能将所有的卡顿信息都上传，原因有两点，一，太浪费用户流量；二、文件太多，App内存储和上传后服务器端保存都会占用空间。

可以参考微信的做法，7天以上的文件删除，随机抽取上传，并且上传前对文件进行压缩处理等。