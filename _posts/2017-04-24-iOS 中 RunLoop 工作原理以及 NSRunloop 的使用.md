---
layout: post
title: iOS 中 RunLoop 工作原理以及 NSRunloop 的使用
date: 2017-02-26
categories: iOS
---

最近看了一下
[深入研究 Runloop 与线程保活](http://www.jianshu.com/p/10121d699c32)
想到几个问题又研究了一下。
# 为什么使用 RunLoop 会造成内存泄漏
这个问题还需要看怎样定义内存泄漏。如果是像，循环引用这种，出乎程序员意料的方式造成内存得不到释放的情况才叫做内存泄漏，那么 RunLoop 不应该被称为内存泄漏。
RunLoop 占用内存不释放，还是跟单利对象占用内存得不到释放归到一类比较好。因为，RunLoop 就是这样设计的——一旦开启 RunLoop 要跟随程序结束而结束，这点是程序员应该明确的。

# RunLoop 为什么需要一直占用内存
![Operating a custom input source](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/Art/custominputsource.jpg)
使用文档上的著名图片来解释，NSRunLoop 需要不断接受另一个线程发送过来的消息，当然不能退出喽。既然不能退出，NSRunLoop 就会一直持有相关联的对象，导致内存不能释放。另外 NSThread 启动子线程也是需要占用内存的，而且不小，我记得是主线程1m，子线程0.5m。
接受消息的具体工作过程，网络上有很多，就不再 Copy 过来了，想了解的可以看看这篇。
[浅谈NSRunloop工作原理和相关应用](http://www.cnblogs.com/ruihaha/p/5813819.html)
## RunLoop 的启动
文档上说了 RunLoop 不能手动初始化，它跟 NSThread 是一起的，属于 NSThread 的 associateObject，如果要获取一个 NSThread 的 RunLoop，必须调用 [NSRunLoop currentRunLoop]。
NSRunLoop 有三种启动方式:
1. Unconditionally
2. With a set time limit
3. In a particular mode
对应的方法分别为：
1. run
2. runUntilDate
3. runMode:beforeDate:

参照文档以及 RunLoop 的源码[源码在此](https://opensource.apple.com/source/CF/CF-635/CFRunLoop.c.auto.html)
``runMode:beforeDate:``
这个方法才是启动一次 RunLoop! 为什么这样说，稍后再解释。
``runUntilDate:``
这个方法，会循环调用 runMode:beforeDate: 直到达到参数 NSDate 所指定的时间，也就是超时时间。
``run``
这个方法，可以看做是
``[runloop runUntilDate:[NSDate distantFuture]];``

## RunLoop 启动后如何接收消息
RunLoop 启动后，必须注册 source（翻译成消息源好不好）。source 有两种，一种是 NSPort 及其子类，另一种是 NSTimer。
使用 NSRunLoop 的消息机制进行线程通信，就依靠 NSPort 及其子类。关键方法在这里：
```
- (void)setup{
    NSLog(@"%@",[[NSRunLoop currentRunLoop] currentMode]);
    //创建并启动一个 thread
    self.thread = [[NSThread alloc]initWithTarget:self selector:@selector(threadTest) object:nil];
    [self.thread setName:@"Test Thread"];
    [self.thread start];

    //向 RunLoop 发送消息的简便方法，系统会将消息传递到指定的 SEL 里面
    [self performSelector:@selector(receiveMsg) onThread:self.thread withObject:nil waitUntilDone:NO];
        
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        //直接去的子线程 RunLoop 的一个 port，并向其发送消息，这个是比较底层的 NSPort 方式进行线程通信
        [self.port sendBeforeDate:[NSDate date] components:[@[[@"hello" dataUsingEncoding:NSUTF8StringEncoding]] mutableCopy] from:nil reserved:0];
    });
}
```

```
- (void)threadTest{
    NSLog(@"%@",@"child thread start");
    //threadTest 这个方法是在 Test Thread 这个线程里面运行的。
    NSLog(@"%@",[NSThread currentThread]);
    //获取这个线程的 RunLoop 并让他运行起来
    NSRunLoop* runloop = [NSRunLoop currentRunLoop];
    self.port = [[NSMachPort alloc]init];
    self.port.delegate = self;
    [runloop addPort:self.port forMode:NSRunLoopCommonModes];
    //约等于runUntilDate:[NSDate distantFuture]
    [runloop run];
}
```

```
- (void)receiveMsg{
    NSLog(@"%@",[NSThread currentThread]);
    NSLog(@"%@",@"receive msg");
}

#pragma NSMachPortDelegate
- (void)handleMachMessage:(void *)msg{
    NSLog(@"handle message thread:%@",[NSThread currentThread]);
}
```

1. 使用 run 方法启动 NSRunLoop，使用 [self performSelector:@selector(receiveMsg) onThread:self.thread withObject:nil waitUntilDone:NO]; 发送消息，- (void)receiveMsg接受消息打印出来的内容：
```
2017-04-25 17:03:19.322693 test[2553:585069] <NSThread: 0x174c76480>{number = 5, name = Test Thread}
2017-04-25 17:03:19.322921 test[2553:585069] receive msg
```

2. 使用 run 方法启动 NSRunLoop 使用[self.port sendBeforeDate:[NSDate date] components:[@[[@"hello" dataUsingEncoding:NSUTF8StringEncoding]] mutableCopy] from:nil reserved:0];发送消息，- (void)handleMachMessage:(void *)msg 接受消息打印出来的内容：
```
2017-04-25 17:03:20.408981 test[2553:585069] handle message thread:<NSThread: 0x174c76480>{number = 5, name = Test Thread}
```
上面是分别使用高层方法和底层方法进行消息通信。

## RunLoop 是如何随时随地接收消息的。
上文说过使用 run 方法启动 RunLoop 后，会循环调用
``runMode:beforeDate:``
这个方法，我想，可以把这个方法称为 1次消息循环。
下面这段代码是 CFRunloop 源码中的一个函数，我猜是1次消息循环的具体实现。大概的工作方式是这样的：
1. 判断这次 RunLoop 的状态(应该对应文档中 NSRunLoop 的 Model)如果不是退出的状态就继续向下执行。
2. 看看有没有超时时间（这个时间应该是runMode:beforeDate:这里的 date）添加一个 timer。
3. 发送进入 RunLoop 的消息。
4. 如果 NSPort 中没有消息，线程进入阻塞状态，交出 CPU。
5. 如果 NSPort 中有消息，就会唤醒这个 RunLoop 所在的线程，RunLoop 继续运行并处理这个消息。
6. 如果 NSPort 中还有消息，就回到3这一步（因为有消息需要处理，线程不会进入阻塞状态，而是一直处理消息）
7. NSPort 中消息处理完毕（或者 timer 超时时间到或者手动退出这个消息循环，也会唤醒这个线程）RunLoop 退出。
这样，一次消息循环结束。

``` C
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    int64_t startTSR = (int64_t)mach_absolute_time();

    if (__CFRunLoopIsStopped(rl)) {
        __CFRunLoopUnsetStopped(rl);
	return kCFRunLoopRunStopped;
    } else if (rlm->_stopped) {
	rlm->_stopped = false;
	return kCFRunLoopRunStopped;
    }

    mach_port_name_t dispatchPort = MACH_PORT_NULL;
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) dispatchPort = _dispatch_get_main_queue_port_4CF();

    dispatch_source_t timeout_timer = NULL;
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context));
    if (seconds <= 0.0) { // instant timeout
        seconds = 0.0;
        timeout_context->termTSR = 0LL;
    } else if (seconds <= TIMER_INTERVAL_LIMIT) {
	dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, DISPATCH_QUEUE_OVERCOMMIT);
	timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        dispatch_retain(timeout_timer);
	timeout_context->ds = timeout_timer;
	timeout_context->rl = (CFRunLoopRef)CFRetain(rl);
	timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
	dispatch_set_context(timeout_timer, timeout_context); // source gets ownership of context
	dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
        dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
        uint64_t nanos = (uint64_t)(seconds * 1000 * 1000 + 1) * 1000;
	dispatch_source_set_timer(timeout_timer, dispatch_time(DISPATCH_TIME_NOW, nanos), DISPATCH_TIME_FOREVER, 0);
	dispatch_resume(timeout_timer);
    } else { // infinite timeout
        seconds = 9999999999.0;
        timeout_context->termTSR = INT64_MAX;
    }

    Boolean didDispatchPortLastTime = true;
    int32_t retVal = 0;
    do {
        uint8_t msg_buffer[3 * 1024];
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
        mach_msg_header_t *msg = NULL;
#elif DEPLOYMENT_TARGET_WINDOWS
        HANDLE livePort = NULL;
        Boolean windowsMessageReceived = false;
#endif
	__CFPortSet waitSet = rlm->_portSet;

	rl->_ignoreWakeUps = false;

        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);

	__CFRunLoopDoBlocks(rl, rlm);

        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
	}

        Boolean poll = sourceHandledThisLoop || (0LL == timeout_context->termTSR);

        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), 0)) {
                goto handle_msg;
            }
#elif DEPLOYMENT_TARGET_WINDOWS
            if (__CFRunLoopWaitForMultipleObjects(NULL, &dispatchPort, 0, 0, &livePort, NULL)) {
                goto handle_msg;
            }
#endif
        }

        didDispatchPortLastTime = false;

	if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
	__CFRunLoopSetSleeping(rl);
	// do not do any user callouts after this point (after notifying of sleeping)

        // Must push the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced.

        __CFPortSetInsert(dispatchPort, waitSet);

	__CFRunLoopModeUnlock(rlm);
	__CFRunLoopUnlock(rl);

#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
        if (kCFUseCollectableAllocator) {
            objc_clear_stack(0);
            memset(msg_buffer, 0, sizeof(msg_buffer));
        }
        msg = (mach_msg_header_t *)msg_buffer;
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), poll ? 0 : TIMEOUT_INFINITY);
#elif DEPLOYMENT_TARGET_WINDOWS
        // Here, use the app-supplied message queue mask. They will set this if they are interested in having this run loop receive windows messages.
        // Note: don't pass 0 for polling, or this thread will never yield the CPU.
        __CFRunLoopWaitForMultipleObjects(waitSet, NULL, poll ? 0 : TIMEOUT_INFINITY, rlm->_msgQMask, &livePort, &windowsMessageReceived);
#endif
        
        __CFRunLoopLock(rl);
        __CFRunLoopModeLock(rlm);

        // Must remove the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced. Also, we don't want them left
        // in there if this function returns.

        __CFPortSetRemove(dispatchPort, waitSet);

	rl->_ignoreWakeUps = true;

        // user callouts now OK again
	__CFRunLoopUnsetSleeping(rl);
	if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

        handle_msg:;
	rl->_ignoreWakeUps = true;

#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
        mach_port_t livePort = msg ? msg->msgh_local_port : MACH_PORT_NULL;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
        if (windowsMessageReceived) {
            // These Win32 APIs cause a callout, so make sure we're unlocked first and relocked after
            __CFRunLoopModeUnlock(rlm);
	    __CFRunLoopUnlock(rl);

            if (rlm->_msgPump) {
                rlm->_msgPump();
            } else {
                MSG msg;
                if (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE | PM_NOYIELD)) {
                    TranslateMessage(&msg);
                    DispatchMessage(&msg);
                }
            }
            
            __CFRunLoopLock(rl);
	    __CFRunLoopModeLock(rlm);
 	    sourceHandledThisLoop = true;
        } else
#endif
        if (MACH_PORT_NULL == livePort) {
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {
            // do nothing on Mac OS
#if DEPLOYMENT_TARGET_WINDOWS
            // Always reset the wake up port, or risk spinning forever
            ResetEvent(rl->_wakeUpPort);
#endif
        } else if (livePort == rlm->_timerPort) {
#if DEPLOYMENT_TARGET_WINDOWS
            // We use a manual reset timer to ensure that we don't miss timers firing because the run loop did the wakeUpPort this time
            // The only way to reset a timer is to reset the timer using SetWaitableTimer. We don't want it to fire again though, so we set the timeout to a large negative value. The timer may be reset again inside the timer handling code.
            LARGE_INTEGER dueTime;
            dueTime.QuadPart = LONG_MIN;
            SetWaitableTimer(rlm->_timerPort, &dueTime, 0, NULL, NULL, FALSE);
#endif
	    __CFRunLoopDoTimers(rl, rlm, mach_absolute_time());
        } else if (livePort == dispatchPort) {
	    __CFRunLoopModeUnlock(rlm);
	    __CFRunLoopUnlock(rl);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
 	    _dispatch_main_queue_callback_4CF(msg);
#elif DEPLOYMENT_TARGET_WINDOWS
            _dispatch_main_queue_callback_4CF();
#endif
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
	    __CFRunLoopLock(rl);
	    __CFRunLoopModeLock(rlm);
 	    sourceHandledThisLoop = true;
            didDispatchPortLastTime = true;
        } else {
            // Despite the name, this works for windows handles as well
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
		mach_msg_header_t *reply = NULL;
		sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
		if (NULL != reply) {
		    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
		    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
		}
#elif DEPLOYMENT_TARGET_WINDOWS
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls) || sourceHandledThisLoop;
#endif
	    }
        } 
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED
        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
#endif
        
	__CFRunLoopDoBlocks(rl, rlm);

	if (sourceHandledThisLoop && stopAfterHandle) {
	    retVal = kCFRunLoopRunHandledSource;
        } else if (timeout_context->termTSR < (int64_t)mach_absolute_time()) {
            retVal = kCFRunLoopRunTimedOut;
	} else if (__CFRunLoopIsStopped(rl)) {
            __CFRunLoopUnsetStopped(rl);
	    retVal = kCFRunLoopRunStopped;
	} else if (rlm->_stopped) {
	    rlm->_stopped = false;
	    retVal = kCFRunLoopRunStopped;
	} else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
	    retVal = kCFRunLoopRunFinished;
	}
    } while (0 == retVal);

    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }

    return retVal;
}
```

而根据源码得知,run 和 runUntilDate 这两个方法会不停的运行上面这个消息循环函数从而做到一直监听消息。
## RunLoop 怎样才能退出呢
上面提到过，可以手动结束一次消息循环，结束的函数就是这个 CFRunLoopStop()。但是要注意，这个函数只是结束一次消息循环，跟 runMode:beforeDate: 中 timer 超时的效果是一样的。也就是停止等待 NSPort 传递消息，唤醒线程，获取 CPU 时间片完成消息循环剩下的任务。
然而 run 和 runUntilDate 这两个方法还会启动下一次消息循环，这也就是用 run 方法启动 RunLoop 后，无法 terminatel 的原因。runUntileDate 会在 date 时间到达时不再启动新的消息循环，而是让 RunLoop 退出。
这样以来，退出 RunLoop 的方式应该明确了。
1. 使用 run 启动 RunLoop，直到程序结束再退出。主线程的 RunLoop 一定是这种方式启动的。
2. 使用 runUntileDate 启动 RunLoop，date 时间到退出。这里的 date 会当做参数传到下面这个方法中，当 date 时间到，RunLoop 也会因为超时而结束一次循环。
3. 使用 runMode:beforeDate: 这种方式启动一次消息循环，并且自己编写代码来控制一次消息循环结束后是否启动下一次消息循环。也就是类似这样的代码：
```
BOOL shouldKeepRunning = YES;        // global
NSRunLoop *theRL = [NSRunLoop currentRunLoop];
while (shouldKeepRunning && [theRL runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]]);
```
4. 还有一种方式，就是文档上说过，如果 RunLoop 中没有 NSPort，就不会启动下一次 RunLoop 了。如果没有 NSPort，RunLoop 进入阻塞状态后谁负责唤醒呢？当然不能启动了对吧。但是文档也说了，这种方式结束 RunLoop循环，通常情况下由于无法持有所有 NSPort 的引用而做不到从 RunLoop 中移除所有 NSPort，是不推荐的方式。
**注意上一段中一次 RunLoop 与 RunLoop 循环的区别，RunLoop 循环指的是一次 RunLoop 运行结束时再启动一次 RunLoop，也就是 runUntileDate: 做的事情。**
**RunLoop 模式可以节省 CPU 的并等待消息的关键就在于 RunLoop 会进入阻塞状态等待 NSPort 的消息将它唤醒，底层是操作系统最基本的功能——线程同步。通常情况下，这种消息是操作系统产生（例如触摸屏幕等事件），也可以由其它线程产生**

看到这里，应该明白 NSRunLoop 的意义以及线程保活的原理了。其实就是建立在操作系统线程同步的基础上，比较底层的一种可以节省 CPU 又能实现进程间通信的一种方式。这种 RunLoop 功能在各种操作系统上都有类似的实现，比如 Android 和 Windows。

# NSRunLoop 的测试
1. run启动，NSMachPort 发送消息
**文档上说了，iOS 只支持 NSMachPort**
```
- (void)setup{
    NSLog(@"%@",[[NSRunLoop currentRunLoop] currentMode]);
    
    
    self.thread = [[NSThread alloc]initWithTarget:self selector:@selector(threadTest) object:nil];
    [self.thread setName:@"Test Thread"];
    [self.thread start];

    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self.port sendBeforeDate:[NSDate date] components:[@[[@"hello" dataUsingEncoding:NSUTF8StringEncoding]] mutableCopy] from:nil reserved:0];
    });
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(6 * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self.port sendBeforeDate:[NSDate date] components:[@[[@"hello" dataUsingEncoding:NSUTF8StringEncoding]] mutableCopy] from:nil reserved:0];
    });
}

- (void)threadTest{
    NSLog(@"%@",@"child thread start");
    NSLog(@"%@",[NSThread currentThread]);
    NSRunLoop* runloop = [NSRunLoop currentRunLoop];
    self.port = [[NSMachPort alloc]init];
    self.port.delegate = self;
    [runloop addPort:self.port forMode:NSRunLoopCommonModes];
    [runloop run];
}

#pragma NSMachPortDelegate
- (void)handleMachMessage:(void *)msg{
    NSLog(@"handle message thread:%@",[NSThread currentThread]);
}
```
输出
```
2017-04-25 18:01:34.011602 test[2633:602345] kCFRunLoopDefaultMode
2017-04-25 18:01:34.013161 test[2633:604579] child thread start
2017-04-25 18:01:34.014156 test[2633:604579] <NSThread: 0x171660280>{number = 5, name = Test Thread}
2017-04-25 18:01:35.098691 test[2633:604579] handle message thread:<NSThread: 0x171660280>{number = 5, name = Test Thread}
2017-04-25 18:01:36.213378 test[2633:604579] handle message thread:<NSThread: 0x171660280>{number = 5, name = Test Thread}
```
1s 和2s 发送的消息都能收到。

2. runUntilDate:启动并发送消息
还是上面的代码
如果
```
[runloop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:4]];
```
RunLoop 运行时间只有4s，收不到6s 时发送的消息，如果将运行时间改长久可以收到了。输出
```
2017-04-25 18:05:57.176821 test[2657:607056] kCFRunLoopDefaultMode
2017-04-25 18:05:57.178433 test[2657:607261] child thread start
2017-04-25 18:05:57.186123 test[2657:607261] <NSThread: 0x17106b0c0>{number = 5, name = Test Thread}
2017-04-25 18:06:00.477071 test[2657:607261] handle message thread:<NSThread: 0x17106b0c0>{number = 5, name = Test Thread}
```
**这里需要注意的是，如果[runloop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:4]];时间改为5，是有可能受到6s 发送的消息的，这里可能是多线程运行时，线程运行时机并不能精确确定导致的。**
3. runMode:beforeDate: 启动并发送消息
```    
[runloop runMode:NSRunLoopCommonModes beforeDate:[NSDate dateWithTimeIntervalSinceNow:15]];
```
输出:
```
2017-04-25 18:11:22.310154 test[2668:609276] kCFRunLoopDefaultMode
2017-04-25 18:11:22.312849 test[2668:609542] child thread start
2017-04-25 18:11:22.314500 test[2668:609542] <NSThread: 0x1758639c0>{number = 5, name = Test Thread}
```
**无论如何也收不到第二个消息，这是由于接受第一个消息是，RunLoop 被唤醒，处理完消息后直接退出了，NSThread 线程也会结束。**


[深入研究 Runloop 与线程保活](http://www.jianshu.com/p/10121d699c32)
[http://www.cocoabuilder.com/archive/cocoa/305204-runloop-not-being-stopped-by-cfrunloopstop.html](http://www.cocoabuilder.com/archive/cocoa/305204-runloop-not-being-stopped-by-cfrunloopstop.html)
[https://opensource.apple.com/source/CF/CF-635/CFRunLoop.c.auto.html](https://opensource.apple.com/source/CF/CF-635/CFRunLoop.c.auto.html)

另外，NSObject 有一个方法
``performSelector: onThread: withObject: waitUntilDone: modes:``
waitUntilDone这个参数可以指定在子线程运行时阻塞当前线程。具体实现原理可以参考下面这两个链接。
这里虽然是 Linux 的实现，但是 iOS Linux Windows 都是遵循 POSIX 标准的。
[之前介绍 thread join和detach的区别但是不详细 （详细介绍)](http://blog.csdn.net/gyafdxis/article/details/49620809)
[pthread_join和pthread_detach的用法（转）](http://www.cnblogs.com/sanchrist/p/3566313.html)