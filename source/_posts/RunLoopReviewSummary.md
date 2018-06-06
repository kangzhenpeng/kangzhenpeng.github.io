---
title: RunLoop 回顾总结
date: 2018-05-08 20:42:16
tags: [iOS, RunLoop, Thread]
categories: 
- iOS
- RunLoop
- 设计
toc: true   

---
# 概念
一般来说线程在处理完任务后，就会结束。为了让线程可以持续接受任务并执行，就需要有一个循环来持续接受消息并处理。通常称这个循环为EventLoop，这种模式在大多数系统中是类似的。

```   
function eventLoop() {
    initialize(); // 内部初始化
        do{
            let msg = getNextMsg(); // 获取下一个消息/事件
            processMsg(msg); // 执行消息/事件
        } while (msg != quit); // 循环知道退出消息到达
}
    
```
RunLoop 是在 OSX/iOS中，苹果对EventLoop的一种实现。有NSRunLoop、CFRunLoop两种可用。其中NSRunLoop是对CFRunLoop的一种封装。

# 应用
<figure>
    <img src="http://p8pq9azjn.bkt.clouddn.com/image/runloop/RunLoopApply.png!blogPicStyle">
</figure>

# 框架图
<figure>
    <img src="http://p8pq9azjn.bkt.clouddn.com/image/runloop/RunLoopStructureChart.png!blogPicStyle">
</figure>

# 代码分析
## Loop - Thread Relation
 - Thread 与 RunLoop 是一对一的，且存储在一个全局的字典中。
 - 主线程的 RunLoop 是默认创建的，其他线程的 RunLoop 是用时创建，懒加载的。
 - 线程的 RunLoop 并不是每次都是从全局字典中获取的，而是从全局或者 TLS 中获取。这样可以加快获取的过程，有利于性能的提升。
    - 主线程的 RunLoop 在第一次是从全局字典获取，之后直接使用全局变量 __main。
    - 其他线程的 RunLoop 在第一次是从全局字典获取，之后直接从 TLS 中获取。
 - CFRunLoop 中使用了锁来保证线程安全。NSRunLoop 是基于 CFRunLoop 封装的，没有设计线程安全。

### 流程图
<figure>
    <img src="http://p8pq9azjn.bkt.clouddn.com/image/runloop/ThreadLoopRelation.png!blogPicStyle">
</figure>

### 核心代码  
`CFRunLoop.c line:1353`
```
static pthread_t kNilPthreadT = (pthread_t)0; // 初始化  kNilPthreadT 为空线程
static CFMutableDictionaryRef __CFRunLoops = NULL; // 声明全局的一个字典变量并初始化为空。
static CFLock_t loopsLock = CFLockInit; // 声明一个全局的锁，用来对 loops 保证线程安全。

// 获取 runnLoop 的函数。
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    if (pthread_equal(t, kNilPthreadT)) {
        t = pthread_main_thread_np();
    } // 入参线程为空，则默认使用主线程。

    __CFLock(&loopsLock); // 操作 loops 前加锁。

    if (!__CFRunLoops) { // 判断全局字典 loops 是否存在。
        __CFUnlock(&loopsLock); // 解锁。
    
        CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks); // 创建一个临时可变字典。
        CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np()); // 创建 mainLoop。
        CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop); // 将 mainLoop 与 mainThread 对应存储在临时的可变字典中。
        // 将 dict 赋值给 loops，内部有锁。
        if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) { 
            CFRelease(dict); // 释放临时变量dict。
        }
    
        CFRelease(mainLoop); // 释放mainLoop，因为已经在 dict retain 了。
        __CFLock(&loopsLock); // 加锁。
    }

    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t)); // 从 loops 中获取线程 t 的 loop
    __CFUnlock(&loopsLock); // 解锁。

    // loops 中没有线程 t 的 loop
    if (!loop) { 
        CFRunLoopRef newLoop = __CFRunLoopCreate(t); // 创建线程 t 的 loop。
        __CFLock(&loopsLock); // 解锁。
        loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t)); // 再次从 loops 中获取线程 t 的 loop，减小碰撞几率。
        // 二次校验 loops 中有没有线程 t 的 loop。
        if (!loop) {
            CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop); // 将创建的 newloop 存储在 loops 中。
            loop = newLoop; // 赋值给 loop。
        }
        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        __CFUnlock(&loopsLock); // 解锁。
        CFRelease(newLoop); // 释放 newLoop。
    }

    if (pthread_equal(t, pthread_self())) { // 线程 t 是否是当前线程
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL); // TLS(线程局部存储) 存储线程 t 的 loop，稍后会有详细代码展示。
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop); // TLS 存储线程销毁和 loop 销毁，实现线程销毁的时候，loop 也销毁。
        }
    }
    return loop; // 返回线程 t 的 loop。
}

// 获取主线程的 mainLoop 函数
CFRunLoopRef CFRunLoopGetMain(void) {
    CHECK_FOR_FORK(); // 检查当前进程是否有 fork，有则退出。
    // 创建全局变量 _mian，方便在其他线程中获取主线程的 loop。
    static CFRunLoopRef __main = NULL; // no retain needed
    // 不存在，则获取主线程的 mainLoop。
    if (!__main) __main = _CFRunLoopGet0(pthread_main_thread_np()); // no CAS needed
    return __main;
}

// 获取当前线程的 loop 函数
CFRunLoopRef CFRunLoopGetCurrent(void) {
    CHECK_FOR_FORK(); // 检查当前进程是否有 fork，有则退出。
    // TLS 中获取当前线程的 loop。
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
    if (rl) return rl;
    // 不存在，则获取当前线程的 loop。
    return _CFRunLoopGet0(pthread_self());
}
```

