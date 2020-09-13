---
title: Crash捕获
date: 2020-09-13 13:23:51
tags: crash 捕获
toc: false

---



前段时间在做皮肤适配的时候，遇到了一些问题，在某些机型上面，如果 hook setBackgroundColor 方法，某些情况下回概率性的崩溃。在 goolgle 上面搜索很多资料，尝试了一些方法，但是仍然会有少部分机型概率性的崩溃。再加上其他模块，比如书架等也会偶现数据库数据异常的问题。针对这些写了一个小的工具，就是当 App 在短时间内连续 crash 超过两次，下次用户进入APP时会提示用户是否重置账号，其实 APP的工作就是还原数据和屏蔽一些可能导致 crash 的功能。（PS: 其实还可以通过开关来控制，但是开关不是本部门维护的，所以... ... 懂得，自己来才靠谱）



### 关于 crash 做了两种监听

- NSException 异常捕获
- Signal 异常

```objective-c
static NSUncaughtExceptionHandler * originalUncaughtExceptionHandler;
- (void)_registerExceptionHandler {
    if (NSGetUncaughtExceptionHandler()) {
        originalUncaughtExceptionHandler = NSGetUncaughtExceptionHandler();
    }
    NSSetUncaughtExceptionHandler(&uncaughtExceptionHandler);
}
// sigaction --
- (void)_registerSignalHandler {
    // 由 abort 函数调用发生的终止信号
    signal(SIGABRT, signalHandler);
    // 由内存地址未对齐导致的终止信号
    signal(SIGBUS, signalHandler);
    // 由浮点数异常导致的终止信号
    signal(SIGFPE, signalHandler);
    // 由非法指令产生的终止信号
    signal(SIGILL, signalHandler);
    // 通过端口发送消息失败导致的终止信号
    signal(SIGPIPE, signalHandler);
    // 无效的内存导致的终止信号
    signal(SIGSEGV, signalHandler);
}
```



此处关于捕获需要注意判断之前是否已经有人再捕获，有的话需要保留之前捕获的现场。

每次收到异常后更新时间戳，然后在下次APP启动时判断是否需要重置账号信息：

```objective-c
- (void)_recordContinuouslyCrash {
    
    NSUserDefaults * userDefaults = [NSUserDefaults standardUserDefaults];
    NSDictionary * storageInfo = [userDefaults objectForKey:kContinuouslyCrash];
    NSMutableDictionary * crashInfo = [NSMutableDictionary dictionary];
    if (storageInfo) {
        [crashInfo addEntriesFromDictionary:storageInfo];
    }
    NSDate * lateCrashTime = [crashInfo objectForKey:kContinuouslyCrashLateTime];
    NSDate * nowTime = [NSDate date];
    
    // 如果 5 分钟内有连续的两次 crash，则需要进入 reset 状态
    if (lateCrashTime && (nowTime.timeIntervalSince1970 - lateCrashTime.timeIntervalSince1970 <= 5 * 60)) {
        [crashInfo setObject:@(YES) forKey:kContinuouslyCrashNeedReset];
    }
    
    [crashInfo setObject:nowTime forKey:kContinuouslyCrashLateTime];
    [userDefaults setObject:crashInfo forKey:kContinuouslyCrash];
    [userDefaults synchronize];
}


/// 是否有连续的 crash
- (BOOL)hasContinuouslyCrash {
    BOOL hasContinuouslyCrash = NO;
    NSDictionary * crashInfo = [[NSUserDefaults standardUserDefaults] objectForKey:kContinuouslyCrash];
    if (crashInfo) {
        id value = [crashInfo objectForKey:kContinuouslyCrashNeedReset];
        if ([value respondsToSelector:@selector(boolValue)]) {
            hasContinuouslyCrash = [value boolValue];
        }
    }
    return hasContinuouslyCrash;
}
```



[Demo地址](https://github.com/jueying-xiangfeng/recordCrashLog)

