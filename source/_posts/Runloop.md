---
title: 浅析Runloop
date: 2016-12-08 13:48:50
tags: [iOS,Runloop]
---

有一定iOS开发经验的人可能都听说过`RunLoop`。`RunLoop`，顾名思义，就是run loop ，跑圈的意思。

Apple对`Runloop`是这么解释的：

>  The NSRunLoop class declares the programmatic interface to objects that manage input sources. An NSRunLoop object processes input for sources such as mouse and keyboard events from the window system, NSPort objects, and NSConnection objects. An NSRunLoop object also processes NSTimer events.

​	 
简单的，`Runloop`可以理解为一个事件循环，循环中执行不同的代码，直到进入下一次循环的条件不足为止！

`Runloop`不是线程，不是GCD，而是一个对象，在一个APP里面不是唯一的。<!--more -->

下面的这个图介绍了代码执行最常见的两种方式，命令式和event驱动的。其中一个是一次执行到底，而另外一个是反复不停地进行某一个行为，也就是跑圈。
![runloop_1](http://7xt8tf.com1.z0.glb.clouddn.com/runloop_1.png/blog)


`RunLoop`就是事件驱动模型的代表。这种模型被称作 Event Loop。 Event Loop 在很多系统和框架里都有实现，比如 Node.js 的事件处理，比如 Windows 程序的消息循环，再比如 OSX/iOS 里的 `RunLoop`。实现这种模型的关键点在于：如何管理事件/消息，如何让线程在没有处理消息时休眠以避免资源占用、在有消息到来时立刻被唤醒。

所以，RunLoop 实际上就是一个对象，这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行上面 Event Loop 的逻辑。线程执行了这个函数后，就会一直处于这个函数内部 "接受消息-&gt;等待-&gt;处理" 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。

OSX/iOS 系统中，提供了两个这样的对象：`NSRunLoop` 和 `CFRunLoopRef`。
`CFRunLoopRef` 是在 `CoreFoundation` 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。而`NSRunLoop` 是基于` CFRunLoopRef`的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。`CoreFoundation`和`Foundation`对象在ARC中处理也是不一样的。所以使用`RunLoop`的时候一定要小心。

Apple 在`RunLoop`的介绍里还特别强调了下：

> The NSRunLoop class is generally not considered to be thread-safe and its methods should only be called within the context of the current thread. You should never try to call the methods of an NSRunLoop object running in a different thread, as doing so might cause unexpected results.

虽然我们无法创建RunLoop，但是Apple给我们提供了两个自动获取的函数：`CFRunLoopGetMain()` 和 `CFRunLoopGetCurrent()`

线程和 `RunLoop` 之间是一一对应的，一个线程只能有唯一对应的`runloop`，但这个`runloop`里可以嵌套子`runloop`，然后把他们之间的关系保存在一个全局的 Dictionary 里。线程刚创建时并没有 `RunLoop`，如果你不主动获取，那它一直都不会有。`RunLoop` 的创建是发生在第一次获取时，`RunLoop` 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 `RunLoop`（主线程除外）

下面介绍一点稍深入点的知识
先上图
![runloop_2](http://7xt8tf.com1.z0.glb.clouddn.com/runloop_2.png/blog)



`RunnLoop`有几个运行状态下的Mode

1. `kCFRunLoopDefaultMode`: App的默认 Mode，通常主线程是在这个 Mode 下运行的。
2. `UITrackingRunLoopMode`: 界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响，提高用户体验。
3. `UIInitializationRunLoopMode`: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用。
4. `GSEventReceiveRunLoopMode`: 接受系统事件的内部 - 
5. `kCFRunLoopCommonModes`: 这是一个占位的 Mode，没有实际作用。



一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 `RunLoop` 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。
​		
Source/Timer/Observer 被统称为 mode item，一个 item 可以被同时加入多个 mode。但一个 item 被重复加入同一个 mode 时是不会有效果的。如果一个 mode 中一个 item 都没有，则 `RunLoop` 会直接退出，不进入循环。

这里还有个概念叫 CommonModes，一个 Mode 可以将自己标记为Common。每当 `RunLoop` 的内容发生变化时，RunLoop 都会自动将 CommonMode Items 里的 Source/Observer/Timer 同步到有 Common  标记的所有Mode里。

我们在开发中经常会用到定时器，如果细心点就会发现，timer在你滑动的时候就会被停止，当滑动结束的时候才会继续。这就是因为mode不同造成的。我们可以把timer也加到滑动专用的trackingMode中去，这样timer就可以在滑动的时候保持继续运行！

详解：
在主线程的 `RunLoop` 里有两个预置的 Mode：`kCFRunLoopDefaultMode` 和 `UITrackingRunLoopMode`。这两个 Mode 都已经被标记为 Common 属性。DefaultMode 是 App 默认状态下所处的状态，`TrackingRunLoopMode` 是追踪滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个滚动视图时，`RunLoop` 会将 mode 切换为 `TrackingRunLoopMode`，这时 Timer 就不会被回调，并且也不会影响到滑动操作。如果将这个 Timer 分别加入这两个 Mode，或者将 Timer 加入到顶层的 RunLoop 的 CommonMode Items 中。CommonModeItems 被 `RunLoop` 自动更新到所有具有 Common 属性的 Mode 里去。这样就解决了Timer的回调问题。

还有一个`RunLoop`对象类型，叫做`CFRunLoopObserverRef`。它就是RunLoop的观察者，每一个observer都需要指定一个回调函数的指针，在当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化。
`RunLoop`的状态有这么几个
```kCFRunLoopEntry---------------------------------------即将进入Loop
kCFRunLoopBeforeTimers--------------------------------即将处理Timer
kCFRunLoopBeforeSources-------------------------------即将处理Source
kCFRunLoopBeforeWaiting-------------------------------即将进入休眠
kCFRunLoopAfterWaiting--------------------------------刚从休眠中唤醒
kCFRunLoopExit----------------------------------------即将退出Loop
```

有个很出名的cell自动计算行高的高性能三方框架就是利用这个做的优化！ 利用`RunLoop`即将进入休眠的间隙去做一些耗时的运算，可以大幅减少数据刷新的整体耗时，提高用户体验！

小Tips：
初学iOS的时候，很多人会有疑问，被标记了`autorelease`的对象究竟在什么时候释放了？到了`RunLoop`这里就有了答案。`RunLoop`BeforeWaiting时，对`autorelease`的对象发送消息,将这次Loop中产生的autorelease对象释放！

最后提供一些资料：
1. [sunnyxx的RunLoop线下分享](http://v.youku.com/v_show/id_XODgxODkzODI0.html)
2. [深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/) 
3. [CFRunLoopRef源码](https://opensource.apple.com/tarballs/CF/)