### 辅助代码
#### TLS 操作函数 
`CFPlatform.c  line: 648`
```
// 存储函数
CF_EXPORT void *_CFSetTSD(uint32_t slot, void *newVal, tsdDestructor destructor) {
    if (slot > CF_TSD_MAX_SLOTS) { // 大于最大槽个数，则报错
        _CFLogSimple(kCFLogLevelError, "Error: TSD slot %d out of range (set)", slot);
        HALT;
    }
    // 获取 TSL 中的 table
    __CFTSDTable *table = __CFTSDGetTable();
    if (!table) { // table 不存在，报错
        // Someone is setting TSD during thread destruction. The table is gone, so we can't get any data anymore.
        _CFLogSimple(kCFLogLevelWarning, "Warning: TSD slot %d set but the thread data has already been torn down.", slot);
        return NULL;
    }

    void *oldVal = (void *)table->data[slot]; // 按 key 从 data 中获取之前的值
    
    table->data[slot] = (uintptr_t)newVal; // 存储新值
    table->destructors[slot] = destructor; // 存储析构器
    
    return oldVal; // 返回数据变更之前的值
}

// 读取函数
CF_EXPORT void *_CFGetTSD(uint32_t slot) {
    if (slot > CF_TSD_MAX_SLOTS) { // 大于最大槽个数，则报错
        _CFLogSimple(kCFLogLevelError, "Error: TSD slot %d out of range (get)", slot);
        HALT;
    }
    // 获取 TSL 中的 table
    __CFTSDTable *table = __CFTSDGetTable();
    if (!table) { // table 不存在，则报错
        // Someone is getting TSD during thread destruction. The table is gone, so we can't get any data anymore.
        _CFLogSimple(kCFLogLevelWarning, "Warning: TSD slot %d retrieved but the thread data has already been torn down.", slot);
        return NULL;
    }
    uintptr_t *slots = (uintptr_t *)(table->data);
    return (void *)slots[slot]; // 按 key 从 data 中获取值并返回
}

// 获取 TLS 的 table
static __CFTSDTable *__CFTSDGetTable() {
    // 获取 TLS 的 table
    __CFTSDTable *table = (__CFTSDTable *)__CFTSDGetSpecific();
    // Make sure we're not setting data again after destruction.
    if (table == CF_TSD_BAD_PTR) { // 被析构了
        return NULL;
    }
    // Create table on demand
    if (!table) {
        // This memory is freed in the finalize function
        table = (__CFTSDTable *)calloc(1, sizeof(__CFTSDTable)); // 申请 table 的内存空间
        pthread_key_init_np(CF_TSD_KEY, __CFTSDFinalize); // 析构绑定 
        __CFTSDSetSpecific(table); // 存储 table
    }
    
    return table; //返回 table
}

// table 获取函数
static void *__CFTSDGetSpecific() {
    return _pthread_getspecific_direct(CF_TSD_KEY);
}
```

## Loop Run
 - RunLoop 实质上是一个 do...While 的循环，通过 port trap 消息来执行。
 - RunLoop 有五种运行 mode，每个 mode 中有多个 modeItem。
    - kCFRunLoopDefaultMode: 默认运行 mode。
    - UITrackingRunLoopMode: 界面追踪 mode，用户追踪 scrollView 的滑动。
    - UIInitializationRunLoopMode: 启动初始化 mode，只在该 mode 下运行一次。
    - GSEventReceiveRunLoopMode：系统内部事件的 mode。
    - kCFRunLoopCommonModes：一个集合，集合其他 mode。
    - ModeItem: Source/Timer/Observer 的统称，没有实际的数据结构。
    - CommonModes: 标记为 common 的 mode，这些 mode 的 modeItem 是相同的。可以使得 RunLoop 可以看似在多个 mode 下共同运行。
    - CommonModeItems: common mode 下运行的事件集合。
 - 事件类型：
    - Timer: 与 NSTimer 是 Toll-Free Bridging 的，基于时间的触发源。多个 Timer 对应一个modeQueuePort。
    - Source0: 只有 block，没有 port，用户级事件。多个 source0 对应一个 wakeupPort，当 source0 被标记后，会通过 wakeupPort 唤醒执行。
    - Source1: 有 blok 和 port，系统级事件。
    - Observer: 对 loop 过程的一个监听。
        - KCFRunLoopEntry: loop 进入。
        - KCFRunLoopBeforeTimers: 开始处理Timers。
        - KCFRunLoopBeforeSources: 开始处理Source。
        - KCFRunLoopBeforeWaiting: 开始休眠。
        - KCFRunLoopAfterWaiting: 结束休眠。
        - KCFRunLoopExit: loop 退出。
 - RunLoop 在处理完 source0 后多空转一次，是为了保证source0被确切的执行完毕。
 - RunLoop 只能在单一 Mode 下运行，切换 mode 后会保存之前 mode 的运行数据。新 mode 下运行结束后，并不会继续之前的 mode 下重新 run，但是会还原之前的运行数据。
 - RunLoop 的 block 是用单向链表存储的，在第一次运行完 block 的时候会变更为环形链表，便于后续 block 执行的查找。block 猜测是 dispatch 过来的。
 - perform...selector 最后对应的是一个定时器事件。

