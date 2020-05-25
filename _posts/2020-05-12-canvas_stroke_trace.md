---
published: false
tags:
  - blink
---

chrome://tracing 显示 VSync 的方法：
在Andoid中 Compositor 线程的 Scheduler::BeginFrame 中的 args.frame_time_us 记录了vsync的开始位置，单位是微秒。
详见：https://chromium.googlesource.com/external/github.com/catapult-project/catapult/+/HEAD/tracing/tracing/extras/vsync/vsync_auditor.html


cc的调度：
ProxyImpl 对接调度器，运行在Compositor线程中，ProxyMain 执行真正的 paint和draw等，它运行在主线程，commit运行在Compositor线程中，tile运行在独立的tile线程，
compositor线程中的 ProxyImpl::ScheduledActionSendBeginMainFrame 将 ProxyMain::BeginMainFrame 调度到Render的主线程；


https://www.w3.org/2018/12/games-workshop/slides/20-reducing-input-latency.pdf
![](/images/2020-05-12-16-59-16.png)

https://developer.mozilla.org/en-US/docs/Web/API/PointerEvent/getCoalescedEvents
The getCoalescedEvents() method of the PointerEvent interface returns a sequence of all PointerEvent instances that were coalesced into the dispatched pointermove event.

采用了 [offscreen-canvas](https://developers.google.com/web/updates/2018/08/offscreen-canvas) 机制后，requestAnimationFrame的执行频率可以达到50fps以上，有时可以达到60fps，但是在demo中发现实际的渲染帧率是很低的，可能是因为demo中没有显示的进行画面的提交。TODO: 调查原因。

CanvasRenderingContext对应h5中canvas的Context对象，在js中可以通过`canvas.getContext()`获取一个context对象，它对应C++中的`blink::CanvasRenderingContext`对象，它实现了`base::Thread::TaskObserver`接口，用来监听当前线程的消息循环，这会导致`DidProcessTask`方法被调用，在该方法中`CanvasRenderingContext::FinalizeFrame`方法会被执行，它为生成Frame做最后的准备工作，当他执行完毕后，HTMLCanvasElement会根据自己是否是desync的，使用不同的方式生成Frame，如果desync=true，则调用`CanvasResourceDispatcher::DispatchFrame`方法进行CompositorFrame的生成和提交。否则它会等待cc进行下一次的`ThreadProxy::BeginMainFrame`调度，然后使用cc来完成CF的生成和提交。

追踪Rendering的Tracing发现Render的主线程非常繁忙，根本就来不及以60fps的速度生成CF,因此这直接导致了Canvas渲染帧率的降低。另外，由于desync方式下CF的提交并不受cc的调度，而`window.requestAnimationFrame`是受到cc的调度的，因此在desync下不能通过`requestAnimationFrame`来统计帧率，这样统计出来的结果是无效的。

使用手在canvas上进行绘制，但不进行实际的绘制操作，只进行fps，pps，eps的统计，可以发现这些数据基本都可以达到或超过60fps，因此推测性能瓶颈可能不在input部分（虽然eps可以达到60fps，但是并没有验证这些数据的延迟有多大；desync模式可以极大提高eps，使它基本和pps相当；）。

因此要想优化笔记书写，首先要提高Canvas的渲染帧率，使它尽量能够接近60fps，否则即使input很及时，也会由于渲染帧率低而导致笔迹延迟比较高。

要提高fps有两个思路，一是找出当前渲染流水线的瓶颈，然后针对笔记书写场景进行优化，该方案的难度相对高一些，但是可以实现对用户透明，而且从长远来看可以让我们对渲染有更加深刻的理解，从而有利于团队的成长；另一钟思路是重新实现一个Canvas或者CanvasRenderingContext，这样可以让我们直接控制它的渲染从而绕过部分甚至全部的blink渲染流水线，缺点是对用户不透明。

经过思考，决定采用自定义CanvasRenderingContext的方式来绕过cc和viz,直接使用FrameBuffer来进行渲染。
