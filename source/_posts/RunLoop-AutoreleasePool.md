---
title: AutoreleasePool 回顾总结
date: 2018-05-15 16:13:36    
tags: [autorelease, autoreleasePool]   
categories:  
- iOS   
- RunLoop   
- 应用  
toc: true

---

## 概念
AutoreleasePool 是 OC 中的一种内存延迟释放机制。正常情况下变量在其作用域结束便被释放，如果使用 autoreleasePool 包裹，则会延迟释放。除非手动干预 autoreleasePool 的释放，否则将在 runLoop 的休眠和退出的时机释放。

## 整体设计
App 在启动后向主线程注册了两个Observer，监听 RunLoop 的 Entry/BeforeWaiting/Exit 三个时机。

- Observer1: 负责监听 entry 状态，创建 autoreleasePool(poolPush)，在32位上优先级是-(2^32-1)，最高优先级，保证后续所有操作都在这个 pool 中。
- Observer2: 负责监听 beforeWaiting 和 exit 两个状态。优先级是 (2^32-1)，最低优先级，保证在所有操作之后，与 observer1 对应。
	- beforeWaiting: 先销毁 autoreleasePool(poolPop)，释放 pool 中之前延迟释放的对象；然后再创建 autoreleasePool(poolPush)，为 afterWaiting 的操作做准备。
	- exit: 销毁 autoreleasePool(poolPop)，释放 pool 中之前延迟释放的对象。
- AutoReleasePool 是一个隐式的双向链表，每一个节点都是一个 autoreleasePoolPage(可见数据结构)，也就是一个内存页，大小为4k。
- 每一个线程都一个 autoreleasePool 栈，来管理与其关联的 autoreleasePool，当线程销毁时，所有相关联的 pool 都被销毁，并释放其中的对象。

<figure>
    <img src="http://p8pq9azjn.bkt.clouddn.com/image/autoreleasePool/AutoreleasePool.png">
</figure>


## 代码分析
- 自动释放池是用双向链表来实现的，充分利用内存页，能快速查询且 release 延时释放的对象。
- 以哨兵对象作为一次 push 进来的标志，便于 pop 的时候查找，哨兵对象实际就是一个 nil。
- 调用 autorelease 的时候，会将对象压入 autoreleasePoolPage 中。
- pop 的时候会找到就近的哨兵对象，然后依次 release 对象，直到这个哨兵对象。
- 嵌套 autoreleasePool 的 pop，也是一层一层的找哨兵对象，然后释放对象。
- iOS 的入口函数中表明整个 APP 的运行行为都是放在一个 autoreleasePool 的。
- 每个线程都有 autoreleasePool，且1对1。
    - 主线程的 autoreleasePool 是默认开启的。因为 RunLoop 是默认运行的，APP 在观察到 KCFRunLoopEntry 的时候就创建了 pool。
    - 子线程的 runLoop 默认是不开启的。它的 pool 是在第一次调用 autorelease 的时候创建的。
- @autoreleasePool 语法糖的背后实现其实是一个局部 autoreleasePool 变量的生命周期。
    ```   
    - (void)func {
        @autoreleasepool {
            // do something.
        }
    }
    ==>
    - (void)func {
        __AtAutoreleasePool __autoreleasepool; // 当 __autoreleasepool 析构的时候，自动释放池 pop
        // do something.
    }
    因为：
    struct __AtAutoreleasePool {
        __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
        ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
        void * atautoreleasepoolobj;
    };
    
    ```

<figure>
    <img src="http://p8pq9azjn.bkt.clouddn.com/image/autoreleasePool/MemoryStructure.png">
</figure>
<figure>
    <img src="http://p8pq9azjn.bkt.clouddn.com/image/autoreleasePool/CoreProcess.png">
</figure>