### 流程图
<figure>
    <img src="http://p8pq9azjn.bkt.clouddn.com/image/runloop/run.png!blogPicStyle">
</figure>

### 核心代码
`CFRunLoop.c line:2649`
```
// mode 切换函数 
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK(); // 进程检查
    // 检查loop是否被析构
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl); // 加锁
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false); // 查找对应 modeName 的 mode 且对 mode 加锁
    // 判断 mode 是否有效
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
        Boolean did = false;
        if (currentMode) __CFRunLoopModeUnlock(currentMode); // mode 解锁
        __CFRunLoopUnlock(rl); // loop 解锁
        return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished; // 返回运行结果
    }
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl); // 记录当前 mode 的运行数据，并将当前数据置位，准备给新的 mode 使用
    CFRunLoopModeRef previousMode = rl->_currentMode; // 记录当前 mode
    rl->_currentMode = currentMode; // 变更 loop 的 mode 为目标 mode
    int32_t result = kCFRunLoopRunFinished; // 初始化一个运行结果
    
    if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry); // 发出 entry 通知
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode); // 执行 run 函数
    if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit); // 发出 exit 通知
    
    __CFRunLoopModeUnlock(currentMode); // mode 解锁
    __CFRunLoopPopPerRunData(rl, previousPerRun); // 将记录的之前的 mode 的运行数据还原
    rl->_currentMode = previousMode; // 将记录的之前的 mode 还原
    __CFRunLoopUnlock(rl); // loop 解锁
    return result; // 返回目标 mode 的运行结果
}

// Run 函数，已经把架构指令集适配代码简化掉了
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    uint64_t startTSR = mach_absolute_time(); // 获取机器时间
    
    if (__CFRunLoopIsStopped(rl)) { // 判断 loop 是够已经停止
        __CFRunLoopUnsetStopped(rl); // 置位状态
        return kCFRunLoopRunStopped; // 返回状态
    } else if (rlm->_stopped) { // 对应的 mode 是否已经停止
        rlm->_stopped = false; // 置位状态
        return kCFRunLoopRunStopped; // 返回状态
    }
    
    mach_port_name_t dispatchPort = MACH_PORT_NULL; // 给dispatch 初始化一个空端口
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ))); // 判断是否第一次配发到主线程
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) dispatchPort = _dispatch_get_main_queue_port_4CF(); // 在主线程下，会给 dispatch 配置一个 port
    
    mach_port_name_t modeQueuePort = MACH_PORT_NULL; // 给 modeQueue 初始化一个空 port，查看全文代码，queue 里面只是存储了 timer
    if (rlm->_queue) { // queue 存在
        modeQueuePort = _dispatch_runloop_root_queue_get_port_4CF(rlm->_queue); // 配置一个 port
        if (!modeQueuePort) { // 配置 port 失败，报错
            CRASH("Unable to get port for run loop mode queue (%d)", -1);
        }
    }
    
    dispatch_source_t timeout_timer = NULL; // 初始化一个空timer，用来做 loop 超时监测的
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context)); // 超时上下文内存初始化
    if (seconds <= 0.0) { // instant timeout
        seconds = 0.0;
        timeout_context->termTSR = 0ULL; // 超时时间为0
    } else if (seconds <= TIMER_INTERVAL_LIMIT) { // 判断是够在限制时间间隔之内
        dispatch_queue_t queue = pthread_main_np() ? __CFDispatchQueueGetGenericMatchingMain() : __CFDispatchQueueGetGenericBackground(); // 根据线程分配队列
        timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue); // 创建定时器
        dispatch_retain(timeout_timer); // 增加引用计数
        // 配置上下文
        timeout_context->ds = timeout_timer; // 设置 timer
        timeout_context->rl = (CFRunLoopRef)CFRetain(rl); // 设置 loop
        timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds); // 设置时间
        // 设置上下文
        dispatch_set_context(timeout_timer, timeout_context); // source gets ownership of context
        // 设置超时操作
        dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
        // 设置取消操作
        dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
        uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
        // 设置超时定时器
        dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        // 启动定时器
        dispatch_resume(timeout_timer);
    } else { // infinite timeout
        seconds = 9999999999.0; 
        timeout_context->termTSR = UINT64_MAX; // 设置最大超时时间
    }
    
    Boolean didDispatchPortLastTime = true; // 初始化 dispatch port 的标记
    int32_t retVal = 0; // 运行结果初始化
    // 进入主体事件循环
    do {
        voucher_mach_msg_state_t voucherState = VOUCHER_MACH_MSG_STATE_UNCHANGED;
        voucher_t voucherCopy = NULL;
        uint8_t msg_buffer[3 * 1024]; // 初始化一个 msg 的缓冲

        mach_msg_header_t *msg = NULL; // msg
        mach_port_t livePort = MACH_PORT_NULL; // 当前活动的 port，初始化为 null
        __CFPortSet waitSet = rlm->_portSet; // 待监听的 port 的集合
        
        __CFRunLoopUnsetIgnoreWakeUps(rl); // 设置可唤醒状态
        
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers); // 通知要处理 timers
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources); // 处理要处理 sources, 这里是 source0
        
        __CFRunLoopDoBlocks(rl, rlm); // 执行 block，从代码上猜测应该是 dispatch 塞进的 block，使用双向链表保存
        
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle); // 执行可以执行的 source0
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm); // 执行block
        }
        
        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR); // source 被执行完 或者 loop 超时，需要再空转一圈，来保证 source0 被彻底执行完，因为 source0 不具备主动唤醒能力，只能被动执行
        
        // 第一次不会执行
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) { // 是否是 dispatch 的 port 
            msg = (mach_msg_header_t *)msg_buffer; // 缓冲中取 msg
            // 监听 port 中消息
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg; // 跳转到消息执行体
            }
        }
        
        didDispatchPortLastTime = false; // 置位
        
        // 通知即将要休眠
        if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        __CFRunLoopSetSleeping(rl); // 设置 sleep 状态
        // do not do any user callouts after this point (after notifying of sleeping)
        
        // Must push the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced.
        
        __CFPortSetInsert(dispatchPort, waitSet); // 收敛所有的 port
        
        __CFRunLoopModeUnlock(rlm); // 解锁 mode
        __CFRunLoopUnlock(rl); // 解锁 loop
        
        CFAbsoluteTime sleepStart = poll ? 0.0 : CFAbsoluteTimeGetCurrent(); // 记录休眠开始时间点
        
        // 进入内部监听循环体，只要跳出该循环体，则代表 loop 被唤醒
        do {
            if (kCFUseCollectableAllocator) {
                // objc_clear_stack(0);
                // <rdar://problem/16393959>
                memset(msg_buffer, 0, sizeof(msg_buffer)); // 清空缓冲区
            }
            msg = (mach_msg_header_t *)msg_buffer;
            
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy); // 监听 port 集合
            
            // 是否为 modeQueuePort，用来处理 timers 
            if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) { 
                // Drain the internal queue. If one of the callout blocks sets the timerFired flag, break out and service the timer.
                while (_dispatch_runloop_root_queue_perform_4CF(rlm->_queue)); // 遍历 modeQueue，判断是否有 timer 到时了
                // 到时，则置位，且跳出内部循环体
                if (rlm->_timerFired) { 
                    // Leave livePort as the queue port, and service timers below
                    rlm->_timerFired = false; // 置位
                    break;
                } else { // 继续读取 msg
                    if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
                }
            } else { // 非 modeQueuePort 的 port msg 被捕获，则跳出内部循环体
                // Go ahead and leave the inner loop.
                break;
            }
        } while (1);
        
        __CFRunLoopLock(rl); // 加锁 loop
        __CFRunLoopModeLock(rlm); // 加锁 mode
        
        rl->_sleepTime += (poll ? 0.0 : (CFAbsoluteTimeGetCurrent() - sleepStart)); // 记录自己的休眠时间
        
        // Must remove the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced. Also, we don't want them left
        // in there if this function returns.
        
        __CFPortSetRemove(dispatchPort, waitSet); // 将 dispatchPort 从监听集合中移除
        
        __CFRunLoopSetIgnoreWakeUps(rl); // 设置忽略唤醒，因为已经醒来了
        
        // user callouts now OK again
        __CFRunLoopUnsetSleeping(rl); // 置位 sleep 标记
        // 通知结束休眠
        if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting); 
        
        // 消息执行体
    handle_msg:;
        __CFRunLoopSetIgnoreWakeUps(rl); // 设置忽略唤醒，对应 handle_msg 之前的设置
        
        if (MACH_PORT_NULL == livePort) { // 是否为空 port
            CFRUNLOOP_WAKEUP_FOR_NOTHING(); // 标记自己被唤醒的原因是“啥事也没发生”，一种兜底方案吧
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) { // 是否为唤醒 port 
            CFRUNLOOP_WAKEUP_FOR_WAKEUP(); // 被唤醒 port 唤醒
            // do nothing on Mac OS
        }
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) { // 是否为 modeQueuePort
            CFRUNLOOP_WAKEUP_FOR_TIMER(); // 标记为被 Timer 事件唤醒
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) { // 根据机器事件执行 timer，但为成功
                // Re-arm the next timer, because we apparently fired early
                __CFArmNextTimerInMode(rlm, rl); // 再次执行 timer 
            }
        }
        else if (livePort == dispatchPort) { // 是否为 dispatch 的 port
            CFRUNLOOP_WAKEUP_FOR_DISPATCH(); // 标记唤醒的原因是 dispatch
            __CFRunLoopModeUnlock(rlm); // 解锁 mode
            __CFRunLoopUnlock(rl); // 解锁 loop
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);f
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
            __CFRunLoopLock(rl); // 加锁 loop
            __CFRunLoopModeLock(rlm); // 加锁 mode
            sourceHandledThisLoop = true;  // 标记执行了 source
            didDispatchPortLastTime = true; // 标记 dispatch 的 port 换新被执行完
        } else {
            CFRUNLOOP_WAKEUP_FOR_SOURCE(); // 标记唤醒的原因是 source
            
            // If we received a voucher from this mach_msg, then put a copy of the new voucher into TSD. CFMachPortBoost will look in the TSD for the voucher. By using the value in the TSD we tie the CFMachPortBoost to this received mach_msg explicitly without a chance for anything in between the two pieces of code to set the voucher again.
            voucher_t previousVoucher = _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, (void *)voucherCopy, os_release);
            
            // Despite the name, this works for windows handles as well
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort); // 获取对应 port 的 source1
            if (rls) {
                mach_msg_header_t *reply = NULL;
                // 执行 source1
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
                if (NULL != reply) {
                    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
                    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
                }
            }
            
            // Restore the previous voucher
            _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, previousVoucher, os_release);
            
        }
        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
        
        __CFRunLoopDoBlocks(rl, rlm); // 执行 block
        
        if (sourceHandledThisLoop && stopAfterHandle) {
            retVal = kCFRunLoopRunHandledSource; // 处理完事件的运行结果
        } else if (timeout_context->termTSR < mach_absolute_time()) {
            retVal = kCFRunLoopRunTimedOut; // 超时运行结果
        } else if (__CFRunLoopIsStopped(rl)) {
            __CFRunLoopUnsetStopped(rl); // 置位
            retVal = kCFRunLoopRunStopped; // 停止的运行结果
        } else if (rlm->_stopped) { 
            rlm->_stopped = false; // 置位
            retVal = kCFRunLoopRunStopped; // 停止的运行结果
        } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) { // mode 已经被执行完了
            retVal = kCFRunLoopRunFinished; // 结束的运行结果
        }
        voucher_mach_msg_revert(voucherState);
        os_release(voucherCopy);
        
    } while (0 == retVal); // 外层循环执行的条件判断
    
    // 释放超时定时器
    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }
    
    return retVal;
}
```

