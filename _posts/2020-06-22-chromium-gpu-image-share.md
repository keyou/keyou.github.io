---
title: Chromium GPU Resource Share (Shared Image)
published: true
tags:
  - chromium
  - graphics
  - rendering
  - opengl
---

> Update:
>
> - 2020.8.5: 添加使用 GPU-R 和 OOP-R 接口的代码示例。
> - 2022.7.26: 修正关于 OOP-D 的错误描述。

资源共享指的是在一个 Context 中的创建的 Texture 资源可以被其他 Context 所使用。一般来讲只有相同 `share group` Context 创建的 Texture 才可以被共享，而 Chromium 设计了一套允许不同 `share group` 并且跨进程的 Texture 共享机制。

Chromium 中有新旧两套共享 Texture 的机制，一套是 Mailbox 机制，一套是 SharedImage 机制。

## Mailbox 机制

Mailbox 机制由 [`CHROMIUM_texture_mailbox`](https://source.chromium.org/chromium/chromium/src/+/master:gpu/GLES2/extensions/CHROMIUM/CHROMIUM_texture_mailbox.txt) 扩展提供，它定义了一种在不同Context之间共享 Texture 对象中的图片数据的方式，不管这些Context是否处于相同的`share group`。

> 注意： 共享的是 Texture 对象中的图片数据，而不是 Texture 对象本身。

它定义了2个方法：

```c++
void glProduceTextureDirectCHROMIUM(GLuint texture, GLbyte *mailbox);
GLuint glCreateAndConsumeTextureCHROMIUM(const GLbyte *mailbox);
```

`glProduceTextureDirectCHROMIUM` 方法传入一个当前Context中已经存在的 `texture` 对象，然后返回一个指向该texture的 `mailbox`。后续可以在其他的Context中使用 `glCreateAndConsumeTextureCHROMIUM` 方法通过这个 `mailbox` 创建一个新的 `texture` 对象，新对象存在于新的 Context 中。结合 Command Buffer，可以实现跨进程共享Texture的效果。

`Mailbox` 类本身是一个定长的字节数组，作为资源的唯一标识符，默认是16字节，系统全局唯一，可以跨进程。

Mailbox 机制使用起来非常方便，但是它对 service 端的运行环境依赖非常严重，比如要求 service 端的所有 Context 都必须属于相同 share group，这导致service端在某些平台上需要使用Virtual Context或者很多的同步机制才能实现，而这些会导致性能损失。再加上这种机制基于GL，无法很好的支持Vulkan，因此 Mailbox 机制已经被标记为 `deprecated`，在当前的 Chromium 中只有 media 模块还在使用。新代码应该使用 `SharedImage` 机制。

## SharedImage 机制

ShareImage 机制从2018年开始引入，设计用来取代 Mailbox 机制并且支持 Vulkan。它引入了一套 client 端的 SharedImage 接口以及一个新的GL扩展 `CHROMIUM_shared_image`。

### SharedImage 接口

主要接口为 `SharedImageInterface`，用来创建 shared images，定义如下：

```c++
class GPU_EXPORT SharedImageInterface {
 public:
  virtual Mailbox CreateSharedImage(...) = 0;
  virtual void UpdateSharedImage(...) = 0;
  virtual void DestroySharedImage(...) = 0;

  // SwapChain 只在 Windows 上使用，这里不做介绍
  // virtual SwapChainMailboxes CreateSwapChain(...) = 0;
  // virtual void PresentSwapChain(...) = 0;
  ...
};
```

这是一个 client 端的接口，可以通过 `gpu::GpuChannelHost` 来获取到。对它的调用会通过 IPC 接口发送到 service 端，service 端会使用合适的机制来存储 SharedImage 的数据，比如GL Texture，GMB(GpuMemoryBuffer)等。

`CreateSharedImage` 方法创建一个新的 SharedImage 并返回一个 mailbox 指向它。`UpdateSharedImage` 方法更新指定的 SharedImage 的属性。`DestroySharedImage` 销毁指定的 SharedImage，释放相关内存。

在client端，通过 `CHROMIUM_shared_image` 扩展提供的方法来读写 SharedImage 数据。`Mailbox` 机制中 `CHROMIUM_texture_mailbox` 扩展提供的方法也可以用来访问 `SharedImage`，因为 `SharedImage` 机制兼容了 `Mailbox` 机制。但应该尽量比避免这样使用，因为 Mailbox 机制已经过时了。在 service 端也可以用这种方法来访问 ShareImage。

在servcie端，`SharedImage` 实现了大概3类存储机制，分别为 GLTexture/EGLImage，GMB 和 VulkanImage，这三大类又被抽象为了很多种小类。下面这些都是 `SharedImage` 可能的存储后端：

```c++
gpu::SharedImageBackingGLTexture
gpu::SharedImageBackingGLImage
gpu::SharedImageBackingEglImage
gpu::SharedImageBackingAHB
gpu::SharedImageBackingD3D
gpu::SharedImageBackingOzone
gpu::ExternalVkImageBacking
gpu::SharedImageVideo
```

其中 `gpu::SharedImageBackingGLTexture` 使用 GL Texture 来存储 SharedImage 数据。`gpu::SharedImageBackingEglImage` 使用 EGLImage 来存储数据。`gpu::SharedImageBackingAHB` 仅用于 Android 平台，使用 Android 提供的 AHardwareBuffer 来存储数据。`gpu::ExternalVkImageBacking` 对接 Vulkan。`gpu::SharedImageBackingGLImage` 比较特殊，它表示使用 `gpu::GLImage` 类来进行存储，`gpu::GLImage` 又抽象了不同的存储后端，最终也可能使用 GL Texture。

SharedImage 机制本质上抽象了 GPU 的**数据存储**能力。即允许应用直接把数据存储到 GPU （GPU 能访问到的内存）中，以及直接从 GPU 中读取数据，并且允许跨过`shared group`边界。理解了这一点，应该比较容易想到哪些场景可以使用 SharedImage 机制，下面这些是 Chromium 中使用 SharedImage 机制的一些场景：

1. CC模块： 先将画面 Raster 到 SharedImage，然后再发送给 Viz 进行合成。
2. OffscreenCanvas： 先将 Canvas 的内容 Raster 到 SharedImage，然后再发送给 Viz 进行合成。
3. 图片处理/渲染： 一个线程将图片解码到 GPU 中，另一个线程使用 GPU 来修改或者渲染图片。
4. 视频播放： 一个线程将视频解码到 GPU 中，另一个线程来渲染。

下面介绍2个操作 ShareImage 的扩展。需要注意的是在使用这些扩展方法之前都先要有一个指向 SharedImage 的 mailbox，可以使用 `SharedImageInterface` 接口创建，也可以是从其他地方传过来。

### CHROMIUM_shared_image 扩展

[`CHROMIUM_shared_image`](https://source.chromium.org/chromium/chromium/src/+/master:gpu/GLES2/extensions/CHROMIUM/CHROMIUM_shared_image.txt) 扩展定义了以下4个方法:

```c++
// Extension CHROMIUM_shared_image
GLuint glCreateAndTexStorage2DSharedImageCHROMIUM(const GLbyte *mailbox);
GLuint glCreateAndTexStorage2DSharedImageWithInternalFormatCHROMIUM(
                                            const GLbyte *mailbox,
                                            GLenum internal_format);
void glBeginSharedImageAccessDirectCHROMIUM(GLuint texture,
                                            GLenum mode);
void glEndSharedImageAccessDirectCHROMIUM(GLuint texture);
```

`glCreateAndTexStorage2DSharedImageCHROMIUM*` 方法根据传入的 `mailbox` 创建一个新的 Texture 对象。然后应用可以使用 `glBeginSharedImageAccessDirectCHROMIUM` 方法获取读/写 `texture` 对象的权限，然后使用常规的读写texture的GL命令访问texture的内容，比如`glGetTexImage`，`glReadPixels`，`glTexImage2D`等或者使用skia来间接访问texture的内容。操作结束之后，调用 `glEndSharedImageAccessDirectCHROMIUM` 方法释放权限。

~~这些接口用于 `OOP-D(Out-Of-Process Display Compositor)` 机制下的 Raster。关于 OOP-D 见后续文档，下面的代码演示使用 OOP-D 接口创建 SharedImage：~~

这些接口用于 `GPU-R` 机制下的 Raster，这种 Raster 机制已经被 `OOP-R` Raster 机制替代，并且在2022年2月份被移除，这里只用于演示旧版本 `GPU-R` 方式的 Raster：

```c++
// cc/raster/gpu_raster_buffer_provider.cc

static void RasterizeSourceGPUR(const RasterSource* raster_source,...) {
  gpu::raster::RasterInterface* ri = context_provider->RasterInterface();
  auto* sii = context_provider->SharedImageInterface();

  // 创建空的 SharedImage
  auto mailbox = sii->CreateSharedImage(
      resource_format, resource_size, color_space, kTopLeft_GrSurfaceOrigin,
      kPremul_SkAlphaType, flags, gpu::kNullSurfaceHandle);
  ri->WaitSyncTokenCHROMIUM(sii->GenUnverifiedSyncToken().GetConstData());

  // 使用 GPU-R 接口给 SharedImage 填充内容，内容来自 RasterSource
  GLuint texture_id = ri->CreateAndConsumeForGpuRaster(mailbox);
  ri->BeginSharedImageAccessDirectCHROMIUM(
      texture_id, GL_SHARED_IMAGE_ACCESS_MODE_READWRITE_CHROMIUM);
  {
    // 使用 texture_id 作为渲染目标
    viz::ClientResourceProvider::ScopedSkSurface scoped_surface(texture_id,...);
    SkSurface* surface = scoped_surface->surface();
    SkCanvas* canvas = surface->getCanvas();
    // 将 RasterSource 中的 DisplayItem 应用到 Canvas 中
    raster_source->PlaybackToCanvas(canvas,...);
  }
  ri->EndSharedImageAccessDirectCHROMIUM(texture_id);
  ri->DeleteGpuRasterTexture(texture_id);
  
  // 后续就可以使用 SharedImage 创建 TransferableResource
  ...
}
```

### CHROMIUM_raster_transport

```c++
// Extension CHROMIUM_raster_transport
void glBeginRasterCHROMIUM(GLuint sk_color, GLuint msaa_sample_count, GLboolean can_use_lcd_text, const GLbyte* mailbox);
void glRasterCHROMIUM(GLuint raster_shm_id, GLuint raster_shm_offset, GLuint raster_shm_size, GLuint font_shm_id, GLuint font_shm_offset, GLuint font_shm_size);
void glEndRasterCHROMIUM(void);
```

`glBeginRasterCHROMIUM` 方法表示要开始执行 Raster 操作了，Raster的结果存放到传入的 `mailbox` 对应的 SharedImage 中。`glRasterCHROMIUM` 将 `cc::DisplayItemList` 序列化后发送到service端。参数`raster_shm_id`指向存储 `cc::DisplayItemList` 序列化数据的共享内存。`glEndRasterCHROMIUM` 结束 Raster 操作。

这些接口用于 `OOP-R(Out-Of-Process Raster)` 机制下的 Raster。关于 OOP-R 见后续文档，
下面的代码演示使用 OOP-R 接口创建 SharedImage：

```c++
// cc/raster/gpu_raster_buffer_provider.cc

// OOPR Raster
static void RasterizeSourceOOP(const cc::RasterSource* raster_source,...) {
  gpu::raster::RasterInterface* ri = context_provider_->RasterInterface();
  auto* sii = context_provider->SharedImageInterface();
  
  // 创建空的 SharedImage
  auto mailbox = sii->CreateSharedImage(
        resource_format, resource_size, color_space, kTopLeft_GrSurfaceOrigin,
        kPremul_SkAlphaType, flags, gpu::kNullSurfaceHandle);
  ri->WaitSyncTokenCHROMIUM(sii->GenUnverifiedSyncToken().GetConstData());

  // 使用 OOP-R 接口给 SharedImage 填充内容，内容来自 RasterSource
  ri->BeginRasterCHROMIUM(
      raster_source->background_color(), playback_settings.msaa_sample_count,
      playback_settings.use_lcd_text, color_space, mailbox.name);
  // 注意这里直接将 DisplayItemList 传到 Service 端进行 Raster，
  // 这是 OOP-R 和 OOP-D 的本质区别。
  ri->RasterCHROMIUM(raster_source->GetDisplayItemList().get(),...);
  ri->EndRasterCHROMIUM();
  
  // 后续就可以使用 SharedImage 创建 TransferableResource
  ...
}
```

### SharedImage 架构设计

![sharedimage](/data/gpu_share_image_arch.svg)

SharedImage 被设计用于多进程架构，Client 端可以有多个，比如 Browser/Render/Gpu 进程都可以作为 Client 端，Service 端只能有一个，它运行在 Gpu 进程中。Client 和 Servcie 通过 IPC 进行通信。Client 端的接口主要包括 SharedImageInterface 和 2 个扩展。Service 端将数据存储在基于不同技术实现的 SharedImageBacking 中。

### SharedImage 实现原理

SharedImage 部分的类图如下（单击查看大图）：

[![sharedimage](/data/gpu_share_image.svg)](/data/gpu_share_image.svg)

client 端：

应用调用 `gpu::SharedImageInterface::CreateSharedImage` 接口来创建一个新的 SharedImage，返回一个 mailbox 指向它。该调用通过向 service 端发送 `GpuChannelMsg_CreateSharedImage` 或者 `GpuChannelMsg_CreateGMBSharedImage` 消息来创建 ShareImage。

应用通过 `CHROMIUM_shared_image` 和 `CHROMIUM_raster_transport` 扩展来读/写 SharedImage 中的数据，这两个扩展的实现分别位于 `gpu::GLES2Interface` 和 `gpu::RasterInterface` 接口，这两个接口是 Command Buffer 机制的入口，因此，对这两个扩展的调用会通过 Command Buffer 机制发送到 servcie 端。

servcie 端：

client 端发送的 `GpuChannelMsg_CreateSharedImage` IPC 消息会被路由到 `gpu::SharedImageStub` 类，该类通过用 `gpu::SharedImageFactory` 来创建 SharedImage。注意没有一个名为 SharedImage 的类，在 SharedImage 机制下，代表数据的 SharedImage 类被分为了2个概念，一个是 `SharedImageRepresentation`（简称 Representation 类），一个是 `SharedImageBacking`（简称 Backing 类），Representation 类是 SharedImage 框架外可见的类，它包装了 Backing 类。Backing 类是维护最终存储空间的类。所以可以说一个 `SharedIamgeBacking` 对象就表示一个 SharedImage。
所有的 `SharedImageBacking` 对象在创建后都会通过 `gpu::SharedImageManager::Register` 注册到 SharedImageManager 中。

在 `CHROMIUM_shared_image` 和 `CHROMIUM_raster_transport` 扩展方法的实现中，会使用 `gpu::SharedImageManager` 类来操作 `SharedImageBacking`。

所以不管是 SharedImageInterface 还是这两个扩展最终都是在操作 `SharedImagebacking`。因为它是真正存储数据的地方。

另外，`gpu::GpuMemoryBufferFactory` 用来申请 Gpu 可以访问到的内存，在不同平台上有不同的机制，在 Android 上是 AHardwareBuffer，在 Linux 上是 DMA buffer,而 Windows 上使用 DXGI。

## 总结

在当前的 Chromium 中，Mailbox 机制是建立在 SharedImage 机制之上的。旧的 Mailbox 机制的接口正在被废弃（Mailbox 类本身并不会被废弃），在新的代码中应该使用新的 SharedImage 接口。

在所有需要“分阶段”渲染的场合都可以使用 SharedImage 机制，在需要从 GPU 读取数据的场合也可以使用 SharedImage 机制。SharedImage 机制只提供内存的管理，应用可以使用常规的读写GPU数据的方式来读写SharedImage中的数据。

---------

参考文档：

- [Shared images and synchronization - Google Docs](https://docs.google.com/document/d/12qYPeN819JkdNGbPcKBA0rfPXSOIE3aIaQVrAZ4I1lM/edit#)
- [789238 - Use RasterDecoder instead of GLES2Decoder for oop-rasterization - chromium](https://bugs.chromium.org/p/chromium/issues/detail?id=789238)