### 核心代码
#### Push 操作
`NSObject.mm line:1881`  
```    
// push 函数
void *objc_autoreleasePoolPush(void)
{
    // 调用 AutoreleasePoolPage 的 push
    return AutoreleasePoolPage::push(); 
}

#  define POOL_BOUNDARY nil // 每次 push 的起始标记，也叫哨兵

// AutoreleasePoolPage 的 push 函数，以简化掉 debug 的兼容代码
static inline void *push()
{
    // 将 POOL_BOUNDARY 压入 autoreleasePoolPage 中，作为这次 autoreleasePool 的一个标记
    id *dest = autoreleaseFast(POOL_BOUNDARY);
    assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}
// 快速将对象压入 page 的函数 
static inline id *autoreleaseFast(id obj)
{
    AutoreleasePoolPage *page = hotPage(); // 获取当前可用的 page
    if (page && !page->full()) { // page 未满的时候，压入对象
        return page->add(obj); // 压入对象
    } else if (page) { // page 已满，需要创建新的 page，并压入对象
        return autoreleaseFullPage(obj, page);
    } else { // 当前没有 page，是一个空链表
        return autoreleaseNoPage(obj);
    }
}
// page 要溢出时的处理函数
static __attribute__((noinline)) id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
    assert(page == hotPage()); // 断言一定是当前可用的 page
    assert(page->full()); // 断言 page 一定满了
    
    do {
        if (page->child) page = page->child; // 使用下一个 page
        else page = new AutoreleasePoolPage(page); // 创建新 page，并加入到链表中
    } while (page->full()); // 只有 page 满的情况才执行
    
    setHotPage(page); // 重新设定当前可用 page，并存储在 TLS 中
    return page->add(obj); // 把目标对象压入 page
}
// 空链表的时候处理的函数
static __attribute__((noinline)) id *autoreleaseNoPage(id obj)
{
    assert(!hotPage()); // 断言当前没有可用的 page
    
    bool pushExtraBoundary = false; // 是否添加哨兵的标志位
    if (haveEmptyPoolPlaceholder()) { // 有占位的 page
        pushExtraBoundary = true; // 需要添加哨兵
    }
    else if (obj != POOL_BOUNDARY) { // 目标对象不是哨兵
        // We are pushing an object with no pool in place,
        // and no-pool debugging was requested by environment.
        _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
                     "autoreleased with no pool in place - "
                     "just leaking - break on "
                     "objc_autoreleaseNoPool() to debug",
                     pthread_self(), (void*)obj, object_getClassName(obj));
        objc_autoreleaseNoPool(obj);
        return nil;
    }
    else if (obj == POOL_BOUNDARY) { // 目标对象是哨兵
        return setEmptyPoolPlaceholder(); // 添加一个占位的 page
    }
    
    创建链表中的第一个节点 page
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page); // 设置当前可用的 page
    
    if (pushExtraBoundary) {
        page->add(POOL_BOUNDARY); // 添加哨兵
    }
    
    // Push the requested object or pool.
    return page->add(obj); // page 压入对象
}
```

#### Pop 操作
`NSObject.mm line:1886`
```
// pop 函数
void objc_autoreleasePoolPop(void *ctxt)
{
    // 调用 AutoreleasePoolPage 的 pop
    AutoreleasePoolPage::pop(ctxt);
}
// AutoreleasePoolPage 的 pop 函数，token 为 push 函数的返回值
static inline void pop(void *token)
{
    AutoreleasePoolPage *page;
    id *stop;
    
    // token 是占位 page
    if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
        // Popping the top-level placeholder pool.
        if (hotPage()) { // 当前有可用的 page
            pop(coldPage()->begin()); // 从第一个不可用的 page 的第一个压入的对象开始递归
        } else { // 当前 page 为空，说明 pool 没有被使用
            setHotPage(nil); // 设置当前 page 为空
        }
        return;
    }
    
    page = pageForPointer(token); // 查找 token 所在的 page
    stop = (id *)token;
    if (*stop != POOL_BOUNDARY) { // token 不是哨兵
        // 一些错误的处理
        if (stop == page->begin()  &&  !page->parent) {
            // Start of coldest page may correctly not be POOL_BOUNDARY:
            // 1. top-level pool is popped, leaving the cold page in place
            // 2. an object is autoreleased with no pool
        } else {
            // Error. For bincompat purposes this is not
            // fatal in executables built with old SDKs.
            return badPop(token); 
        }
    }
    
    if (PrintPoolHiwat) printHiwat(); // 重新计算最高水位
    
    page->releaseUntil(stop); // 释放对象知道遇到 token，也就是哨兵
    
    if (page->child) {
        // hysteresis: keep one empty child if page is more than half full
        if (page->lessThanHalfFull()) {
            page->child->kill();
        }
        else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```