### 辅助代码
#### Loop 运行数据 push/pop 函数
`CFRunLoop.c line:660`
```
// push 函数
CF_INLINE volatile _per_run_data *__CFRunLoopPushPerRunData(CFRunLoopRef rl) {
    volatile _per_run_data *previous = rl->_perRunData; // 保存之前的运行数据
    rl->_perRunData = (volatile _per_run_data *)CFAllocatorAllocate(kCFAllocatorSystemDefault, sizeof(_per_run_data), 0); // 重新创建一个运行数据
    // 对运行数据做初始化置位
    rl->_perRunData->a = 0x4346524C;
    rl->_perRunData->b = 0x4346524C; // 'CFRL'
    rl->_perRunData->stopped = 0x00000000;
    rl->_perRunData->ignoreWakeUps = 0x00000000;
    return previous; // 返回当前的运行数据
}

// pop 函数
CF_INLINE void __CFRunLoopPopPerRunData(CFRunLoopRef rl, volatile _per_run_data *previous) {
    // 判断当前的运行数据是否存在
    if (rl->_perRunData) CFAllocatorDeallocate(kCFAllocatorSystemDefault, (void *)rl->_perRunData // 存在，则销毁
    rl->_perRunData = previous; // 切换运行数据为上一次的运行数据
}
```

