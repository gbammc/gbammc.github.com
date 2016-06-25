---
layout: post
title: "iOS 的多线程原理、分类与应用"
date: 2013-10-06 23:58
comments: true
categories: iOS
keywords: iOS,多线程,原理,GCD
description: 今天查资料才发现，iOS 中的线程使用不是无限制的，官方文档给出的资料显示 iOS 下的主线程堆栈大小是 1M，第二个线程开始都是 512KB，并且该值不能通过编译器开关或线程 API 函数来更改。
---

今天查资料才发现，iOS 中的线程使用不是无限制的，官方文档给出的[资料](https://developer.apple.com/library/mac/documentation/cocoa/conceptual/Multithreading/CreatingThreads/CreatingThreads.html)显示 iOS 下的主线程堆栈大小是 1M，第二个线程开始都是 512KB，并且该值不能通过编译器开关或线程 API 函数来更改。另外只有主线程有直接修改 UI 的能力。所以也学习并总结下 iOS 的多线程编程来加深下吧。

* [关于 RunLoop](#runloop)
* [NSThread](#nsthread)
* [NSOperationQueue 和 NSOperation](#nsoperation)
* [GCD](#gcd)
* [NSOperationQueue 与 GCD 的对比](#compare)

---

## <span id="runloop">关于 RunLoop</span>

首先关于 RunLoop，iOS 中的 RunLoop 准确的说是线程中的循环。首先循环体的开始需要检测是否有需要处理的事件，如果有则去处理，如果没有则进入睡眠以节省 CPU 时间。 所以重点便是这个需要处理的事件，在 RunLoop 中，需要处理的事件分两类，一种是输入源，一种是定时器。定时器好理解，就是那些需要定时执行的操作；输入源分三类：performSelector 源，基于端口（Mach port）的源，以及自定义的源。而RunLoop 在每一次循环的开始便去检查这些事件源是否有需要处理的数据，有的话则去处理。

系统会自动为应用程序的主线程生成一个与之对应的 Runloop 来处理其消息循环。在触摸 UIView 时之所以能够触发 touchesBegan/touchesMoved 等等函数被调用，就是因为应用程序的主线程在 UIApplicationMain 里面有这样一个 Runloop 在分发 input 或 timer 事件。

而每一个线程都有其对应的 Runloop，但是默认非主线程的 Runloop 是没有运行的，需要为 Runloop 添加至少一个事件源，然后去 run 它。一般情况下我们是没有必要去启用线程的 Runloop 的，除非你在一个单独的线程中需要长久的检测某个事件。

iOS 中的多线程编程主要分以下三类：**1.NSThread**；**2.NSOperation/NSOperationQueue**；**3.GCD**。后两者其实都是对 NSThreads 的调用再进行一次封装，以便开发人员更容易使用 iOS 中的多线程编程。而对于 NSOperation/NSOperationQueue 和 GCD 的比较，支持者们意见不太统一，应该适时选择合适的。这里也附上 StackOverflow 上的[讨论](http://stackoverflow.com/questions/10373331/nsoperation-vs-grand-central-dispatch)(本文后面也有列出大概原因)情况。

---

## <span id="nsthread">NSThread</span>

* 优点：NSThread 比其他两个轻量级；
* 缺点：需要自己管理线程的生命周期，线程同步。线程同步对数据的加锁会有一定的系统开销。

### NSThread的使用

``` objc
- (id)initWithTarget:(id)target selector:(SEL)selector object:(id)argument
+ (void)detachNewThreadSelector:(SEL)aSelector toTarget:(id)aTarget withObject:(id)anArgument
```

第一个是实例方法，第二个是类方法。

``` objc
// 类方法
[NSThread detachNewThreadSelector:@selector(doSomething:) toTarget:self withObject:nil];

// 实例方法的声明
NSThread* myThread = [[NSThread alloc] initWithTarget:self
                                        selector:@selector(doSomething:)
                                        object:nil];
// 实例方法的调用
[myThread start];
```

上面两种方式的区别是：前一种一调用就会立即创建一个线程来做事情；而后一种虽然你 alloc 了也 init 了，但是要直到我们手动调用 ```start``` 启动线程时才会真正去创建线程。这种延迟实现思想在很多跟资源相关的地方都有用到。后一种方式我们还可以在启动线程之前，对线程进行配置，比如设置 stack 大小，线程优先级。

还有一种间接的方式，更加方便，我们甚至不需要显式编写 NSThread 相关代码。那就是利用 NSObject 的类方法 ```performSelectorInBackground:withObject:``` 来创建一个线程：

``` objc
[myObj performSelectorInBackground:@selector(myThreadMainMethod) withObject:nil];
```

其效果与 NSThread 的 ```detachNewThreadSelector:toTarget:withObject:``` 是一样的

### 线程同步

线程的同步方法跟其他系统下类似，我们可以用原子操作，可以用 mutex，lock 等。

iOS 的原子操作函数是以 OSAtomic 开头的，比如：OSAtomicAdd32，OSAtomicOr32 等等。这些函数可以直接使用，因为它们是原子操作。

iOS中的 mutex 对应的是 NSLock，它遵循 NSLooking 协议，我们可以使用 `lock`，`tryLock`，`lockBeforeData：`来加锁，用 `unLock` 来解锁。使用示例：

``` objc
BOOL moreToDo = YES;
NSLock *theLock = [[NSLock alloc] init];
...
while (moreToDo) {
    /* Do another increment of calculation */
    /* until there’s no more to do. */
    if ([theLock tryLock]) {
        /* Update display used by all threads. */
        [theLock unlock];
    }
}
```

我们可以使用指令 `@synchronized` 来简化 NSLock 的使用，这样我们就不必显示编写创建 NSLock，加锁并解锁相关代码。

``` objc
- (void)myMethod:(id)anObj
{
    @synchronized(anObj)
    {
        // Everything between the braces is protected by the @synchronized directive.
    }
}
```

还有其他的一些锁对象，比如：循环锁 NSRecursiveLock，条件锁 NSConditionLock，分布式锁 NSDistributedLock 等等，在这里就不一一介绍了。

### 用 NSCodition 同步执行的顺序

NSCodition 是一种特殊类型的锁，我们可以用它来同步操作执行的顺序。它与 mutex 的区别在于更加精准，等待某个 NSCondtion 的线程一直被 lock，直到其他线程给那个 condition 发送了信号。下面我们来看使用示例：

``` objc
// 某个线程等待着事情去做，而有没有事情做是由其他线程通知它的
[cocoaCondition lock];
while (timeToDoWork <= 0) [cocoaCondition wait];

timeToDoWork--;
// Do real work here.

[cocoaCondition unlock];

//其他线程发送信号通知上面的线程可以做事情了：
[cocoaCondition lock];
timeToDoWork++;
[cocoaCondition signal];
[cocoaCondition unlock];
```

### 线程间通信

线程在运行过程中，可能需要与其它线程进行通信。我们可以使用 NSObject 中的一些方法：

``` objc
// 在应用程序主线程中做事情：
performSelectorOnMainThread:withObject:waitUntilDone:
performSelectorOnMainThread:withObject:waitUntilDone:modes:

// 在指定线程中做事情：
performSelector:onThread:withObject:waitUntilDone:
performSelector:onThread:withObject:waitUntilDone:modes:

// 在当前线程中做事情：
performSelector:withObject:afterDelay:
performSelector:withObject:afterDelay:inModes:

// 取消发送给当前线程的某个消息
cancelPreviousPerformRequestsWithTarget:
cancelPreviousPerformRequestsWithTarget:selector:object:
```

如在我们在某个线程中下载数据，下载完成之后要通知主线程中更新界面等等，可以使用如下接口：

``` objc
- (void)callMainThreadMethod
{
    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
    // to do something in your thread job
    ...
    [self performSelectorOnMainThread:@selector(updateUI) withObject:nil waitUntilDone:NO];
    [pool release];
}
```

### 隐式调用

用 NSObject 的类方法 ```performSelectorInBackground:withObject: ``` 创建一个线程：

``` objc
[myObj performSelectorInBackground:@selector(doSomething) withObject:nil];
```

---

## <span id="nsoperation">NSOperationQueue 和 NSOperation</span>

多线程编程是防止主线程堵塞，增加运行效率等等的最佳方法。而原始的多线程方法存在很多的毛病，包括线程锁死等。在 Cocoa 中，Apple 提供了 NSOperation 这个类，提供了一个优秀的多线程编程方法。

NSOperationQueue 会建立一个线程管理器，每个加入到线程 operation 会有序的执行。

用 NSOperationQueue 的过程：

1.  建立一个 NSOperationQueue 的对象；
2.  建立一个 NSOperation 的对象；
3.  将 operation 加入到 NSOperationQueue 中；
4.  release 掉 operation。

本次介绍 NSOperation 的子集，简易方法的 NSInvocationOperation：

``` objc
@implementation MyCustomClass
- (void)launchTaskWithData:(id)data
{
    //创建一个NSInvocationOperation对象，并初始化到方法
    //在这里，selector参数后的值是你想在另外一个线程中运行的方法（函数，Method）
    //在这里，object后的值是想传递给前面方法的数据
    NSInvocationOperation* theOp = [[NSInvocationOperation alloc] initWithTarget:self
                    selector:@selector(myTaskMethod:) object:data];
    // 下面将我们建立的操作“Operation”加入到本地程序的共享队列中（加入后方法就会立刻被执行）
    // 更多的时候是由我们自己建立“操作”队列
    [[MyAppDelegate sharedOperationQueue] addOperation:theOp];
}
// 这个是真正运行在另外一个线程的“方法”
- (void)myTaskMethod:(id)data
{
    // Perform the task.
}
@end
```

一个 NSOperationQueue 操作队列，就相当于一个线程管理器，而非一个线程。因为你可以设置这个线程管理器内可以并行运行的的线程数量等等。下面是建立并初始化一个操作队列：

``` objc
@interface MyViewController : UIViewController {
    NSOperationQueue *operationQueue;
    //在头文件中声明该队列
}
@end
@implementation MyViewController
- (id)init
{
    self = [super init];
    if (self) {
        operationQueue = [[NSOperationQueue alloc] init]; //初始化操作队列
        [operationQueue setMaxConcurrentOperationCount:1];
        //在这里限定了该队列只同时运行一个线程
        //这个队列已经可以使用了
    }
    return self;
}
- (void)dealloc
{
    [operationQueue release];
    //正如Alan经常说的，我们是程序的好公民，需要释放内存！
    [super dealloc];
}
@end
```

---

## <span id="gcd">GCD</span>

GCD 是一个替代诸如 NSThread，NSOperationQueue，NSInvocationOperation 等技术的很高效和强大的技术，它看起来象就其它语言的闭包（Closure）一样，但苹果把它叫做 block。

### GCD的定义

简单 GCD 的定义有点象函数指针，差别是用 ^ 替代了函数指针的 * 号，如下所示：

``` objc
// 申明变量
 (void) (^loggerBlock)(void);
 // 定义
 loggerBlock = ^{
      NSLog(@"Hello world");
 };
 // 调用
 loggerBlock();
```

但是大多数时候，我们通常使用内联的方式来定义它，即将它的程序块写在调用的函数里面，例如这样：

``` objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
     // something
});
```

从上面大家可以看出，block 有如下特点：

* 程序块可以在代码中以内联的方式来定义；
* 程序块可以访问在创建它的范围内的可用的变量。

为了方便地使用 GCD，苹果提供了一些方法方便我们将 block 放在主线程或后台线程执行，或者延后执行。使用的例子如下：

``` objc
//  后台执行：
 dispatch_async(dispatch_get_global_queue(0, 0), ^{
      // something
 });
 // 主线程执行：
 dispatch_async(dispatch_get_main_queue(), ^{
      // something
 });
 // 一次性执行：
 static dispatch_once_t onceToken;
 dispatch_once(&onceToken, ^{
     // code to be executed once
 });
 // 延迟2秒执行：
 double delayInSeconds = 2.0;
 dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, delayInSeconds * NSEC_PER_SEC);
 dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
     // code to be executed on the main queue after delay
 });
// dispatch_queue_t 也可以自己定义，如要要自定义queue，可以用dispatch_queue_create方法，示例如下：

dispatch_queue_t urls_queue = dispatch_queue_create("blog.devtang.com", NULL);
dispatch_async(urls_queue, ^{
     // your code
});
dispatch_release(urls_queue);
```

### 后台运行

GCD 的另一个用处是可以让程序在后台较长久的运行。在没有使用 GCD 时，当 app 被按 home 键退出后，app 仅有最多 5 秒钟的时候做一些保存或清理资源的工作。但是在使用 GCD 后，app 最多有 10 分钟的时间在后台长久运行。这个时间可以用来做清理本地缓存，发送统计数据等工作。

另外，GCD 还有一些高级用法，例如让后台 2 个线程并行执行，然后等 2 个线程都结束后，再汇总执行结果。这个可以用 dispatch_group，dispatch_group_async 和 dispatch_group_notify 来实现，示例如下：

``` objc
dispatch_group_t group = dispatch_group_create();
dispatch_group_async(group, dispatch_get_global_queue(0,0), ^{
      // 并行执行的线程一
 });
 dispatch_group_async(group, dispatch_get_global_queue(0,0), ^{
      // 并行执行的线程二
 });
 dispatch_group_notify(group, dispatch_get_global_queue(0,0), ^{
      // 汇总结果
 });
```

让程序在后台长久运行的示例代码如下:

``` objc
// AppDelegate.h文件
@property (assign, nonatomic) UIBackgroundTaskIdentifier backgroundUpdateTask;

// AppDelegate.m文件
- (void)applicationDidEnterBackground:(UIApplication *)application
{
    [self beingBackgroundUpdateTask];
    // 在这里加上你需要长久运行的代码
    [self endBackgroundUpdateTask];
}

- (void)beingBackgroundUpdateTask
{
    self.backgroundUpdateTask = [[UIApplication sharedApplication] beginBackgroundTaskWithExpirationHandler:^{
        [self endBackgroundUpdateTask];
    }];
}

- (void)endBackgroundUpdateTask
{
    [[UIApplication sharedApplication] endBackgroundTask: self.backgroundUpdateTask];
    self.backgroundUpdateTask = UIBackgroundTaskInvalid;
}
```

## <span id="compare">NSOperationQueue 与 GCD 的对比</span>

对于 NSOperationQueue 和 GCD 应该用哪个，一般来说可以用编程界比较通用的原则来决定：

	Always use the highest-level abstraction available to you, and drop down to lower-level abstractions when measurement shows that they are needed.

简意是：尽可能用更高级抽象的方法。但前面提到的 StackOverflow 里的讨论里却分别说出了两者的优缺点：

__NSOperation 好处：__

* 很容易设置两个 NSOperation 之间的依赖来让某一个操作在上一个操作完成后才执行（establishing dependencies between operations）；
* 方便设置在同一时间运行的操作个数（bandwidth-constrained queues that only run N operations at a time）；
* 您可以创建操作，支持在第一时间被取消（you can create operations that support being cancelled in the first place）。

__GCD 好处：__

* NSOperation 对象在创建或释放过程中会消耗明显的CPU资源（The NSOperation object allocation and deallocation process took a significant amount of CPU resources when dealing with small and frequent actions, like rendering an OpenGL ES frame to the screen. GCD blocks completely eliminated that overhead, leading to significant performance improvements）；
* 使用 block 后代码比使用 NSOperation 更简洁（code is cleaner when using blocks than NSOperations）。

当然，具体使用哪一个还是要看你的使用场合了。

