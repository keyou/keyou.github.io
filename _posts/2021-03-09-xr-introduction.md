---
title: Mixed Reality (MR/XR) SDK
tags:
  - xr
  - rendering
---

要开发XR应用，一般都要用到某种XR SDK，这些SDK一般会包含以下这些功能：

![xr-sdk](/data/xr-sdk.svg)

这些功能分为两类，一类是输入输出（Input），一类是渲染（Rendering）。

## 输入输出（Input）

输入输出主要从各种传感器获取原始数据数据，然后进行加工并转换为有意义的输出给到应用程序。

### 3DoF/6DoF

Dof 表示 **D**egrees **o**f **F**reedom，即自由度。3Dof 表示3个自由度，分别是 pitch，yall，row，也就是空间中的任意旋转。在XR中主要表示应用可以获取XR眼镜在空间中的旋转姿态。支持 6Dof 的眼镜，应用除了可以获取到旋转姿态之外还可以获取到位置信息。比如眼镜向前后，向左右，向上下移动的距离。示意图如下：

![](https://virtualspeech.com/img/blog/3dof-6dof-vr-headset.jpg)

所以，支持3DoF的眼镜，用户只能呆在原地通过转头的方式和应用交互，支持6DoF的眼镜，用户可以通过四处走动的方式和应用交互，体验显然要比3DoF好很多。

3Dof 数据一般通过获取加速度计，陀螺仪和磁力计来计算得出，算法相对简单。算法的难点主要包括静态检测，抗干扰，滤波算法，传感器融合等。6DoF 算法就相对要复杂很多，一般还要使用到摄像头，深度传感器，激光雷达等传感器。

### 传感器预测（STP）

由于传感器数据采集，传输，计算都需要消耗一定的时间，而且有些传感器的数据采集频率比较低，导致渲染画面的延迟变高。而一般现实世界的运动都是连续的，所以为了弥补这部分延迟，可以通过算法对传感器的数据进行预测，比如预测10ms，20ms等，以便降低最终的渲染延迟。

### 目标追踪（Object Tracking）

目标追踪指的是建立起虚拟空间中的物体和现实世界中的物体的关联关系，比如让虚拟空间中的物体始终放置于物理空间中的某一个位置，透过XR眼镜让用户感觉好像现实世界中也放置了一个物体在那里。

### 深度映射（Deep Mapping）

通过算法获取真实世界的深度信息的方法称为深度映射，其结果是生成一张真实世界的深度图，这张图用来指导后续的虚实结合渲染，比如将虚拟物体渲染到真实物体的“背后”。

## 渲染（Rendering）

渲染主要解决画面的显示问题，由于XR应用的显示和普通App的显示有很大区别，因此要解决一些更复杂的问题，以下这些技术手段本质上都是为了降低渲染的延迟。

在了解如何降低延迟之前，必须先指导延迟的产生原因。在 Android 系统中，正常情况下渲染的流程如下：

![xr-vsync-delay-two-frame](/data/xr-vsync-delay-two-frame.svg)

当 App 收到一个 VSYNC 信号的时候，它就开始第 N 帧的绘制，由于 App 默认使用双缓冲策略，因此绘制结束之后需要通过 `SwapBuffer` 提交绘制的结果到 `SurfaceFlinger`,当 SurfaceFlinger 收到 VSYNC 信号的时候，取出 App 绘制的第 N 帧进行合成，合成结束之后，通过 hwcomposer 进程讲结果 Commit 给内核，也就是CRTC，当 CRTC 收到 VSYNC 信号的时候开始显示第 N 帧画面到屏幕上。所以从第 N 帧的产生到显示至少会有 2 帧的延迟。

### Front Buffer Rendering

前缓冲渲染，通过设置应用使用单缓冲，可以避免SurfaceFlinger带来的延迟。该功能在 EGL1.2 以上可用，并且需要硬件设备支持。设置方法如下：

```c++
const EGLint surface_attribs[] = {
  // use single buffer
  EGL_RENDER_BUFFER, EGL_SINGLE_BUFFER, 
  EGL_NONE};
surface = eglCreateWindowSurface(display_, config_, window_, surface_attribs);
// or
bool result = eglSurfaceAttrib(display, surface_, EGL_RENDER_BUFFER, EGL_SINGLE_BUFFER);
if(!result) {
    LOGW("Set EGL_SINGLE_BUFFER error.");
}

// egl ext: EGL_ANDROID_front_buffer_auto_refresh
result = eglSurfaceAttrib(display, surface_, EGL_FRONT_BUFFER_AUTO_REFRESH_ANDROID, 1);
if(!result) {
    LOGW("Set EGL_FRONT_BUFFER_AUTO_REFRESH_ANDROID error.");
}
```

### Async Time Wrap（ATW）

`异步时间扭曲（ATW）`,`动态时间帧补偿（DTR）`,`异步二次投影（Asynchronous Reproduction）`都是类似的技术，主要解决的问题是降低渲染的延迟。当生成一帧画面的时间比较长（大于8ms）的时候，渲染的画面会和眼睛的运动不匹配，比如人戴着眼睛扭头的时候画面来不及改变，这会让人感觉“画面粘滞，不跟随”。这些技术会缓解这些问题。采用的办法是对已经生成的画面根据人体的运动进行二次处理（平移/扭曲变形），使画面看起来更“跟随”。示意图如下：

![xr-vsync-atw](/data/xr-vsync-atw.svg)

这种方式先将App画面渲染到缓存中，然后ATW线程获取当前最新的一帧画面以及当前最新的DoF数据，对画面进行Wrap操作，让画面看起来就像是使用了最新的DoF数据后的画面。比如，第 N+1 帧 App Rendering 画面还没有完成渲染，此时可以使用 App Rendering 的第 N 帧进行 Wrap，生成 Time Wrap N'，然后进行显示，这样就可以在一定程度上缓解由于耗时帧产生的丢帧问题。

> 注意：由于对 Front Buffer Rendering 以及 CRTC 的实现逻辑没有深究，所以这里的解释并不严谨，留待以后研究之后才能给出更严谨的解释。

如果把 Time Wrap 操作放在 RenderThread 线程中，这种方式称为 Time Wrap，如果 Time Wrap 放在独立的线程中就叫做 Async Time Wrap。

### Scanline Racing

扫描线追踪指的是通过渲染调度保证渲染的时机和屏幕的刷新（扫描线）同步。这种方案利用了屏幕的刷新特性，要求屏幕必须要按照从左眼到右眼（或相反）的方向按“行”进行刷新，并且应用对渲染Buffer的修改可以“实时”影响屏幕刷新所使用的Buffer才行。示意图如下：  
![xr-scanline-racing](/data/xr-scanline-racing.svg)  
上图中，屏从左往右刷新，当ScanLine刷新到左半屏的时候，应用降渲染的第N+1帧画面写入到右半缓冲区，当ScanLine刷新到右半屏的时候就可以直接显示N+1帧的画面，并且此时应用开始将N+2帧的画面写入到左半缓冲区（此时由于刷新线已经过了这个区域，所以画面还不能显示到屏幕上）。这样可以降低一半的延迟。如果渲染可以被拆分为更多的阶段，比如三段，四段，则渲染延迟理论上可以进一步降低。

既然左半屏和右半屏显示的内容不是同一帧，那这样做会产生撕裂么？理论上来讲此时是存在撕裂的，不过由于在XR中，左半屏和右半屏本来就有分割，所以用户并不会感受到明显的异常。

另外，屏幕显示一帧画面是需要时间的，比如60Hz的屏完整显示一帧画面可能需要16.6ms的时间。正因为此，所以才有可能使用扫描线追踪技术。这种技术的复杂度很高，对时间非常敏感，系统的抖动或者硬件的不稳定都可能导致这种技术变成负优化技术。

### MultiView Rendering

`MultiView Rendering` 在有些时候也称为`单步渲染`。在 XR 中，为了产生立体效果，一般左眼和右眼会看到略微不同的图像，一般情况下程序需要单独针对这两幅画面进行生成，这导致在一帧内 XR 应用消耗了比普通应用多一倍计算，而左右眼只是略微的不一样，那有没有办法只渲染一次就生成左右眼两幅不同的画面呢？答案就是 MultiView Rendering。

OpenGL 的 `GL_OVR_multiview` 和 `GL_OVR_multiview2` 扩展提供了对 MultiView Rendering 的支持。它们扩展了 GLSL，允许 Shader 在执行的时候使用 `gl_ViewID_OVR` 来为不同的画面选择不同的参数，比如不同的 MVP 矩阵，从而用户只需要调用一次 `glDraw*` 就能生成两幅不同的画面。不过这MultiView带来的性能提升可能并不是理论上的2倍，这个底层实现有关，在最差的情况下，它们可能并不能带来任何的性能提升，甚至是性能的下降。因为在一些设备上虽然用户层只调用了一次绘制操作，但是底层依然是通过隐式的调用2次绘制来实现的。可能导致性能下降是因为这两个扩展并不支持直接上屏，应用必须为它们创建专门的 FrameBuffer Object，这会带来额外的性能开销。

### Deep Mixing

通过DeepMapping可以获取一张真实世界的深度图，通过该深度图和虚拟世界的深度图进行叠加计算，根据深度的大小隐藏虚拟世界的一些物体，就好像虚拟世界的物体被真实世界中的物体遮挡了一样。通过这种方法可以把虚拟物体“放置”于真实世界某个物体的后面。从而给用户更真实的体验。

### MultiResolution Rendering

用户眼镜在同一时刻能注意到的范围是有限的，因此，为了降低渲染的资源消耗，可以对画面的不同位置使用不同的渲染精度。比如背景画面，天空，环境等画面使用较低的分辨率进行渲染，而主要的虚拟对象使用较高的精度，这种技术即为多分辨率渲染。

### Distortion Correction

在 XR 中由于设备的公差，透镜等因素，可能无法很好的实现虚拟世界和真实世界的结合，为了解决这些问题，一般都需要在显示最终画面之前对画面进行“畸变矫正”。一般的畸变矫正都不会针对每一个像素进行，而是采用网格的方式对网格进行畸变矫正，这样可以减少计算量。

## 总结

目前市面上各个厂家的SDK质量参差不齐，在延迟方面做的最好的是 HelloLens，Oculus 做的也非常好，毕竟他们积累的时间比较长。相比这些产品，我们目前的 SDK 还有很长的路要走。

-----
参考文档：

* [eglQueryContext - EGL Reference Pages](https://www.khronos.org/registry/EGL/sdk/docs/man/html/eglQueryContext.xhtml)
* [VSYNC - Android Open Source Project](https://source.android.com/devices/graphics/implement-vsync)
* [Asynchronous TimeWarp (ATW) - Oculus Developers](https://developer.oculus.com/documentation/native/android/mobile-timewarp-overview/?locale=en_US)
* [OpenGL ES SDK for Android: Using multiview rendering](https://arm-software.github.io/opengl-es-sdk-for-android/multiview.html#multiviewIntroduction)