#### Observer/Timer/Source 操作函数
操作函数基本类似，这里以 observer 的操作为例做代码展开。
`CFRunLoop.c line:1668`
```
// 执行 Observer
static void __CFRunLoopDoObservers(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFRunLoopActivity activity) {	/* DOES CALLOUT */
    CHECK_FOR_FORK(); // 进程 fork 校验
    
    CFIndex cnt = rlm->_observers ? CFArrayGetCount(rlm->_observers) : 0; // observer 个数获取
    if (cnt < 1) return; // 没有则退出
    
    /* Fire the observers */
    STACK_BUFFER_DECL(CFRunLoopObserverRef, buffer, (cnt <= 1024) ? cnt : 1);
    CFRunLoopObserverRef *collectedObservers = (cnt <= 1024) ? buffer : (CFRunLoopObserverRef *)malloc(cnt * sizeof(CFRunLoopObserverRef)); // 创建一个 observer 的集合
    CFIndex obs_cnt = 0;
    for (CFIndex idx = 0; idx < cnt; idx++) {
        CFRunLoopObserverRef rlo = (CFRunLoopObserverRef)CFArrayGetValueAtIndex(rlm->_observers, idx);
        if (0 != (rlo->_activities & activity) && __CFIsValid(rlo) && !__CFRunLoopObserverIsFiring(rlo)) {
            collectedObservers[obs_cnt++] = (CFRunLoopObserverRef)CFRetain(rlo); // 将未被执行的有效的 observer 保存到 observer 的集合中
        }
    }
    __CFRunLoopModeUnlock(rlm); // mode 解锁
    __CFRunLoopUnlock(rl); // loop 解锁
    for (CFIndex idx = 0; idx < obs_cnt; idx++) {
        CFRunLoopObserverRef rlo = collectedObservers[idx];
        __CFRunLoopObserverLock(rlo); // observer 加锁
        if (__CFIsValid(rlo)) {
            Boolean doInvalidate = !__CFRunLoopObserverRepeats(rlo);
            __CFRunLoopObserverSetFiring(rlo); // 设置执行状态
            __CFRunLoopObserverUnlock(rlo); // observer 解锁
            __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(rlo->_callout, rlo, activity, rlo->_context.info); // 执行 observer
            if (doInvalidate) {
                CFRunLoopObserverInvalidate(rlo); // 设置失效状态
            }
            __CFRunLoopObserverUnsetFiring(rlo); // 清除执行状态
        } else {
            __CFRunLoopObserverUnlock(rlo); // observer 解锁
        }
        CFRelease(rlo);
    }
    __CFRunLoopLock(rl); // loop 加锁
    __CFRunLoopModeLock(rlm); // mode 加锁
    
    if (collectedObservers != buffer) free(collectedObservers);
}

// 添加 observer
void CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef rlo, CFStringRef modeName) {
    CHECK_FOR_FORK(); // 进程 fork 校验
    CFRunLoopModeRef rlm;
    if (__CFRunLoopIsDeallocating(rl)) return; // loop 析构的时候退出
    if (!__CFIsValid(rlo) || (NULL != rlo->_runLoop && rlo->_runLoop != rl)) return; // observer 和 loop 校验 
    __CFRunLoopLock(rl); // loop 加锁
    if (modeName == kCFRunLoopCommonModes) { // common 标记的 mode 的处理
        // 获取 common 标记的 mode 的集合
        CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
        // 获取 common 标记的 modeItem 的集合
        if (NULL == rl->_commonModeItems) {
            rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
        }
        // 将 observer 添加到 common 标记的 modeItem 的集合中
        CFSetAddValue(rl->_commonModeItems, rlo);
        if (NULL != set) {
            CFTypeRef context[2] = {rl, rlo};
            /* add new item to all common-modes */
            CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
            CFRelease(set);
        }
    } else {
        rlm = __CFRunLoopFindMode(rl, modeName, true); // 查询 mode
        // observer 的集合校验，为空则创建
        if (NULL != rlm && NULL == rlm->_observers) {
            rlm->_observers = CFArrayCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeArrayCallBacks);
        }
        // 查询 observer 是否已经存在，不存在，则添加
        if (NULL != rlm && !CFArrayContainsValue(rlm->_observers, CFRangeMake(0, CFArrayGetCount(rlm->_observers)), rlo)) {
            Boolean inserted = false;
            for (CFIndex idx = CFArrayGetCount(rlm->_observers); idx--; ) {
                CFRunLoopObserverRef obs = (CFRunLoopObserverRef)CFArrayGetValueAtIndex(rlm->_observers, idx);
                if (obs->_order <= rlo->_order) {
                    CFArrayInsertValueAtIndex(rlm->_observers, idx + 1, rlo);
                    inserted = true;
                    break;
                }
            }
            // 不存在，则添加
            if (!inserted) {
                CFArrayInsertValueAtIndex(rlm->_observers, 0, rlo);
            }
            rlm->_observerMask |= rlo->_activities;
            __CFRunLoopObserverSchedule(rlo, rl, rlm); // 对 observer 中的记录的 loop count 做变更
        }
        if (NULL != rlm) {
            __CFRunLoopModeUnlock(rlm); // mode 解锁
        }
    }
    __CFRunLoopUnlock(rl); // loop 解锁
}

// 移除 observer
void CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef rlo, CFStringRef modeName) {
    CHECK_FOR_FORK(); // 进程 fork 校验
    CFRunLoopModeRef rlm;
    __CFRunLoopLock(rl); // loop 加锁
    if (modeName == kCFRunLoopCommonModes) { // 是否为 common 标记的 mode 
        // common 标记的 mode 集合中是否存在目标 observer
        if (NULL != rl->_commonModeItems && CFSetContainsValue(rl->_commonModeItems, rlo)) {
            CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
            CFSetRemoveValue(rl->_commonModeItems, rlo); // 从 common 标记的 modeItem 的集合中移除目标 observer
            if (NULL != set) {
                CFTypeRef context[2] = {rl, rlo};
                /* remove new item from all common-modes */
                CFSetApplyFunction(set, (__CFRunLoopRemoveItemFromCommonModes), (void *)context);
                CFRelease(set);
            }
        } else {
        }
    } else {
        rlm = __CFRunLoopFindMode(rl, modeName, false); // 查询 mode
        // observer 集合校验
        if (NULL != rlm && NULL != rlm->_observers) {
            CFRetain(rlo);
            CFIndex idx = CFArrayGetFirstIndexOfValue(rlm->_observers, CFRangeMake(0, CFArrayGetCount(rlm->_observers)), rlo); // 查找 observer 
            if (kCFNotFound != idx) {
                CFArrayRemoveValueAtIndex(rlm->_observers, idx);
                __CFRunLoopObserverCancel(rlo, rl, rlm); // 变更 observer 中记录的 loop count 
            }
            CFRelease(rlo);
        }
        if (NULL != rlm) {
            __CFRunLoopModeUnlock(rlm); // mode 解锁
        }
    }
    __CFRunLoopUnlock(rl); // loop 解锁
}
```

