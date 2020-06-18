---
title: OpengGL 中的同步及资源共享
categories: 
  - opengl
tags:
  - opengl
---

## 为什么需要同步

由于 OpenGL API的执行是异步的，所以需要同步，如果是这些API是同步的就没有这个话题了。异步API可以进行缓存，从而可以在合适的时机批量的将这些API调用（称为API命令）发送给GPU执行，避免应用过于频繁的在内核态和用户态切换。

这里的异步指的是一个GL API调用结束并不表示它已经被GPU执行了。GL命令会先被GPU驱动程序缓存在内存中，然后在某一个时机驱动程序再把GL命令发送到GPU硬件中，GPU硬件中有个命令队列，GPU会从这个队列中取出命令进行执行。所以一个GL命令会经过2次缓存，一次在GPU驱动程序中，一次在GPU硬件中。

> 有2个命令可以控制这两个缓存，`glFinish` 和 `glFlush`。
>
> - `glFinish` 命令可以保证所有发送到GPU的命令已经执行完毕（此时驱动和GPU硬件中的命令缓存都是空的），否则它会一直阻塞，所以应用应该谨慎使用，因为这可能会导致性能降低。
> - `glFlush` 命令可以保证GPU驱动中的所有命令都被放入GPU硬件（此时只有驱动缓存是空的）。

## OpenGL 提供的GL同步机制

正常情况下处于缓存中的命令什么时候被实际执行应用是不知道的，但是如果应用需要使用这些GL命令执行的结果，比如把渲染的结果作为位图读到内存，这个时候就必须要保证所有的GL绘制命令都被执行完成才可以，否则读到的位图就是不完整的，这里就需要同步机制，在单个GL Context中，OpenGL会保证这里需要的同步，这种同步属于`隐式同步(Implicit synchronization)`，也就是说这个同步不需要应用主动发起，而是OpenGL内部帮我们实现的。在单个GL Context中一般来讲应用都不需要主动使用同步机制，GL内部会在需要的时候进行隐式同步。

在多GL Context的时候情况就不一样了，比如进程中有两个GL Context，称为 ContextA 和 ContextB，ContextA 负责生成Texture，ContextB负责使用Texture，此时就需要保证在ContextB使用Texture之前Texture是完整的，如果此时用于生成Texture的GL命令还没有执行完毕，那么应用就需要主动调用同步机制来保证这些GL命令已经执行完毕。这种同步机制就是`显式同步(Explicit synchronization)`。

在OpenGL中显式同步使用`Sync Object`机制实现。涉及的API包括：

```c++
// 创建 sync object
GLsync glFenceSync(GLenum condition​, GLbitfield flags​);
// 等待sync object的信号触发，阻止**驱动**向GPU硬件发送新的GL命令直到sync信号触发（此时应用依然可以向驱动发送新的GL命令，所以称这种阻塞为server端阻塞）
void glWaitSync(GLsync sync, GLbitfield flags, GLuint64 timeout);
// 等待sync object的信号触发，阻止**应用**向GPU驱动发送新的GL命令直到sync信号触发（此时应用不能再向驱动发送新的GL命令，所以称这种阻塞为client端阻塞）
GLenum glClientWaitSync(GLsync sync​, GLbitfield flags​, GLuint64 timeout​);
```

这些API在 OpenGL3.2 或者 OpenGLES3.0 以上才支持，而chromium中要兼容 OpenGLES2.0，因此 chromium 使用类似的扩展，比如`GL_ARB_sync`，`GL_APPLE_fence`，`EGL_KHR_fence_sync`，`GL_NV_fence`等来解决这个问题。

> 更多信息参考官方文档： [Synchronization - OpenGL Wiki](https://www.khronos.org/opengl/wiki/Synchronization)

## OpenGL 资源共享

在一个应用中可以有多个线程，每个线程都可以创建自己的GL Context,所以可以实现让一些线程来专门生成资源，另一些线程来使用资源从而提高性能，这种资源的使用方式称为资源共享。

在 OpenGL 中可以使用 `share group` 来实现资源在不同Context之间的共享。创建share group的的API如下：

```c++
EGLContext eglCreateContext(EGLDisplay display,
  EGLConfig config,
  EGLContext share_context, // 这个参数用于指定需要共享资源的 Context
  EGLint const * attrib_list);
```

使用演示：

```c++
// 创建第一个 context，线程1
contextA = eglCreateContxt(display,config,NULL,attrib_list);
// 创建第二个 context,线程2
contextB = eglCreateContxt(display,config,contextA,attrib_list);
// 创建第三个 context,线程3
contextC = eglCreateContxt(display,config,contextA,attrib_list); // 这里使用 contextA 或者 contextB 效果一样
```

这里创建了三个 context,第一个context不需要指定`share_context`参数，后面两个context指定之前创建的context为share_context，这样这三个context之间就可以共享texture等资源了，我们称这三个 context 在同一个 `share group` 中。

此时资源可以共享了，但由于资源处于不同的线程（不同的Context）中，因此需要对资源的访问进行显式的同步控制。

> 注意(1)：
>
> 1. 严格来讲 OpenGL 并没有规定如何在Context之间共享资源，现有的资源共享方案都是由 EGL/CGL/WGL 等提供的。
> 2. 在不同平台上可能有不同的资源共享方法，比如 WGL 中的 `wglShareLists`，CGL中的 `EAGLSharegroup`,EGL中的 `share group`。
> 3. 资源共享一般都是在相同进程中的多个Context之间进行的，跨进程的资源共享一般需要将资源从GPU读到内存，然后在另一个进程中再次把资源上传到GPU，由于涉及到资源从GPU->内存->GPU这个过程，因此应该尽量避免使用这种方式；
> 4. 在一个线程中可以创建多个GL Context，但是这种Context之间是否是会自动共享资源，或者自动进行同步，这些行为是没有定义的，有些实现是每个线程有一个命令队列，而有些实现是每一个Context一个命令队列。
>
> 注意(2):
>
> 请记住`share group`这个概念，很多地方会使用到。

----------

参考文档：

- [Synchronization - OpenGL Wiki](https://www.khronos.org/opengl/wiki/Synchronization)
- [Sync Object - OpenGL Wiki](https://www.khronos.org/opengl/wiki/Sync_Object)
- [EAGLSharegroup - OpenGL ES - Apple Developer Documentation](https://developer.apple.com/documentation/opengles/eaglsharegroup)
- [wglShareLists function (wingdi.h) - Win32 apps - Microsoft Docs](https://docs.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-wglsharelists)
- <https://www.khronos.org/registry/EGL/extensions/KHR/EGL_KHR_fence_sync.txt>
- <https://www.khronos.org/registry/EGL/extensions/KHR/EGL_KHR_wait_sync.txt>