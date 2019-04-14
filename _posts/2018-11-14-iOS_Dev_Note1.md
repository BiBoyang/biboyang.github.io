---
layout: post
title: RunLoop笔记
date: 2018-11-14
tags: iOS
---

# RunLoop笔记

对于最开始我们写的程序来说，执行完任务之后就会退出。
比如这样的

```
int main(int argc, char * argv[]) {
    NSlog(@"hello world");
    return 0;
}
```
但是对于App，这就有些不合时宜了。
我们需要程序有序的执行，在有活动的时候迅速反应，在没有任务执行的时候不应该直接退出，而是处于一种待执行、休眠的状态等待事件的执行。
```
int main(int argc, char * argv[]) {
    while (AppIsRunning) {
        id whoWakesMe = SleepForWakingUp();
        id event = GetEvent(whoWakesMe);
        HandleEvent(event);
    }
    return 0;
}
```
不止在iOS/MacOS中有这一概念，node.js中，Windows中都有类似的概念，不过这就不是要讲的了。
[开源代码](https://opensource.apple.com/tarballs/CF/)

## RunLoop in Cocoa
我们用到的一般是 **NSRunLoop**，是位于Foundation框架当中的。而NSRunLoop实际上是 **CFRunLoop**的封装。我们直接查看 **CFRunLoop**就可以。
RunLoop在iOS中可以说应用到了方方面面：
NSTimer、CADisplayLink、AutoreleasePool等系统库都是源自RunLoop，而我们也可以用RunLoop的机制来监控页面卡顿等等。
在iOS/MacOS系统中，RunLoop除了保证程序持续运行外，还成为了App中各种事件的载体，以及帮助我们节省CPU的资源。
![](https://ws3.sinaimg.cn/large/006tNbRwly1fxksgejd2aj30f607ndhj.jpg)
如图所示，runloop依赖于 **Input sources**和 **Timer sources**，**Input sources**提供异步事件，通常是来自另一个线程或来自不同应用程序的消息。**Timer sources**提供同步事件，发生在预定时间或重复间隔。两种类型的源都使用特定于应用程序的处理程序例程来处理事件。

## RunLoop和线程的关系、创建过程
Runloop是基于pthread进行管理的，而pthread是由C语言编写的底层多线程API，在iOS中是 **mach thread**的上层封装，和 **NSThread**一一对应。
在子线程中，并不会直接创建RunLoop，但是提供了两个方法： `CFRunLoopGetMain()`和 `CFRunLoopGetCurrent()`，来获取RunLoop。我们在去第一次获取RunLoop的时候，才会创建对应线程的Runloop，RunLoop和线程一一对应，而在子线程被销毁的时候，对应的RunLoop也被同时销毁掉。当然，主线程就比较特殊了，它是会一直存在的。

```
//--自动获取主线程runloop
CFRunLoopRef CFRunLoopGetMain(void) {
    CHECK_FOR_FORK();
    static CFRunLoopRef __main = NULL; // no retain needed
    if (!__main) __main = _CFRunLoopGet0(pthread_main_thread_np()); // no CAS needed
    return __main;
}
//--自动获取子线程runloop
CFRunLoopRef CFRunLoopGetCurrent(void) {
    CHECK_FOR_FORK();
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
    if (rl) return rl;
    return _CFRunLoopGet0(pthread_self());
}
```
这里我们发现都引用了同一个方法
```
static CFMutableDictionaryRef __CFRunLoops = NULL;
static CFLock_t loopsLock = CFLockInit;
// should only be called by Foundation
// t==0 is a synonym for "main thread" that always 
works
// t == 0 是主线程一直在工作的同义词
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    if (pthread_equal(t, kNilPthreadT)) {
	t = pthread_main_thread_np();
    }
    __CFLock(&loopsLock);
    if (!__CFRunLoops) {
        __CFUnlock(&loopsLock);
	CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
	// 根据传入的主线程获取主线程对应的RunLoop
	CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
	// 保存主线程 将主线程-key和RunLoop-Value保存到字典中
	CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
	if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
	    CFRelease(dict);
	}
	CFRelease(mainLoop);
        __CFLock(&loopsLock);
    }
    
    // 从字典里面拿，将线程作为key从字典里获取一个loop
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    __CFUnlock(&loopsLock);
    
    // 如果loop为空，则创建一个新的loop，所以runloop会在第一次获取的时候创建
    if (!loop) {
	CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        __CFLock(&loopsLock);
	loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
	
	// 创建好之后，以线程为key runloop为value，一对一存储在字典中，下次获取的时候，则直接返回字典内的runloop
	if (!loop) {
	    CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
	    loop = newLoop;
	}
        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        __CFUnlock(&loopsLock);
	CFRelease(newLoop);
    }
    if (pthread_equal(t, pthread_self())) {
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop);
        }
    }
    return loop;
}
```
**__CFRunLoops**是一个全局的字典，key 是 pthread_t， value 是 CFRunLoopRef，存储的是线程和RunLoop。
**loopsLock**是访问dic的锁，是一个 **pthread_mutex_t**锁，互斥锁。
这里我们就明白，可以通过 **[NSRunLoop currentRunLoop];**来获取并创建RunLoop，如果之前创建过，字典中有对应的RunLoop，就不会继续创建。


## RunLoop的结构
从代码上看<!--RunLoop也是一个对象。-->
```
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 唤醒runloop
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread;
    uint32_t _winthread;
    CFMutableSetRef _commonModes;           // 字符串，记录所有标记为common的mode
    CFMutableSetRef _commonModeItems;       // 所有commonMode的item(source、timer、observer)
    CFRunLoopModeRef _currentMode;          // 当前运行的mode
    CFMutableSetRef _modes;                 // 存储的是CFRunLoopModeRef
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};
```
一个RunLoop是包含多个Mode。 **CFRunLoopModeRef**是对应的Mode，我们接着往下看。
```
typedef struct __CFRunLoopMode *CFRunLoopModeRef;

struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
    CFStringRef _name;  //当前Mode的名字
    Boolean _stopped;
    char _padding[3];
    // 核心事件
    CFMutableSetRef _sources0;    // source0 set，非基于Port的，接收点击事件，触摸事件等APP 内部事件
    CFMutableSetRef _sources1;    // source1 set，基于Port的，通过内核和其他线程通信，接收，分发系统事件
    CFMutableArrayRef _observers; // observer 数组
    CFMutableArrayRef _timers;    // timer 数组
    
    
    CFMutableDictionaryRef _portToV1SourceMap;   // source1 对应的端口号
    __CFPortSet _portSet;
    CFIndex _observerMask;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO
    mach_port_t _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```
通过这里我们知道了， **source0**、 **source1**、 **timer**、 **observer**都是由Mode来管理的。
现在我们一般用得到的主要是以下两种Mode。
> * **kCFRunLoopDefaultMode**:App的默认 Mode，通常主线程是在这个 Mode 下运行的
> * **UITrackingRunLoopMode**:界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响。

我们在[这里](http://iphonedevwiki.net/index.php/CFRunLoop)可以查看更多的Mode，但是我们基本上用不到了。

#### RunLoopSource
存在着两个命名非常诡异的source： **source0**、**source1**。
source0 主要负责APP内部事件，如点击事件，触摸事件等；
source1 是基于 **mach_port**信号的，用于通过内核来和其他线程发送消息。我们可以用它在子线程唤醒主线程的RunLoop。

```
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;			/* immutable */
    CFMutableBagRef _runLoops;
    union {
	CFRunLoopSourceContext version0;	/* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */
    } _context;
};
```
我们可以继续查看source0和source1的结构
```
//source0
typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
    Boolean	(*equal)(const void *info1, const void *info2);
    CFHashCode	(*hash)(const void *info);
    void	(*schedule)(void *info, CFRunLoopRef rl, CFStringRef mode);
    void	(*cancel)(void *info, CFRunLoopRef rl, CFStringRef mode);
    void	(*perform)(void *info);
} CFRunLoopSourceContext;

//source1
typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
    Boolean	(*equal)(const void *info1, const void *info2);
    CFHashCode	(*hash)(const void *info);
#if (TARGET_OS_MAC && !(TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)) || (TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)
    mach_port_t	(*getPort)(void *info);
    void *	(*perform)(void *msg, CFIndex size, CFAllocatorRef allocator, void *info);
#else
    void *	(*getPort)(void *info);
    void	(*perform)(void *info);
#endif
} CFRunLoopSourceContext1;
```
我们可以发现source1是比source0多一个 **mach_port_t (*getPort)(void *info);**，用于触发系统回调。

#### CFRunLoopObserver
这个是观察者，用来观察RunLoop的各种状态并发出回调。
```
struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFIndex _rlCount;
    CFOptionFlags _activities;      /* immutable */
    CFIndex _order;         /* immutable */
    CFRunLoopObserverCallBack _callout; /* immutable */
    CFRunLoopObserverContext _context;  /* immutable, except invalidation */
};
```

我们也可以得知，观察的状态分为六种。
```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};

```

#### CFRunLoopTimer
这个是定时器，用来 **在设定好的时间点抛出回调**。
```
struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;  //标记fire状态
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;        //添加该timer的runloop
    CFMutableSetRef _rlModes;     //存放所有 包含该timer的 mode的 modeName，意味着一个timer可能会在多个mode中存在
    CFAbsoluteTime _nextFireDate;
    CFTimeInterval _interval;     //理想时间间隔  /* immutable */
    CFTimeInterval _tolerance;    //时间偏差      /* mutable */
    uint64_t _fireTSR;          /* TSR units */
    CFIndex _order;         /* immutable */
    CFRunLoopTimerCallBack _callout;    /* immutable */
    CFRunLoopTimerContext _context; /* immutable, except invalidation */
};
```
它和 NSTimer 是toll-free bridged 的，可以相互转换。

## RunLoop的流程
借用[iOS刨根问底-深入理解RunLoop](https://www.cnblogs.com/kenshincui/p/6823841.html)的图片来表述一下流程。
![这里](https://images2015.cnblogs.com/blog/62046/201705/62046-20170508103512066-65199905.png)
或者文字表述一遍
> 1.通知观察者已经输入了运行循环。
> 2.通知观察员任何准备好的计时器即将开火。
> 3.通知观察者任何非基于端口的输入源即将触发。
> 4.触发任何准备触发的基于非端口的输入源。
> 5.如果基于端口的输入源准备就绪并等待触发，请立即处理该事件。转到第9步。
> 6.通知观察者线程即将睡眠。
> 7.将线程置于睡眠状态，直到发生以下事件之一：
> * 事件到达基于端口的输入源。
> * 计时器开始。
> * 为运行循环设置的超时值到期。
> * 运行循环被明确唤醒。

> 8.通知观察者线程刚刚醒来。
> 9.处理待处理事件。
> * 如果触发了用户定义的计时器，则处理计时器事件并重新启动循环。转到第2步。
> * 如果输入源被触发，则传递事件。
> * 如果运行循环被明确唤醒但尚未超时，请重新启动循环。转到第2步。

> 10.通知观察者运行循环已退出。

## 销毁RunLoop
线程销毁，runloop自动销毁。
Mode为空的时候退出。
手动设置停止。
```
[NSRunLoop currentRunLoop]runUntilDate:(nonnull NSDate *)
[NSRunLoop currentRunLoop]runMode:(nonnull NSString *) beforeDate:(nonnull NSDate *)

```

## RunLoop的使用须知
主线程的RunLoop事默认开启的，我们不必理会。
在子线程中有以下几种情况是需要开启RunLoop的。
> ping其他线程，唤醒其他线程的时候；
> 在子线程中使用NSTimer和performSelector相关API的时候；
> 需要持续的使用一个线程的时候

如果要启动一个子线程的RunLoop，一定要保证存在source、timer或者observes的存在，不然Runloop是会自动退出的。
另外，CFRunLoop是一个C的API，它是线程安全的；而NSRunLoop是封装过的，但是并不保证线程安全了。

## 应用
#### 长驻线程
比如原来的AFNetWorking使用这个方法来保证线程一直存在。
```
        [[NSThread currentThread] setName:@"AFNetworking"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
```
同样的道理performSelecter:afterDelay方法也要要求线程中存在runloop。

#### NSTimer
NSTimer 其实就是 CFRunLoopTimerRef，他们之间是 toll-free bridged 的。一个 NSTimer 注册到 RunLoop 后，RunLoop 会为其重复的时间点注册好事件。例如 10:00, 10:10, 10:20 这几个时间点。RunLoop为了节省资源，并不会在非常准确的时间点回调这个Timer。Timer 有个属性叫做 Tolerance (宽容度)，标示了当时间点到后，容许有多少最大误差。
这部分相关文章可以查看[iOS倒计时的探究与选择](https://biboyang.github.io/2018/05/iOS_Dev_Note/)这篇文章。



#### CADisplayLink
CADisplayLink是一个能让我们以和屏幕刷新率同步的频率将特定的内容画到屏幕上的定时器类。 CADisplayLink以特定模式注册到runloop后， 每当屏幕显示内容刷新结束的时候，runloop就会向 CADisplayLink指定的target发送一次指定的selector消息， CADisplayLink类对应的selector就会被调用一次。


#### autoreleasePool
当开启或者唤醒runloop的时候，会创建一个autoreleasePool；kCFRunLoopBeforeWaiting | kCFRunLoopExit当runloop睡眠之前或者退出runloop的时候会释放autoreleasePool；
#### 卡顿检测
主线程卡顿监控的实现思路：开辟一个子线程，然后实时计算 kCFRunLoopBeforeSources 和 kCFRunLoopAfterWaiting 两个状态区域之间的耗时是否超过某个阀值，来断定主线程的卡顿情况，可以将这个过程想象成操场上跑圈的运动员，我们会每隔一段时间间隔去判断是否跑了一圈，如果发现在指定时间间隔没有跑完一圈，则认为在消息处理的过程中耗时太多，视为主线程卡顿。


```
static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info)
{
    MyClass *object = (__bridge MyClass*)info;
    
    // 记录状态值
    object->activity = activity;
    
    // 发送信号
    dispatch_semaphore_t semaphore = moniotr->semaphore;
    dispatch_semaphore_signal(semaphore);
}

- (void)registerObserver
{
    CFRunLoopObserverContext context = {0,(__bridge void*)self,NULL,NULL};
    CFRunLoopObserverRef observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                                            kCFRunLoopAllActivities,
                                                            YES,
                                                            0,
                                                            &runLoopObserverCallBack,
                                                            &context);
    CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
    
    // 创建信号
    semaphore = dispatch_semaphore_create(0);
    
    // 在子线程监控时长
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        while (YES)
        {
            // 假定连续5次超时50ms认为卡顿(当然也包含了单次超时250ms)
            long st = dispatch_semaphore_wait(semaphore, dispatch_time(DISPATCH_TIME_NOW, 50*NSEC_PER_MSEC));
            if (st != 0)
            {
                if (activity==kCFRunLoopBeforeSources || activity==kCFRunLoopAfterWaiting)
                {
                    if (++timeoutCount < 5)
                        continue;
                    // 检测到卡顿，进行卡顿上报
                }
            }
            timeoutCount = 0;
        }
    });
}                                                 
```






## 参考
[Run Loops](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1)  

[iOS刨根问底-深入理解RunLoop](https://www.cnblogs.com/kenshincui/p/6823841.html)    

[深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)

[iOS 性能监控 SDK —— Wedjat（华狄特）开发过程的调研和整理](https://github.com/aozhimin/iOS-Monitor-Platform)