#### Block 操作函数
`CFRunLoop.c line:1618`
```
// 执行 block 函数
static Boolean __CFRunLoopDoBlocks(CFRunLoopRef rl, CFRunLoopModeRef rlm) { // Call with rl and rlm locked
    // blocks 链表校验
    if (!rl->_blocks_head) return false;
    // mode 校验
    if (!rlm || !rlm->_name) return false;
    Boolean did = false;
    // 获取链表的表头
    struct _block_item *head = rl->_blocks_head;
    // 获取链表的表尾
    struct _block_item *tail = rl->_blocks_tail;
    rl->_blocks_head = NULL;
    rl->_blocks_tail = NULL;
    CFSetRef commonModes = rl->_commonModes;
    CFStringRef curMode = rlm->_name;
    __CFRunLoopModeUnlock(rlm);
    __CFRunLoopUnlock(rl);
    struct _block_item *prev = NULL;
    struct _block_item *item = head; // 从头开始遍历
    while (item) {
        struct _block_item *curr = item;
        item = item->_next;
        Boolean doit = false;
        // 校验是否执行
        if (CFStringGetTypeID() == CFGetTypeID(curr->_mode)) {
            doit = CFEqual(curr->_mode, curMode) || (CFEqual(curr->_mode, kCFRunLoopCommonModes) && CFSetContainsValue(commonModes, curMode));
        } else {
            doit = CFSetContainsValue((CFSetRef)curr->_mode, curMode) || (CFSetContainsValue((CFSetRef)curr->_mode, kCFRunLoopCommonModes) && CFSetContainsValue(commonModes, curMode));
        }
        // 不执行，则还原
        if (!doit) prev = curr;
        // 执行，则继续向后遍历
        if (doit) {
            // 变更链表的头尾
            if (prev) prev->_next = item;
            if (curr == head) head = item;
            if (curr == tail) tail = prev;
            void (^block)(void) = curr->_block;
            CFRelease(curr->_mode);
            free(curr);
            // 执行 block
            if (doit) {
                __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
                did = true;
            }
            Block_release(block); // do this before relocking to prevent deadlocks where some yahoo wants to run the run loop reentrantly from their dealloc
        }
    }
    __CFRunLoopLock(rl);
    __CFRunLoopModeLock(rlm);
    // 环形链表变更
    if (head) {
        tail->_next = rl->_blocks_head;
        rl->_blocks_head = head;
        if (!rl->_blocks_tail) rl->_blocks_tail = tail;
    }
    return did;
}

// block 存储函数
void CFRunLoopPerformBlock(CFRunLoopRef rl, CFTypeRef mode, void (^block)(void)) {
    CHECK_FOR_FORK();
    // mode 类型的判断
    if (CFStringGetTypeID() == CFGetTypeID(mode)) {
        mode = CFStringCreateCopy(kCFAllocatorSystemDefault, (CFStringRef)mode);
        __CFRunLoopLock(rl);
        // ensure mode exists
        CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, (CFStringRef)mode, true); // 查询 mode，没有则创建
        if (currentMode) __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopUnlock(rl);
    } else if (CFArrayGetTypeID() == CFGetTypeID(mode)) {
        CFIndex cnt = CFArrayGetCount((CFArrayRef)mode);
        const void **values = (const void **)malloc(sizeof(const void *) * cnt);
        CFArrayGetValues((CFArrayRef)mode, CFRangeMake(0, cnt), values);
        mode = CFSetCreate(kCFAllocatorSystemDefault, values, cnt, &kCFTypeSetCallBacks);
        __CFRunLoopLock(rl);
        // ensure modes exist
        for (CFIndex idx = 0; idx < cnt; idx++) {
            CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, (CFStringRef)values[idx], true);
            if (currentMode) __CFRunLoopModeUnlock(currentMode);
        }
        __CFRunLoopUnlock(rl);
        free(values);
    } else if (CFSetGetTypeID() == CFGetTypeID(mode)) {
        CFIndex cnt = CFSetGetCount((CFSetRef)mode);
        const void **values = (const void **)malloc(sizeof(const void *) * cnt);
        CFSetGetValues((CFSetRef)mode, values);
        mode = CFSetCreate(kCFAllocatorSystemDefault, values, cnt, &kCFTypeSetCallBacks);
        __CFRunLoopLock(rl);
        // ensure modes exist
        for (CFIndex idx = 0; idx < cnt; idx++) {
            CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, (CFStringRef)values[idx], true);
            if (currentMode) __CFRunLoopModeUnlock(currentMode);
        }
        __CFRunLoopUnlock(rl);
        free(values);
    } else {
        mode = NULL;
    }
    block = Block_copy(block);
    // 校验 mode 和 block
    if (!mode || !block) {
        if (mode) CFRelease(mode);
        if (block) Block_release(block);
        return;
    }
    __CFRunLoopLock(rl);
    // 链表节点创建，表尾添加
    struct _block_item *new_item = (struct _block_item *)malloc(sizeof(struct _block_item));
    new_item->_next = NULL;
    new_item->_mode = mode;
    new_item->_block = block;
    if (!rl->_blocks_tail) {
        rl->_blocks_head = new_item;
    } else {
        rl->_blocks_tail->_next = new_item;
    }
    rl->_blocks_tail = new_item;
    __CFRunLoopUnlock(rl);
}
```

