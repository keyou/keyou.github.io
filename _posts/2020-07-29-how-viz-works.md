---
title: How viz works
published: true
categories:
  - chromium
  - graphics
tags:
  - chromium
  - graphics
  - viz
---

在 Chromium 中 `viz` 的核心逻辑运行在 GPU 进程中，负责接收其他进程产生的 `viz::CompositorFrame（简称 CF）`，然后把这些 CF 进行合成，并将合成的结果最终渲染在窗口上。

可以将这个过程拆解成以下几个步骤来分析：

0. `viz` 的初始化；
1. `viz` 的架构设计；
2. CF 的创建；
3. CF 的合成；
4. CF 的渲染；

`viz` 的初始化只是一次性的过程，大多数人应该都更加关心 `viz` 是如何执行渲染的，因此我把 `viz` 的初始化放在最后介绍。CF 是 `viz` 中最重要的数据结构，因此我们先来看下 CF 的创建。

## CF 的创建

一个 CF 表示一个矩形显示区域中的一帧画面。内部存储了 3 类数据，如下图所示：

![compositor frame](/data/viz-compositor-frame.svg)

### CF 的元数据（Metadata）

`viz::CompositorFrameMetadata` 记录了 CF 相关的元数据，比如画面的缩放级别，滚动区域，引用到的 Surface 等。需要注意的是这里面还存储了一个 `ui::LatencyInfo` 对象，该对象记录了从 Chromium 接收到用户的交互事件到画面更新每个阶段的延时信息，具体都有哪些时间被记录了下来，参见 [latency_info.h](https://source.chromium.org/chromium/chromium/src/+/master:ui/latency/latency_info.h;drc=8d3351f2b93dc4fef60b794c47ec22ebf54f3e8b;bpv=1;bpt=1;l=42)。通过 `ui::LatencyInfo` 对象可以方便的追踪交互到渲染各阶段的延时信息。

### CF 引用到的资源（Resource）

`viz::TransferableResource` 记录了该 CF 引用到的资源，所谓的资源可以理解为一张图片。资源有两种存在形式，一类是存储在内存中的 Software 资源，一类是存储在 GPU 中的 Texture（这样表达并不严格，但目前为止大家都可以先这样理解）。如果没有开启 Chromium 的硬件加速渲染，则只能使用 Software 资源。如果开启了硬件加速，则只能使用硬件加速的资源。

如何创建一个资源呢？
如果是 Software 资源，只需要申请一块内存（如果资源需要夸进程可以使用共享内存），然后将资源的像素数据放进去就可以了。就像下面这样：

```c++
// 使用共享内存来传递资源到 viz
viz::ResourceId CreateSoftwareResource(const gfx::Size& size,
                                        const SkBitmap& source) {
  TRACE_EVENT0("viz", "LayerTreeFrameSink::CreateSoftwareResource");
  viz::SharedBitmapId shared_bitmap_id = viz::SharedBitmap::GenerateId();
  // 创建共享内存
  base::MappedReadOnlyRegion shm =
      viz::bitmap_allocation::AllocateSharedBitmap(size, viz::RGBA_8888);
  base::WritableSharedMemoryMapping mapping = std::move(shm.mapping);

  SkImageInfo info = SkImageInfo::MakeN32Premul(size.width(), size.height());
  // 将 SkBitmap 中的像素数据拷贝到共享内存
  source.readPixels(info, mapping.memory(), info.minRowBytes(), 0, 0);

  // 将共享内存及与之对应的资源Id发送到 viz service 端
  frame_sink_remote_->DidAllocateSharedBitmap(std::move(shm.region),
                                              shared_bitmap_id);
  // 使用 SharedBitmap 创建资源
  viz::TransferableResource transferable_resource =
      viz::TransferableResource::MakeSoftware(shared_bitmap_id, size,
                                              viz::RGBA_8888);
  // 把资源存入 ClientResourceProvider 进行统一管理。
  // 后续会使用 ClientResourceProvider::PrepareSendToParent
  // 将已经存入的资源添加 到 CF 中。
  return client_resource_provider_->ImportResource(
      transferable_resource,
      viz::SingleReleaseCallback::Create(base::DoNothing()));
}
```

上面的代码直接将 `SkBitmap` 中的像素数据拷贝到共享内存中，然后在 `viz::TransferableResource` 中引用该共享内存的地址。

如果是 Texture 资源，就要复杂一点，因为需要先初始化 GL Context。关于如何初始化 GL Context 放在后面的 `viz` 初始化中解释，这里先忽略初始化过程直接看如何创建 GL 资源。示例代码如下：

```c++
viz::ResourceId CreateGpuResource(const gfx::Size& size,
                                  const SkBitmap& source) {
  TRACE_EVENT0("viz", "LayerTreeFrameSink::CreateGpuResource");
  DCHECK(context_provider_);
  gpu::SharedImageInterface* sii = context_provider_->SharedImageInterface();
  DCHECK(sii);

  auto pixels =
      base::make_span(static_cast<const uint8_t*>(source.getPixels()),
                      source.computeByteSize());
  auto format = viz::RGBA_8888;
  auto color_space = gfx::ColorSpace();
  // 直接使用像素数组创建 SharedImage。
  // 也可以先创建空的 SharedImage 然后再通过其他 API 来填充 SharedImage 的内容，
  // 目前有两种填充 SharedImage 内容的方法，OOP-D 和 OOP-R，详见
  // Chromium GPU Resource Share。
  gpu::Mailbox mailbox = sii->CreateSharedImage(
      format, size, color_space, gpu::SHARED_IMAGE_USAGE_DISPLAY, pixels);

  // 获取用于 GL 同步的 SyncToken，详见 Chromium GPU Synchronization
  gpu::SyncToken sync_token = sii->GenVerifiedSyncToken();

  // 使用 Mailbox 创建 GL 资源
  viz::TransferableResource gl_resource = viz::TransferableResource::MakeGL(
      mailbox, GL_LINEAR, GL_TEXTURE_2D, sync_token, size,
      false /* is_overlay_candidate */);
  gl_resource.format = format;
  gl_resource.color_space = std::move(color_space);
  auto release_callback = viz::SingleReleaseCallback::Create(
      base::BindOnce(&InkClient::DeleteSharedImage, base::Unretained(this),
                      context_provider_, mailbox));
  // 同上
  return client_resource_provider_->ImportResource(
      gl_resource, std::move(release_callback));
}
```

上面的代码将 `SkBitmap` 中的像素数据使用 `gpu::SharedImageInterface` 接口放入 `SharedImage` 中，然后使用它返回的 Mailbox 创建 `viz::TransferableResource`。就像代码中注释的那样，不一定非要使用像素数据来创建 SharedImage，也可以使用 OOP-D 和 OOP-R 相关接口来创建 SharedImage。关于 Mailbox，SharedImage 以及 OOP-D 和 OOP-R 的相关内容可以参考[Chromium GPU Resource Share (Shared Image)]({% post_url 2020-06-22-chromium-gpu-image-share %})。

哪些内容会以资源的形式存在呢？
结果可能出乎你的意料，网页中的每一个 Tile 都引用一个资源。`cc` 会把网页分成很多的 Tiles，每一个都以 `viz::TileDrawQuad` 的形式存在，而每一个 `viz::TileDrawQuad` 只是简单引用一个资源（关于 DrawQuad 见下文）。此时，这些**资源就是 `cc` Raster 的产物**。

### CF 包含的绘制操作（RenderPass/DrawQuad）

`viz::RenderPass` 由一系列相关的 `viz::DrawQuad` 构成。可以对一个 RenderPass 单独应用特效，变换，mipmap，缓存，截图(CopyOutputRequest)等。DrawQuad 有很多种类型，分别用于不同的目的：

- `viz::TextureDrawQuad` 内部引用一个资源。
- `viz::TileDrawQuad` 表示一个 Tile 块（Tile的概念见`cc`），和 TextureDrawQuad 类似，内部也引用一个资源，DisplayItemList 会被 `cc` Raster 为 TileDrawQuad；
- `viz::PictureDrawQuad` 内部直接存放 DisplayItemList，但是目前只能用于Android WebView；
- `viz::SolidColorDrawQuad` 表示一个颜色块；
- `viz::RenderPassDrawQuad` 内部引用另外一个 RenderPass 的 Id；
- `viz::SurfaceDrawQuad` 内部保存一个 viz::SurfaceId，该 Surface 的内容由其他 `CompositorFrameSinkClient` 创建，**用于 viz 的嵌套**，比如 OOPIF,OffscreenCanvas等；

由于 `viz::RenderPassDrawQuad` 的存在使得 CF 中可以存储一颗 RenderPass 树，由于 `viz::SurfaceDrawQuad` 的存在使得viz可以实现UI的夸进程嵌套。

关于这些 RenderPass 和 DrawQuad 的应用示例见 [chromium_demo/demo_viz_layer.cc at c/80.0.3987 · keyou/chromium_demo](https://github.com/keyou/chromium_demo/blob/c/80.0.3987/demo_viz/demo_viz_layer.cc)。

### 小结1

CF 是 `viz` 中的核心数据结构，它代表某块区域中UI的一帧画面，使用 DrawQuad 来存储 UI 要显示的内容。它代表了 viz 运行时的数据流。

## `viz` 的架构设计

### `viz` 的三个端

在设计层面上 viz 的架构如下图所示：

![viz client](/data/viz-client-host-service.svg)

在设计上 `viz` 分了三个端，分别是 client 端, host 端和 service 端。

`client` 端用于生成要显示的画面(CF)。应用中至少有一个 root client，可以有多个 child client，它们组成了一个 client 树，每个 Client 都有一个 FrameSinkId 和一个 LocalSurfaceId，如果父子 client 之间的 UI 需要嵌入，则子 client 作为 SurfaceDrawQuad 嵌入到父 client 中。在 Chromium 中，每一个浏览器窗口都对应一个 client 树，拥有一个 root client 和零个或多个子 client。比如，网页中的一个 OOPIF 可以是一个子 client，OffscreenCanvas 也是一个子 client。每个 client 可以独立进行刷新，生成独立的 CF。client 端的核心接口为 `viz::mojom::CompositorFrameSinkClient`， 开发者可以通过继承该类来创建一个 client。

`host` 端用于将 client 注册到 service，只能运行在特权端（没有沙箱，比如浏览器进程），负责协助 client 建立起和 service 的连接，也负责建立 client 和 client 之间的树形关系。所有的 client 想要生效必须经过 host 进行注册。host 端的核心接口为 `viz::mojom::FrameSinkManagerClient`，以及其实现类 `viz::HostFrameSinkManager`。

`service` 端运行 Viz 的内核，进行 UI 合成以及最终的渲染。它接收所有 client 端生成的 CF，然后把这些 CF 进行合成，并最终显示在窗口中。它内部会为每个 client 创建一个 `viz::CompositorFrameSink(Impl/Support)`，然后通过 `viz::Display` 将这些 CF 进行合成，最后通过 `viz::DirectRenderer` 将这些 CF 渲染到 `viz::OutputSurface` 中，`viz::OutputSurface` 包装了各种渲染目标，比如窗口，内存等。

在 Chromium 的具体实现中 `viz` 的架构可以用下图表示：

![viz archtecture](/data/viz-archtecture.svg)

绿色部分为 client 的实现，负责提交 CF 到 GPU, 橙色部分 `ui::Compositor` 为 host 的实现，负责将所有的 client 注册到 service，红色部分为 service 端的实现，负责接收、合成，并最终渲染 CF。

### `viz` 的线程架构

viz 是多线程架构，其中最重要的有两个线程，一个是 `VizCompositorThread`，也称为（viz的） Compositor 线程（注意和 cc 中的 Compositor 线程区分，它们没有关系）。另一个是 `CrGpuMain` （多进程架构下）或者 `Chrome_InProcGpuThread`（单进程架构下）线程，也称为 GPU 线程(只有在开启了硬件加速渲染时才存在)。帧率的调度，CF 的合成，DrawQuad 的绘制都发生在Compositor线程中，`viz::Display`, `viz::DisplayScheduler`，`viz::SurfaceAggregator`, `viz::DirectRenderer` 也运行在该线程中。而所有最终真实的绘制（比如 GL 的执行，Real SwapBuffer）最终都运行在 `CrGpuMain` 或 `Chrome_InProcGpuThread` 线程中。

如果要在架构上体现这两个线程，可以这样划分：

![viz archtecture](/data/viz-threads.svg)

虚线之上运行在 Compositor 线程，虚线之下运行在 GPU 线程，OutputSurface 需要对接 DirectRenderer 和最下层的渲染，所以不同部分运行在不同的线程中。

目前 Compositor 线程和 GPU 线程之间的数据传递有两种方式，一种是先将 GL 调用放入进程内的 CommandBuffer，然后在 GPU 线程中取出 GL 命令并执行，GLOutputSurface 使用这种方式。另一种是直接通过 PostTask 将要渲染的操作发送到 GPU 线程，SkiaOutputSurface 使用这种方式（关于 GLOutputSurface 和 SkiaOutputSurface 见下文）。

### `viz` 的类图

下面是 viz 更详细的类图：

[![viz archtecture](/data/viz-classes.svg)](/data/viz-classes.svg)

- `viz::mojom::CompositorFrameSinkClient` 作为 client，表示一个画面的来源；
- `viz::CompositorFrameSink(Impl/Support)` 用于处理 CF 的地方，一个 client 可以有多个 CompositorFrameSink；
- `viz::RootCompositorFrameSinkImpl` 作用和 CompositorFrameSink 类似，只不过专门处理 root client，每个 client 有且只有一个该对象。它还负责为对应的 client 初始化渲染环境，包括 OutputSurface， Display 的创建。
- `viz::FrameSinkManagerImpl` 用于管理 CompositorFrameSink, 包括其创建和销毁；
- `viz::SurfaceAggregator` 负责 Surface/CF 的合成，比如dirty区域的计算等，不负责绘制；
- `viz::OutputSurface` 封装渲染目标，和各平台的渲染目标直接对接；
- `viz::DirectRenderer` 封装绘制 DrawQuad 的方式，负责将 DrawQuad 绘制到 OutputSurface 上；
- `viz::Display` 一个中控类，将 SurfaceAggregator/DirectRenderer 以及 Overlay 的功能串起来形成流水线；
- `viz::DisplayScheduler` 调度 `viz::Display` 何时应该采取行动；

### `viz` 的渲染目标

`viz::DirectRenderer` 和 `viz::OutputSurface` 用于管理渲染目标 。他们对理解 Chromium UI 的呈现方式至关重要。这两个类并不是相互独立的，在 Chromium 中他们有以下组合：

1. `viz::GLRenderer` + `viz::GLOutputSurface`  
  GLRenderer 已经被标记为 deprecated, 未来会被 SkiaRenderer 取代。它使用基于 CommandBuffer 的 GL Context 来渲染 DrawQuad 到 GLOutputSurface 上，GLOutputSurface 使用窗口句柄创建 Native GL Context。GL 调用发生在 VizCompositorThread 线程中，通过 InProcessCommandBuffer 这些 GL 调用最终在 `CrGpuMain` 线程中执行。关于 CommandBuffer 相关内容可以参考 [Chromium Command Buffer]({% post_url 2020-06-10-commandbuffer %})。  
  GLOutputSurface 有一系列的子类，不同的子类对接不同平台的渲染目标，比如 GLOutputSurfaceAndroid 用于对接Android平台的渲染，GLOutputSurfaceOffscreen 用于支持 GL 的离屏渲染等。

2. `viz::SkiaRenderer` + `viz::SkiaOutputSurface(Impl)` + `viz::SkiaOutputDevice`  
  SkiaRenderer 是未来的发展方向，以后所有其他的渲染方式都会被这种方式取代。因为它具有最大的灵活性，同时支持GL渲染，Vulkan渲染，离屏渲染等。  
  SkiaRenderer 将 DrawQuad 绘制到由 SkiaOutputSurfaceImpl 提供的 canvas 上，但是该 canvas 并不会进行真正的绘制动作，而是通过 skia 的 ddl(SkDeferredDisplayListRecorder) 机制把这些绘制操作记录下来，等到所有的 RenderPass 绘制完成，这些被记录下来的绘制操作会被通过 `SkiaOutputSurfaceImpl::SubmitPaint` 发送到 `SkiaOutputSurfaceImplOnGpu` 中进行真实的绘制，根据名字可知该类运行在 GPU 线程中。  
  SkiaOutputSurface 对渲染目标的控制是通过 SkiaOutputDevice 实现的，后者有很多子类，其中 SkiaOutputDeviceOffscreen 用于实现离屏渲染，SkiaOutputDeviceGL 用于GL渲染。  

3. `viz::SoftwareRenderer` + `viz::SoftwareOutputSurface` + `viz::SoftwareOutputDevice`  
   SoftwareRenderer 用于纯软件渲染，当关闭硬件加速的时候使用该种渲染方式。这种方式逻辑相对简单，因此留给读者去探索

### `viz` 的数据流

`viz` 中的核心数据为 CF，当用户和网页进行交互时，会触发 UI 的改变，这最终会导致 viz client （比如 cc::LayerTreeHostImpl） 创建一个新的 CF 并使用 `viz::mojom::CompositorFrameSink` 接口将该 CF 提交到 service，service 中的 `viz::CompositorFrameSinkSupport` 会为该 CF 所属的 client 创建一个 `viz::Surface`,然后把 CF 放入该 `viz::Surface` 中。当 servcie 中的 `viz::DisplayScheduler` 到达下一个调度周期的时候，会通知 `viz::Display` 取出当前的 `viz::Surface`，并交给 `viz::SurfaceAggregator` 进行合成，合成的结果会被交给 `viz::DirectRenderer` 进行渲染。`viz::DirectRenderer` 并不直接渲染，它从 `viz::OutputSurface` 中取出渲染表面，然后让其子类在该渲染表面上进行绘制。`viz::OutputSurface` 封装了渲染表面，比如窗口，内存bitmap等。绘制完成之后，`viz::Display` 会调用 `viz::DirectRenderer::SwapBuffer` 将该 CF 最终显示出来。

service 绘制 CF 的核心逻辑位于 `viz::Display::DrawAndSwap` 中，下面是其主要的执行逻辑：

```c++
Display::DrawAndSwap
  DirectRenderer::DrawFrame
    DirectRenderer::DrawRenderPassAndExecuteCopyRequests
      DirectRenderer::DrawRenderPass
        SkiaRenderer::FinishDrawingQuadList
          SkiaOutputSurfaceImpl::SubmitPaint
            if 在绘制 root，则设定 deferred_framebuffer_draw_closure（简称 DFDC) 为 SkiaOutputSurfaceImplOnGpu::FinishPaintCurrentFrame,
            else ScheduleGpuTask(SkiaOutputSurfaceImplOnGpu::FinishPaintRenderPass)
  SkiaRenderer::SwapBuffers
    SkiaOutputSurfaceImpl::SwapBuffers
      ScheduleGpuTask(SkiaOutputSurfaceImplOnGpu::SwapBuffers+DFDC)
        gpu-thread SkiaOutputSurfaceImplOnGpu::SwapBuffers
          if 有 DFDC，则执行 DFDC
            SkiaOutputSurfaceImplOnGpu::FinishPaintCurrentFrame
          if 帧支持局部刷新，则使用 SkiaOutputDeviceGL::PostSubBuffer 或者 SkiaOutputDeviceGL::CommitOverlayPlanes 进行局部刷新
          else SkiaOutputDeviceGL::SwapBuffers
            if gl::GLSurface 支持异步刷新，则 gl::GLSurface::SwapBuffersAsync(SkiaOutputDeviceGL::DoFinishSwapBuffers)
            else gl::GLSurface::SwapBuffers + SkiaOutputDeviceGL::FinishSwapBuffers
```

### `viz` 的分层

从分层架构的角度看，viz 的 API 分了3层，分别为最底层核心实现，中间层mojo接口，最上层viz服务。

最底层是viz的核心实现，主要接口包括 `viz::FrameSinkManager(Impl)`, `viz::CompositorFrameSink(Support)`, `viz::Display`, `viz::OutputSurface`。
使用这些接口是使用 viz 最直接的方式，提供最大的灵活性。但这层接口不提供夸进程通信的能力。
Chromium中直接使用这一层的接口的地方不多，具体demo参考 [chromium_demo/demo_viz_offscreen.cc at c/80.0.3987 · keyou/chromium_demo](https://github.com/keyou/chromium_demo/blob/c/80.0.3987/demo_viz/demo_viz_offscreen.cc)。

中间mojo层主要将底层API包装到 mojo 接口中，这一层的核心接口包括 `viz::HostFrameSinkManager`,`viz::mojom::FrameSinkManager`,`viz::mojom::CompositorFrameSink`,`viz::mojom::CompositorFrameSinkClient`,`viz::HostDisplayClient`。这些接口大多对应底层的 API 接口，将对底层接口的调用转换为对mojo接口的调用，因此这层接口提供夸进程通信的能力。Chromium中几乎所有使用viz的地方都是使用该层接口。viz模块提供的官方demo也是使用该层接口，也可以参考我写的单文件demo，见 [chromium_demo/demo_viz_gui.cc at c/80.0.3987 · keyou/chromium_demo](https://github.com/keyou/chromium_demo/blob/c/80.0.3987/demo_viz/demo_viz_gui.cc)。

最上层viz服务接口主要将 `viz` 服务化，提供将viz运行在独立进程的能力。这一层的主要接口包括 `viz::GpuHostImpl`, `viz::GpuServiceImpl`, `viz::VizMainImpl`, `viz::Gpu`。这些接口需要和中间层mojo接口配合才能起作用，在 Chromium 的多进程架构中使用了该层接口。使用该层接口的demo参考 [chromium_demo/demo_viz_gui_gpu.cc at c/80.0.3987 · keyou/chromium_demo](https://github.com/keyou/chromium_demo/blob/c/80.0.3987/demo_viz/demo_viz_gui_gpu.cc)。

另外，在 viz 中还有一套专门用于 `viz` 中 GPU 渲染的接口 `viz::*ContextProvider`。它主要负责为 viz 初始化 GL 环境，使 viz 可以使用 GPU 进行渲染。

### `viz` 的 Overlay 机制

TODO: 完善 Overlay 文档

### `viz` 的 ID 设计

每一个 client 都至少对应一个 `FrameSinkId` 和 `LocalSurfaceId`，在 client 的整个生命周期中所有 FrameSink 的 `client_id`（见下文）都是固定的，而 `LocalSurfaceId` 会根据 client 显示画面的 size 或 scale factor 的改变而改变。他们两个共同组成了 `SurfaceId`，用于在 service 端全局标识一个 `Surface` 对象。也就是说对于每一个 `Surface`，都可以获得它是由谁在什么 size 或 scale facotr 下产生的。

#### `FrameSinkId`

- `client_id` = uint32_t, 每个 client 都会有唯一的一个 ClientId 作为标识符，标识一个 CompositorFrame 是由哪个 client 产生的，也就是标识 CompositorFrame 的来源；
- `sink_id` = uint32_t, 在 service 端标识一个 CompositorFrameSink 实例，Manager 会为每个 client 在 service 端创建一个 CompositorFrameSink，专门用于处理该 client 生成的 CompositorFrame，也就是标识 CompositorFrame 的处理端；
- `FrameSinkId = client_id + sink_id`，将 CF 的来源和处理者关联起来。

FrameSinkId 可以由 `FrameSinkIdAllocator` 辅助类生成。

在实际使用中，往往一个进程是一个 client，专门负责一个业务模块 UI 的实现，而一个业务往往由很多的 UI 元素组成，因此可以让每个 FrameSink 负责一部分的 UI，此时就用到了单 client 多 FrameSink 机制，这种机制可以实现 UI 的局部刷新（多 client 也能实现这一点）。在 chrome 中浏览器主程序是一个 client，而每一个插件都是一个独立的 client，网页中除了一些特殊的元素比如 iframe和offscreen canvas位于单独的client中，其他元素全部都位于同一个client中。

#### `LocalSurfaceId`

- `parent_sequence_number` = uint32_t, 当自己作为父 client，并且 surface 的 size 和 device scale factor 改变的时候改变；
- `child_sequence_number` = uint32_t, 当自己作为子 client，并且 surface 的 size 和 device scale factor 改变的时候改变；
- `embed_token` = 可以理解为一个随机数， 用于避免 LocalSurafaceId 可猜测，当父 client 和子 client 的父子关系改变的时候改变；
- `LocalSurfaceId = parent_sequence_number + child_sequence_number + embed_token`，当 client 的 size 当前 client 产生的画面改变。

LocalSurfaceId 可以由 `ParentLocalSurfaceIdAllocator` 或者 `ChildLocalSurfaceIdAllocator` 这两个辅助类生成。前者用于由父 client 负责生成自己的 LocalSurfaceId 的时候，后者用于由子 client 自己负责生成自己的 LocalSurfaceId 的时候。使用哪种方式要看自己的 UI 组件之间的依赖关系的设计。

LocalSurfaceId 在很多时候都包装在 `LocalSurfaceIdAllocation` 内，该类记录了 LocalSurfaceId 的创建时间。改时间在创建 CF 的时候需要用到。

#### `SurfaceId`

`SurfaceId = FrameSinkId + LocalSurfaceId`

SurfaceId 全局唯一记录一个显示画面，它可以被嵌入其他的 CF 或者 RenderPass 中，从而实现显示界面的嵌入和局部刷新。除了在 Client 端使用 SurfaceDrawQuad 进行 Surface 嵌套之外，大部分应用场景都在 service 端。

### 小结2

`viz` 作为 Chromium 渲染的底层框架，接收 `cc` 产出的 CF，然后将进行渲染，一般情况下都不需要动到 `viz`，但是如果想要实现某部分UI的独立渲染，或者想要避免 cc 带来的性能损失，可以直接使用 viz 提供的接口。

## CF 的合成

CF 合成指的是把 CF 中的内容或者多个 CF 合成到一起，形成一个完整的画面。这部分功能主要依赖 viz 中的两部分设计。一个是 client 的嵌套，也就是 `viz::SurfaceDrawQuad`，一个是 `viz::SurfaceAggregator`。前者我们已经介绍过，这里主要分析后者。

TODO: 完善 `viz::SurfaceAggregator` 的内容。

## CF 的渲染

CF 的渲染主要由 `viz::DirectRenderer` 和 `viz::OutputSurface` 负责，他们将合成的结果渲染到程序选择的渲染目标上去。

TODO: 完善渲染文档。

## `viz` 的初始化

这部分主要讲解 Chromium 初始化 viz 的过程，由于这个过程是固定的，讲起来就是流水帐，所以建议大家直接阅读 demo 源码 [chromium_demo/demo_viz_layer.cc at c/80.0.3987 · keyou/chromium_demo](https://github.com/keyou/chromium_demo/blob/c/80.0.3987/demo_viz/demo_viz_layer.cc)。这个 demo 使用了 viz 的最上层 servcie 接口，和 Chromium 的用法没有太大的差别。

## `viz` 的应用实例

- 如何实现 OffscreenCanvas；
- 如何实现 Tab/Window Capture;

-----

参考文档：

- [components/viz - chromium/src - Git at Google](https://chromium.googlesource.com/chromium/src/+/master/components/viz/)
- [Viz Services Directory Structure](https://chromium.googlesource.com/chromium/src/+/master/services/viz/README.md)
- [Compositor Stack - Google Slides](https://docs.google.com/presentation/d/1ou3qdnFhKdjR6gKZgDwx3YHDwVUF7y8eoqM3rhErMMQ/edit#slide=id.g17a920a23a_0_676)
