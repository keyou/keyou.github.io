---
title: Chromium GPU 资源共享及同步
published: false
tags:
  - chromium
  - graphics
  - opengl
---

## Chromium 中的资源共享

### Mailbox 机制

Mailbox 机制由 [`CHROMIUM_texture_mailbox`](https://source.chromium.org/chromium/chromium/src/+/master:gpu/GLES2/extensions/CHROMIUM/CHROMIUM_texture_mailbox.txt) 扩展提供，它定义了一种在不同Context之间共享 Texture 对象中的图片数据的方式，不管这些Context是否处于相同的`share group`。

> 注意： 共享的是 Texture 对象中的图片数据，而不是 Texture 对象本身。

它定义了2个方法：

```c++
void glProduceTextureDirectCHROMIUM (GLuint texture, GLbyte *mailbox);
GLuint glCreateAndConsumeTextureCHROMIUM (const GLbyte *mailbox);
```

`glProduceTextureDirectCHROMIUM` 方法传入一个当前Context中已经存在的 `texture` 对象，然后返回一个指向该texture的 `mailbox`。后续可以在其他的Context中使用 `glCreateAndConsumeTextureCHROMIUM` 方法通过这个 `mailbox` 创建一个新的 `texture` 对象，新对象存在于新的 Context 中。结合 Command Buffer，可以实现跨进程共享Texture的效果。

`Mailbox` 类本身是一个定长的字节数组，作为资源的唯一标识符，默认是16字节，系统全局唯一，可以跨进程。

Mailbox 机制使用起来非常方便，但是它对 service 端的运行环境依赖非常严重，导致service端在某些平台上需要使用Virtual Context或者很多的同步机制才能实现，而这些会导致性能损失。再加上这种机制基于GL，无法很好的支持Vulkan，因此mailbox机制已经被标记为 `deprecated`，在当前的 Chromium 中只有 media 模块还在使用。新代码应该使用 `SharedImage` 机制。

### SharedImage 机制

ShareImage 机制从2018年开始引入，设计用来取代 mailbox 并且支持 Vulkan。它引入了一套SharedImage接口以及一个新的GL扩展 `CHROMIUM_shared_image`。

[`CHROMIUM_shared_image`](https://source.chromium.org/chromium/chromium/src/+/master:gpu/GLES2/extensions/CHROMIUM/CHROMIUM_shared_image.txt) 扩展定义了一种在不同进程间共享图片/Texture资源的方式，称为 SharedImage 机制。

```c++
GLuint glCreateAndTexStorage2DSharedImageCHROMIUM (const GLbyte *mailbox);
GLuint glCreateAndTexStorage2DSharedImageWithInternalFormatCHROMIUM (
                                            const GLbyte *mailbox,
                                            GLenum internal_format);
void glBeginSharedImageAccessDirectCHROMIUM (GLuint texture,
                                                   GLenum mode);
void glEndSharedImageAccessDirectCHROMIUM (GLuint texture);
```

`glCreateAndTexStorage2DSharedImageCHROMIUM*` 方法创建一个新的 Texture 对象和一个新的 `mailbox` 并将它们关联到起来。之后应用使用 `glBeginSharedImageAccessDirectCHROMIUM` 方法表明要开始操作（读/写） `texture` 对象了，然后应用就可以像普通的 Texture 对象一样使用 texture 了，比如向它里面写数据，或者读取已经写入的数据等。操作结束之后，调用 `glEndSharedImageAccessDirectCHROMIUM` 方法表明操作结束。



`GpuChannelMsg_CreateSharedImageWithData` 消息用来创建 SharedImage。

## 关于 native GL Context 和 virtual GL Context

native GL Context 直接和驱动打交道，而 virtual GL Context 是由应用自己实现的，它不会直接和 GPU 打交道，一般都要转发到 native GL Context 才能起作用。

使用 virtual GL Context 的好处有：

- 可以实现多个 virtual GL Context 对应一个 native GL Context，从而保证在应用中实际只有一个 GL Context；
- 可以将GL命令序列化后发送到独立的进程执行，实现虚拟的多进程资源共享；
- 可以提供自己的同步机制来进行多个virtual GL Context之间的同步，从而不依赖OpenGL提供的同步机制；
- 可以在应用层调度GL命令的执行；
- 可以做中间层屏蔽底层GL的差异，方便跨平台；

使用 virtual GL Context 的不足：

- 提高了复杂度，中间层可能导致性能降低；

## Chromium 中的GL同步机制

Chromium 中实现了多种机制来保证 GL 命令的时序。这些机制应用的主要目的是同步对共享资源的访问时序，比如要保证先完成向texture的绘制操作，然后才能取出texture进行合成。

> 完整的信息请参考官方文档 [GPU Synchronization in Chrome](https://chromium.googlesource.com/chromium/src.git/+/master/docs/design/gpu_synchronization.md)，这里主要讲自己的理解。

### 单 GL Context 不需要进行显式同步

如果你的代码只涉及一个 GL Context，不需要进行显式同步，不管是直接使用 native GL Context 还是使用基于 command buffer 的 GL Context。

### 单进程中相同 share group 的 native GL Context 之间使用 gl::GLFence 进行同步

`gl::GLFence` 类通过包装 OpenGL 的 `Sync Object` 机制来实现同步。但是由于 `Sync Object` 在OpenGLES2.0上没有提供，因此 Chromium 在不同平台依赖不同的GL扩展。对扩展的依赖情况可以从以下代码看出：

```c++
// ui/gl/gl_fence.cc

std::unique_ptr<GLFence> GLFence::Create() {
  std::unique_ptr<GLFence> fence;
#if !defined(OS_MACOSX)
  if (g_driver_egl.ext.b_EGL_KHR_fence_sync &&
      g_driver_egl.ext.b_EGL_KHR_wait_sync) {
    // Prefer GLFenceEGL which doesn't require GL context switching.
    fence = GLFenceEGL::Create();
    DCHECK(fence);
  } else
#endif
      if (g_current_gl_driver->ext.b_GL_ARB_sync ||
          g_current_gl_version->is_es3 ||
          g_current_gl_version->is_desktop_core_profile) {
    // Prefer ARB_sync which supports server-side wait.
    fence = std::make_unique<GLFenceARB>();
#if defined(OS_MACOSX)
  } else if (g_current_gl_driver->ext.b_GL_APPLE_fence) {
    fence = std::make_unique<GLFenceAPPLE>();
#else
  } else if (g_driver_egl.ext.b_EGL_KHR_fence_sync) {
    fence = GLFenceEGL::Create();
    DCHECK(fence);
#endif
  } else if (g_current_gl_driver->ext.b_GL_NV_fence) {
    fence = std::make_unique<GLFenceNV>();
  }

  DCHECK_EQ(!!fence.get(), GLFence::IsSupported());
  return fence;
}
```

更多详细信息参考：

- <https://www.khronos.org/opengl/wiki/Synchronization>
- <https://www.khronos.org/registry/EGL/extensions/KHR/EGL_KHR_fence_sync.txt>
- <https://www.khronos.org/registry/EGL/extensions/KHR/EGL_KHR_wait_sync.txt>

> 备注：
>
> - 这里以及后文所有涉及到多native GL Context的情况默认都在不同的线程；
> - 直接使用系统原生 GL 的方法称为 `native GL` 或者 `driver-level GL`；
> - TODO: 补充使用示例。

### 多 virtual GL Context 使用 CHROMIUM sync tokens

在 Chromium 中 Command buffer 的client端可以理解为 virtual GL Context。它提供了以上 virtual GL Context的优点。

Chromium 提供了 [`CHROMIUM_sync_point`](https://chromium.googlesource.com/chromium/src.git/+/master/gpu/GLES2/extensions/CHROMIUM/CHROMIUM_sync_point.txt) 扩展来支持 Command Buffer GL Context 中资源的同步。该扩展包括以下几个方法：

```c++
void GenSyncTokenCHROMIUM(GLbyte *sync_token);
void GenUnverifiedSyncTokenCHROMIUM(GLbyte *sync_token);
void VerifySyncTokensCHROMIUM(GLbyte **sync_tokens, GLsizei count);
void WaitSyncTokenCHROMIUM(const GLbyte *sync_token);
```


这种机制和 OpenGL 的 `Sync Object` 机制类似，但又有些不同，下面看一些例子。

```c++

```


------------------

参考文档：

- [Shared images and synchronization - Google Docs](https://docs.google.com/document/d/12qYPeN819JkdNGbPcKBA0rfPXSOIE3aIaQVrAZ4I1lM/edit#)