# 关联知识
## Toll-Free Bridging

- Toll-Free Bridging 是指在多个框架中数据类型可以无缝切换。在 iOS 中是指 CoreFoundation 和 Foundation 两个框架之间数据类型的转换。
- 不是所有者两个框架中的数据类型都可以相互转换。以下是不能相互转换的
    - NSRunLoop 和 CFRunLoop 
    - NSBundle 和 CFBundle
    - NSDateFormatter 和 CFDateFormatter
- MRC 下不涉及内存管理的转移，可以直接转换。
- ARC 下涉及内存管理的转移，在转换时需要指定内存管理的所有权。
    - __bridge: 不改变内存管理方式。
        - CF -> F: F 的内存由编译器管理，CF 的内存管理由开发者管理。
        - F -> CF: F 的内存由编译器管理，CF 没有被 retain，不需要处理。
    - __bridge_reatained: 解决 __bridge 下 F -> CF 时，F 被释放，CF 也就被释放，再使用就会出现内存泄露的问题。
        - F 的内存有编译器管理，CF 会被编译器 retain，内存需要由开发者管理。
        - 由于 CF 被 retain，再使用就不会出现 __bridge 无法解决的内存泄漏问题了。
    - __bridge_transfer: 解决 __bridge 下 CF -> F 时，复杂的中间变量和 CF 的内存管理问题而做的简化处理。
        - F 的内存由编译器管理
        - CF 的内存由编译器转移了，开发者不需要再处理。
        
