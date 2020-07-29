---
title: RunLoop原理探究
date: 2020-07-29 11:17:19
tags:
---



### 1、RunLoop 概念

顾名思义，RunLoop 就是运行循环，在程序运行过程中循环做一些事情。这种模型通常被称作 [Event Loop](https://en.wikipedia.org/wiki/Event_loop)，这种模型的关键点在于：如何管理事件/消息，如果保证线程在没有处理消息时休眠避免资源占用、再有消息到来时立刻被唤醒。

一把来说，一个线程一次只能执行一次任务，执行完成后线程就会退出，如果要保证线程能随时处理事件但不退出，我们可能需要这样做：

```objective-c
int main(int argc, char * argv[]) {
    @autoreleasepool {
        int retVal = 0;
        do {
            // 睡眠中等待消息
            int message = sleep_and_wait();
            // 处理消息
            retVal = process_message(message);
        } while (retVal == 0);
        return 0;
    }
}
```

RunLoop 其实就是一个对象，这个对象管理了其需要处理的消息和事件，并且提供了一个入口函数来执行上面的 Event Loop 逻辑。线程执行了这个函数后，会一直处于：接受消息->等待->处理消息 的循环中，直到 retVal 为0才会退出函数。

可以在[这里](http://opensource.apple.com/tarballs/CF/)下载到 CoreFoundation的源码来查看具体实现。

### 2、RunLoop 结构

iOS 有两套 API 来访问和使用 RunLoop：`NSRunLoop`、`CFRunLoop`。NSRunLoop 是基于 CFRunLoop 的包装

Core Foundation 中关于 RunLoop 的类有5个：

1. CFRunLoopRef
2. CFRunLoopModeRef
3. CFRunLoopSourceRef
4. CFRunLoopTimerRef
5. CFRunLoopObserverRef

CFRunLoopRef：

```c++

struct __CFRunLoop {
    pthread_t _pthread;
    CFMutableSetRef _commonModes;			// set
    CFMutableSetRef _commonModeItems; // set<Source、Observer、Timer>
    CFRunLoopModeRef _currentMode;		// current mode
    CFMutableSetRef _modes;						// set
};
```

CFRunLoopModeRef：

```C++
struct __CFRunLoopMode {
    CFStringRef _name; // mode name：such as - kCFRunLoopDefaultMode
    CFMutableSetRef _sources0;		// set
    CFMutableSetRef _sources1;		// set
    CFMutableArrayRef _observers;	// array
    CFMutableArrayRef _timers;		// array
};
```

这里有个概念叫做 commonModes：一个 mode 可以将自己标记为 common 属性（通过将其 mode name 添加到 RunLoop 的 commonModes 中）。每当 RunLoop 内容发生时，RunLoop 会自动将 commonModeItems 里面的 source/observer/timer 同步到具有 common 标记的所有 mode 里面。

CFRunLoop 对外暴露的管理 mode 的接口有两个：

```C++
CFRunLoopAddCommonMode(CFRunLoopRef runloop, CFStringRef modeName);
CFRunLoopRunInMode(CFStringRef modeName, ...);
```

mode 暴露的管理 mode item 的接口有以下几个

```c++
CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);
CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);
CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);
CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);
CFRunLoopRemoveTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
```

这里只能通过 mode name 来操作内部的 mode，当传入一个新的 mode name 单 RunLoop 里面没有对应的 mode 时，RunLoop 内部会自动创建 CFRunLoopModeRef。对于一个 Runloop 来说，内部的 mode 只能增加，不能删除。

苹果公开的 mode 有两个

1. kCFRunLoopDefaultMode（NSDefaultRunLoopMode）
2. UITrackingRunLoopMode

这里还提供了一个操作 common 标记符的字符串 kCFRunLoopCommonModes (NSRunLoopCommonModes)。使用时注意这个标记和其他的 mode name。

看下 RunLoop 图示结构：

![RunLoop结构](https://blog-key.oss-cn-beijing.aliyuncs.com/blog/runloop/runloop%E7%BB%93%E6%9E%84.png)



从上图的结构可以看到，一个 RunLoop 包含若干个 mode，每个 mode 有包含若干个 source/observer/timer，每次调用 RunLoop 的主函数时，只能指定其中一个 mode 作为 currentMode。如果需要切换 mode，只能退出当前 loop，再重新指定一个 mode 进入。这样做是为了不同组的 source/observer/timer，可以做到互不影响。



**CFRunLoopSourceRef** 是事件产生的地方。source 有两个版本：source0 和 source1

- source0：





### 3、RunLoop 与线程的关系

每条线程都有唯一一个与之对应的 RunLoop 对象，他们是一一对应的。

线程在刚创建时是没有 RunLoop 对象的，RunLoop 会在第一次获取它时创建，我们通常获取的方式有以下几种：

```objective-c
CFRunLoopGetCurrent();
CFRunLoopGetMain();
[NSRunLoop currentRunLoop];
[NSRunLoop mainRunLoop];
```

来看下获取的源码，已 getCurrent 为例：

```c
// 全局的 Dictionary，key 是 pthread_t，value 是 CFRunLoopRef
static CFMutableDictionaryRef __CFRunLoops = NULL;
static CFLock_t loopsLock = CFLockInit;	// 访问 __CFRunLoops 时的锁

// should only be called by Foundation
// t==0 is a synonym for "main thread" that always works
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    // 线程为空则获取主线程
    if (pthread_equal(t, kNilPthreadT)) {
        t = pthread_main_thread_np();
    }
    // 如果全局的 CFMutableDictionaryRef 为空，则先初始化全局的 CFMutableDictionaryRef，并先为主线程创建一个 RunLoop
    if (!__CFRunLoops) {
        CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
        CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
        CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
    }
    // 直接从全局 CFMutableDictionaryRef 里面获取
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    
    // 获取不到则创建一个
    if (!loop) {
        CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
    }
    
    // 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop
    _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop);
    
    return loop;
}
```

有源码分析可知：

- 线程和 RunLoop 是一一对应的
- RunLoop 保存在一个全局的 Dictionary 里面，pthread_t 为 key，RunLoop 为 value
- 线程刚创建时并没有 RunLoop，RunLoop 会在第一次获取它时创建
- RunLoop 会在线程结束时销毁
- 只能在线程内部获取 RunLoop（主线程除外）



### 4、RunLoop 的运行状态







































