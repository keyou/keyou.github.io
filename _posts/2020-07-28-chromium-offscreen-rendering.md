---
title: Chromium Offscreen Rendering And Embedded
published: true
categories:
  - chromium
  - render
tags:
  - chromium
  - render
---

在某些场景下需要将 Chromium 嵌入到其他应用程序中，这一般是通过将浏览器窗口嵌入到应用中来实现的，但是某些自绘的GUI框架无法嵌入原生的窗口，或者嵌入之后无法很好的和这些GUI框架进行互操作，比如 Windows 下的 WPF 框架。这个时候就需要通过 Chromium 的离屏渲染方式来进行嵌入。

## Chromium Render Target

渲染目标指用来存放最终渲染结果的地方，比如一个窗口，一块内存，一个图片文件以及system framebuffer，gl texture等都可以作为渲染目标。对于上层应用开发者来说，渲染目标并不那么重要，比如在Android应用开发中，开发者使用大量系统提供的UI组件来构建UI，此时渲染目标是什么开发者并不关心，因为这部分工作系统已经封装了起来。但是在 Chromium 中并不是这样，因为 Chromium 要考虑跨平台，它要尽可能使用统一的方式来渲染网页，所以它要考虑渲染目标的问题。

在不同系统中，一般都会有窗口的概念，它是最常见的渲染目标。开发者在窗口上的绘制会被用户直接看到。Chromium 的常规渲染目标也是窗口，也就是说一般情况下 Chromium 都是把网页渲染到窗口中。所以可以想到，Chromium 在初始化时，必须存在一个窗口，否则 Chromium 不知道应该将网页渲染到什么地方。这个要求一般情况下都不会有什么问题，但是当用户想要把网页嵌入自己的应用中的时候，事情就复杂了，尤其是当用户使用**自绘**类型的UI框架的时候，如果这些框架不支持窗口的嵌套，那么就没办法把网页所在的窗口嵌入自己的程序。

所以，将窗口作为渲染目标显然有其局限性，如果能够先将网页渲染到一个中间介质中，然后开发者自行控制将该页面呈现给用户，那Chromium就可以被更加灵活的嵌入其它应用中，这种渲染方式称为**离屏渲染**。一般选用内存或者gl texture来作为离屏渲染目标。因为它们可以被方便的嵌入其他应用中。

## Chromium Offscreen Rendering

`viz` 中的 `viz::OutputSurface` 封装了各种渲染目标，包括离屏渲染。但可惜的是 Chromium 并没有在上层提供API允许应用选择和访问底层的渲染目标。

> 我觉得这并没有什么道理，有人说 Chromium 又不是为了能被嵌入其他应用而设计的，不需要提供这样的功能，但一个事实是 WebView 的出现就是为了能够将 Chromium 嵌入其他Android应用，为什么就不能让 WebView 也支持全平台呢？虽然有类似 [CEF](https://github.com/chromiumembedded/cef-project) 这样的项目专门提供将Chromium嵌入其他应用的功能，但由于它属于独立的项目，开发实力远远比不上 Chromium 项目。

因此，要想将 Chromium 以离屏渲染的方式嵌入自己的应用，需要将Chromium中的离屏渲染目标暴露出来，允许上层应用访问。如果选择内存作为离屏渲染目标，可以使用共享内存来存储渲染结果，CEF 的离屏渲染就是通过这种方式，但这种方式只支持软件渲染，性能一般。如果选择以 gl texture 作为离屏渲染目标，需要使用共享Context来实现资源的共享，代码演示见 [demo_viz_layer_offscreen](https://github.com/keyou/chromium_demo/tree/c/80.0.3987/demo_viz).这种方式所能实现的性能更高。

关于 Chromium 中各种渲染目标的介绍见 `viz` 的文档 [How viz works]({% post_url 2020-07-29-how-viz-works %})。