## 进程/线程间通信
- 线程间使用 NSPort 通信 
- 进程间使用 NSTask 通信

## 线程安全
- 多个线程在临界区产生竞态条件，即会引起线程安全问题。
- 多发生在写操作时产生线程安全问题。
- 容易引起线程安全问题的资源：
    - 共享资源
    - 局部对象引用：对象放在共享堆中，可以被多个线程使用，引用是不被共享的。`局部变量不会引起线程安全问题，因为存放在线程的栈中。`
    - 文件、数据库
- 通过加锁来解决线程安全问题。

# 引用
 - [源码](https://opensource.apple.com/tarballs/CF/)
 - [关于RunLoop部分源码的注释](https://blog.csdn.net/ssirreplaceable/article/details/53793456)
 - [深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)
 - [RunLoop学习笔记](http://blog.gocy.tech/2016/09/03/runloop-source-reading/)
 - [老司机出品——源码解析之RunLoop详解](https://www.jianshu.com/p/ec629063390f)
 - [Check_For_Fork](https://stackoverflow.com/questions/47260563/whats-the-meaning-of-check-for-fork)
 - [Toll-Free Bridging](https://www.jianshu.com/p/c53f2eb116ae)
 - [iOS线程通信和进程通信的例子](https://blog.csdn.net/yxh265/article/details/51483822)
 - [iOS-线程安全探究](https://www.jianshu.com/p/4c50e04c82c7)