#### Objc autorelease 操作
`NSObject.mm line:2331`
```
// autorelease 外部可用 API
- (id)autorelease {
    return ((id)self)->rootAutorelease(); // 调用 objc_object 的 autorelease 方法
}
// objc_object 的 autorelease 函数
inline id objc_object::rootAutorelease()
{
    // 被 TaggedPointer 标记，优化内存占用，指针值不是地址，而是值
    if (isTaggedPointer()) return (id)this; 
    // TLS + __builtin_return_address 内存优化
    if (prepareOptimizedReturn(ReturnAtPlus1)) return (id)this;

    return rootAutorelease2(); // 继续调用另一个 autorelease 的函数
}
// autorelease2 函数
__attribute__((noinline,used)) id objc_object::rootAutorelease2()
{
    assert(!isTaggedPointer());
    return AutoreleasePoolPage::autorelease((id)this); // 开始调用 autoreleasePoolPage 的 autorelease 函数，此处开始进入到 autoreleasePool 了。
}
// autoreleasePoolPage 的 autorelease 函数
static inline id autorelease(id obj)
{
    assert(obj);
    assert(!obj->isTaggedPointer());
    id *dest __unused = autoreleaseFast(obj); // 插入到 autoreleasePoolPage
    assert(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
    return obj;
}
```

#### 辅助函数
```
#  define EMPTY_POOL_PLACEHOLDER ((id*)1) // 只是一个占位的 page

// 获取当前可用的 page
static inline AutoreleasePoolPage *hotPage()
{
    AutoreleasePoolPage *result = (AutoreleasePoolPage *)
    tls_get_direct(key); // TLS 中查找保存的 page
    // 遇到占位 page，则无可用 page
    if ((id *)result == EMPTY_POOL_PLACEHOLDER) return nil;
    if (result) result->fastcheck(); // 对 page 结构的完整性做校验
    return result;
}
// 设置当前可用的 page
static inline void setHotPage(AutoreleasePoolPage *page)
{
    if (page) page->fastcheck(); // page 完整性校验
    tls_set_direct(key, (void *)page); // TLS 存储
}
// 构造函数
AutoreleasePoolPage(AutoreleasePoolPage *newParent)
: magic(), next(begin()), thread(pthread_self()),
parent(newParent), child(nil),
depth(parent ? 1+parent->depth : 0),
hiwat(parent ? parent->hiwat : 0)
{
    if (parent) {
        parent->check();
        assert(!parent->child);
        parent->unprotect();
        parent->child = this; // 将新的节点加入到链表中
        parent->protect();
    }
    protect();
}
// 判断是否有空的占位的 page
static inline bool haveEmptyPoolPlaceholder()
{
    id *tls = (id *)tls_get_direct(key);
    return (tls == EMPTY_POOL_PLACEHOLDER);
}
// 添加占位的 page
static inline id* setEmptyPoolPlaceholder()
{
    assert(tls_get_direct(key) == nil);
    tls_set_direct(key, (void *)EMPTY_POOL_PLACEHOLDER);
    return EMPTY_POOL_PLACEHOLDER;
}
// 链表中第一个不可用的 page
static inline AutoreleasePoolPage *coldPage()
{
    AutoreleasePoolPage *result = hotPage(); // 获取当前 page
    if (result) {
        while (result->parent) { // 一直反向遍历
            result = result->parent;
            result->fastcheck();
        }
    }
    return result;
}
// 释放函数
void releaseUntil(id *stop)
{
    // Not recursive: we don't want to blow out the stack
    // if a thread accumulates a stupendous amount of garbage
    
    // 清理到哨兵
    while (this->next != stop) {
        AutoreleasePoolPage *page = hotPage(); // 当前 page
        while (page->empty()) {
            page = page->parent;
            setHotPage(page); // 安全策略，重置当前 page
        }
        
        page->unprotect(); // 允许 write 操作
        id obj = *--page->next; // 找到 next 之前的对象，next 指向的是下一个可用的空地址
        memset((void*)page->next, SCRIBBLE, sizeof(*page->next)); // 内存清理
        page->protect(); // 关闭 write 操作
        
        if (obj != POOL_BOUNDARY) { // 非哨兵
            objc_release(obj); // 想对象发送 release 消息
        }
    }
    
    setHotPage(this); // 重置当前 page
}    
```

## 关联知识
- Tagged Pointer
    - Tagged Pointer 是用来存储一些轻量级的对象，来优化内存管理。比如 NSNumber 等。
    - Tagged Pointer 的指针不再是地址了，而是真正的值。不存储在堆上。
    - 内存读取有3倍效率，创建是以前的106倍。

## 引用
- [源码](https://opensource.apple.com/tarballs/objc4/)
- [AutoreleasePool的原理和实现](https://www.jianshu.com/p/1b66c4d47cd7)
- [黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)
- [Objective-C Autorelease Pool 的实现原理](http://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/)
- [iOS开发笔记（七）：深入理解 Autorelease](https://juejin.im/post/5a66e28c6fb9a01cbf387da1)
- [Objective-C Target Point](https://blog.csdn.net/xiejunyi12/article/details/61195716)


