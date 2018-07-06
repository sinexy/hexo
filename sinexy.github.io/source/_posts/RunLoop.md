---
title: RunLoop
date: 2018-07-06 14:18:56
tags:
---

**RunLoop 的概念**
一般来讲，一个县城一次只能执行一个任务，执行完之后线程就会退出。如果我们需要一个机制，让线程能随时处理事件但并不退出，就需要一个 RunLoop 对象。

实现这种模型的关键带你在于：如果管理事件/消息，如何让线程在没有处理消息时休眠以避免资源占用、在有消息到来时立刻被唤醒。

所以，RunLoop 实际上就是一个对象，这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行上面 Event Loop 的逻辑。线程执行了这个函数后，就会一直处于这个函数内部“接受消息->等待->处理”的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。

**RunLoop 与线程的关系**
首先，iOS 开发中能遇到两个线程对象：pthread_t 和 NSThread。苹果不允许直接创建  RunLoop，它只是提供了两个自动获取的函数：`CFRunLoopGetMain()`和`CFRunLoopGetCurrent()`。这两个函数内部的逻辑大概是这样的：
```
// 全局的 Dictionary，key 是 pthread_t，value 是 CFRunLoopRef
static CFMutableDictionaryRef loopsDic;
// 访问 loopsDic 时的锁
static CFSpinLock_t loopsLock;
// 获取一个 pthread 对应的 RunLoop
CFRunLoopRef _CFRunLoopGet(pthread_t thread){
	OSSpinLockLock(&loopsLock);
	
	if(!loopsLook) {
		// 第一次进入时，初始化全局 Dic，并先为主线程创建一个  RunLoop。
		loopsDic = CFDictionaryCreateMutable();
		CFRunLoopRef mainLoop = _CFRunLoopCreate();
		CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mianLoop);
	}
	// 直接从 Dictionary 里获取。
	CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread);
	if (!loop) {
		// 取不到时，创建一个
		loop = _CFRunLoopCreate();
		CFDictionarySetValue(loopsDic, thread, loop);
		// 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop
		_CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);
	}
	OSSpinLockUnLock(&loopsLock);
	return loop;
}
``` 
从上面的代码可以看出，线程和 RunLoop 之间是一一对应的，其关系是保存再一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。

**RunLoop 对外的接口**
在 CoreFoundation 里面关于 RunLoop 有5个类：

* CFRunLoopRef
* CFRunLoopModeRef
* CFRunLoopSourceRef
* CFRunLoopTimerRef
* CFRunLoopObserverRef

其中 CFRunLoopModeRef 类并没有对外暴露，只是通过 CFRunLoopRef 的接口进行了封装。他们的关系如下：
![](https://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_0.png)

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop的主函数时，只能指定其中一个 Mode，这个 Mode 被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。

CFRunLoopSourceRef 是事件产生的地方。







