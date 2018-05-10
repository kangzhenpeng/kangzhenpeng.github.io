---
title: RunLoop回顾总结之 Thread-Loop Store
date: 2018-05-08 20:42:16
tags: [RunLoop, Thread]
categories: 
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

# 撸代码看设计 
## RunLoop与线程之间的关系
thread - runloop 是一对一 key-value 存储在进程中的全局Dict __CFRunLoops中。
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
### 辅助函数 - TLS 操作函数 
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
### 流程图
![loop 创建流图](../../../../images/runLoop/manageThreadLoop.png)
![mainLoop 获取流程](../../../../images/runLoop/getMainLoop.png)
![currentLoop 获取流程](../../../../images/runLoop/getCurrentLoop.png)
### 总结

 - Thread 与 RunLoop 是一对一的，且存储在一个全局的字典中。
 - 主线程的 RunLoop 是默认创建的，其他的线程的 RunLoop 是用时创建，懒加载的。
 - 线程的 RunLoop 并不是每次都是从全局字典中获取的。这样可以加快获取的过程，有利于性能的提升。
    - 主线程的 RunLoop 在第一次是从全局字典获取，之后直接使用全局变量 __main。
    - 其他线程的 RunLoop 在第一次是从全局字典获取，之后直接从 TLS 中获取。

## 未完，待续...

# 引用
 - https://opensource.apple.com/tarballs/CF/
 - https://blog.csdn.net/ssirreplaceable/article/details/53793456
 - https://blog.ibireme.com/2015/05/18/runloop/
 - https://blog.csdn.net/ssirreplaceable/article/details/53793456
 - https://stackoverflow.com/questions/47260563/whats-the-meaning-of-check-for-fork

