---
title: CADisplayLink 回顾总结
date: 2018-05-18 18:47:33
tags: [iOS, RunLoop, CADisplayLink]
categories:
- iOS
- RunLoop
- 应用
toc: true

---

## 概念
CADisplayLink 是一个保持和屏幕刷新率同步的定时器。之所以说是同步的而不是一样的，是因为 CADisplayLink 可以设置每几帧回调一次。

## 原理
CADisplayLink 添加到 RunLoop 后，在每次 vsync 信号到来时，都会执行回调。vsync 信号是读取下一帧数据的时钟信号。由于没有源码，不知道确切的设计。但是从 CADisplayLink 的特性来猜想，CADisplayLink 可能是向 RunLoop 注册了某个 source1，在 source1 关联的 port 监听到屏幕刷新结束的时候，就回调，然后在 CADisplayLink 内部根据属性设定来决定是否需要对外回调。

## 与其他两种定时器的比较

### CADisplayLink
- 优点：
    - 与屏幕刷新率保持同步。
    - 触发回调的时机相比其他定时器来说，要准确的多。
- 缺点：
    - 不能被继承。
    - 回调处理时间如果太长，会造成掉帧现象。
    - 回调时间灵活度小，间隔时间只能是屏幕刷新间隔的整数倍。

### NSTimer
- 优点：
    - 回调时间灵活度大，可以随意设定时间。
- 缺点：
    - 子线程中使用，需要开启 RunLoop，否则，无效。
    - 一般需要注册到 RunLoop 的 commonMode 中使用，否则在视图滑动的时候，不会接受到定时器到时的回调。
    - 回调时机准确度低，有延时。

### dispatch_source_t
- 优点：
    - 不受 runLoop mode 的影响，在视图滑动的时候也可以接受到回调。
- 缺点：
    - 回调时机较准确，但是不是最准确。
    - 在 GCD 内部管理的线程占满时，回调会阻塞延迟。
    
## 引用
- [CADisplayLink(逐帧动画)](https://www.jianshu.com/p/656d8ec3e678)
- [老司机带你走进Core Animation 之CADisplayLink](https://www.jianshu.com/p/434ec6911148)
- [深入理解CADisplayLink、NSTimer和dispatch_source](http://v2it.win/ios/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3cadisplaylink%E5%92%8Cnstimer/)