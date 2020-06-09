---
title: OpenGL 和 EGL 的背景知识
categories:
  - graphics
  - opengl
tags:
  - graphics
  - opengl
  - egl
---

> 现在网上/书上有非常丰富的 OpenGL 入门资料，但是这些资料很多开头就直接教你使用 `GLFW`，`GLUT`，`SDL` 来搭建开发环境（EGL相对少一些），然后就开始讲 OpenGL 的原理以及 API 的使用方法，这种开局方式很高效也很“职校”，这篇文档想要分享一些 OpenGL 及 EGL 更基础的知识，揣测一下 OpenGL 和 EGL 的设计意图，不会讲解具体怎么使用，具体用法已经是非常成熟的技术了。

`OpenGL` 是由 `Khronos` 组织定义的一个**协议**，一套API，定义了 API 的输入和输出，有**很多**不同的实现，理论上任何人都可以按照这些标准实现一套 （纯软件方式）OpenGL 库，就像标准 C 库有很多不同的实现一样，比如 `SwiftShader` 就是一个基于 CPU 的纯软件的 OpenGL 实现。如果要想使用 GPU 硬件，一般是 GPU 设备制造商来实现这些 API （也有一些开源组织通过逆向工程的方式提供开源版的支持 GPU 的 OpenGL 实现，比如 `Nouveau`），这些库也被称为 GPU 驱动。

`OpenGL` 只负责绘图，不负责创建程序窗口，不负责处理键盘/鼠标等交互事件，甚至**不负责应用应该如何初始化 OpenGL**（这导致了很多麻烦的事情，文章开头的那些 `GLFW`/`GLUT`/`SDL` 就是帮你解决这些麻烦的事情的）。

## 准备 OpenGL 的环境

1. 安装支持 OpenGL 的 GPU 驱动（很多设备都已经默认安装了）；
2. 配置开发环境，主要是配置头文件路径和要链接的库；  
    OpenGL 也有很多高级语言的绑定，比如你可以使用 C#，java，lua，python 等语言来调用OpenGL，这是因为已经有人帮你将原生 C 语言的 OpenGL API `Binding(绑定)`到了这些语言上，这样你才可以使用非 C 语言来调用OpenGL，如果使用 C 版本的裸 OpenGL API，你就需要自己做这一步。
  
3. 初始化 OpenGL；  
    使用所在平台提供的方式或第三方库创建 `OpenGL Context`，加载 OpenGL 库及 Functions。高级语言的Binding一般都已经帮你完成了这一步。

经过以上三步之后，就可以开始调用 OpenGL 的 API 了。

`Context` 是 OpenGL 的核心，大部分负责最终绘制的 API 都依赖于当前的 `Context` 它一般是线程依赖的（内部状态使用LTS保存），但是 OpenGL 没有规定应该如何初始化 Context，因此它在不同系统/UI框架下都是不一样的，推荐使用现有的 系统/UI框架 提供的方式来初始化，比如 Windows 下的 WGL，X11 下的 GLX 等，或者如果当前平台支持 EGL 可以使用 EGL。

最原始的 OpenGL 使用要求用户自己在运行时动态加载 OpenGL 的库并获取OpenGL Functions。有很多OpenGL Loading Libraries可以帮你做这些事，它们很多都是通过自动代码生成的方式来加载这些函数。比如跨平台的 `ANGLE` 库，它通过代码生成的方法来动态加载 OpenGL Functions。

OpenGL是一个Rendering库，它只能看到一系列的三角形以及这些三角形（用于绘制）的一系列状态。它只管向一个buffer绘制内容，然后你需要用平台相关的方法（WGL/GLX/CGL/EGL）来将buffer中的内容swap到界面上。如果需要修改显示的内容你需要重新进行所有的绘制，如果要对显示做动画你就需要循环绘制每一帧的内容，那应该以怎样的频率去绘制呢？ OpenGL 提供了 `VSYNC` 机制来控制绘制的频率，也就是常说的帧率，一般它会控制重绘的帧率和屏幕的刷新率尽量一致。

> OpenGL 分享课件： <https://r302.cc/ylDQ34>

## EGL

OpenGL 定义了如何绘制以及绘制之后的样子是什么。但是它没有定义在哪里绘制（绘制在内存？显存？）以及如何将绘制的结果进行呈现（怎么显示在窗口中？），也没有定义 OpenGL 应该如何初始化。

在 EGL 出来之前市面上已经有了很多的 OpenGL 实现，它们各自都提供了自己的方法来解决以上问题。比如 windows 上的 WGL，X11 上的 GLX，Mac 上的 CGL，以及各种高级语言对这些实现的包装，比如 javascript 的 WebGL（基于OpenGL ES）等，。

可以看到在不同平台下 OpenGL 的初始化都不一样，因此 `Khronos` 推出 `EGL` 来统一这些行为。

`EGL` 定义了一套API，能够统一不同平台 OpenGL 的初始化以及将 OpenGL 和所在平台的 UI 框架用统一的方式对接起来。这样你就可以在不同平台使用相同的 API 去初始化及使用 OpenGL 了。目前很多平台已经支持了 EGL，比如 Android 系统上原生提供 EGL API 的支持。

## 其他

* `OpenCL`：是用来在GPU上进行通用计算的API。
* `DL`: Dynamic Loading,指动态加载的库，在linux下由libdl.so库提供的dlopen,dlsym等方法实现。
* `soname`: 记录在ELF文件中的一个逻辑名，它唯一表示一个库的名字，最简单的情况下它和文件名一样，但是一般都不一样，常用它来标识同一个库的多个不同的版本。
* `Khronos`: 最初是由 Nvidia，Intel，ATI 等组织联合成立的非盈利组织。

-----------

参考链接：

* https://learnopengl.com/
* https://www.khronos.org/opengl/wiki/Getting_Started
* https://www.khronos.org/opengl/wiki/Related_toolkits_and_APIs#Context.2FWindow_Toolkits
* https://www.khronos.org/opengl/wiki/Creating_an_OpenGL_Context_(WGL)
* https://www.khronos.org/opengl/wiki/Load_OpenGL_Functions
* https://www.saschawillems.de/creations/opengl-hardware-capability-viewer/
* https://docs.unity3d.com/Manual/OpenGLCoreDetails.html
* http://opengl.gpuinfo.org/
* http://opengl.gpuinfo.org/compare.php?compare=compare&id%5B3526%5D=on&id%5B3525%5D=on