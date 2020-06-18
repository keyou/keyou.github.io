---
title: Command Buffer 原理解析
tags:
  - chromium
  - graphics
  - commandbuffer
---

`Command Buffer` 是支撑 Chromium 多进程硬件加速渲染的核心技术之一。它基于 OpenGLES2.0 定义了一套序列化协议，这套协议规定了所有 OpenGLES2.0 命令的序列化格式，使得应用对 OpenGL 的调用可以被缓存并传输到其他的进程中去执行（GPU进程），从而实现多个进程配合的渲染机制。

## Command Buffer 命令的序列化

在 CommandBuffer 中共有三类命令，一类是直接对接OpenGLES的命令。例如下面的GL命令：

```c++
void glActiveTexture(GLenum texture);
void glDrawArrays(GLenum mode, GLint first, GLsizei count);
```

他们的序列化格式定义为：

```c++
struct CommandHeader {
  uint32 size:21;     // 命令的长度
  uint32 command:11;  // 命令Id
};

// void glActiveTexture(GLenum texture)
struct ActiveTexture {
  static const CommandId kCmdId = 256;
  CommandHeader header;
  uint32 texture;  //!< GLenum
};

// void glDrawArrays(GLenum mode, GLint first, GLsizei count)
struct DrawArrays {
  static const CommandId kCmdId = 305;
  CommandHeader header;
  uint32 mode;  //!< GLenum
  int32 first;  //!< GLint
  int32 count;  //!< GLsizei
};
```

可以看到序列化的方式并不复杂，每个命令都有个 CommandId 和 header。header 占用 32 位，高 21 位表示当前命令的长度，低 11 位表示命令的 Id。其他字段表示命令的参数。

第二类命令是 CommandBuffer 自己需要用到的，这类命令称为公共命令（common command，Id<256），比如下面这两个：

```c++
struct SetBucketSize {
  static const CommandId kCmdId = 8;

  CommandHeader header;
  uint32 bucket_id;
  uint32 size;
};
struct SetBucketData {
  static const CommandId kCmdId = 9;

  CommandHeader header;
  uint32 bucket_id;
  uint32 offset;
  uint32 size;
  uint32 shared_memory_id;
  uint32 shared_memory_offset;
};
```

这两个命令一个用于创建`Bucket`，一个用于向`Bucket`中放数据（关于`Bucket`后面会讲到）。他们都不是 OpenGL 定义的命令，是 CommandBuffer 为了自己的某种用途而添加的命令。

最后一种命令如下：

```c++

// glShaderSource 的定义：
// void glShaderSource(GLuint shader,GLsizei count, const GLchar **string, const GLint *length);

// 方法1
struct ShaderSource {
  static const CommandId kCmdId = 367;

  CommandHeader header;
  uint32 shader;  //!< GLuint
  uint32 data_shm_id;  //!< uint32
  uint32 data_shm_offset;  //!< uint32
  uint32 data_size;  //!< uint32
};

// 方法2
struct ShaderSourceBucket {
  static const CommandId kCmdId = 435;

  CommandHeader header;
  uint32 shader;  //!< GLuint
  uint32 data_bucket_id;  //!< uint32
};
```

可以看到 CommandBuffer 针对 `glShaderSource` 命令定义了不同的序列化格式，没有一种是按照原本的参数来定义的，这主要是因为 `glShaderSource` 命令可能会传输比较大的数据（第3个参数），如果直接把数据通过 IPC 传输可能会比较低效，因此方法1将数据存放在了共享内存，然后在命令中保留了对共享内存的引用，方法2是将数据保存在 Bucket，然后在命令中引用了 Bucket。这种处理方式主要是针对那些需要传输大批量数据的GL命令。

> 完整的 Command Buffer 序列化格式的定义见 [gpu/command_buffer/docs/gles2_cmd_format_docs.txt](https://chromium.googlesource.com/chromium/src/+/master/gpu/command_buffer/docs/gles2_cmd_format_docs.txt)。

## Command Buffer 的架构设计

前面已经提到，Command Buffer 主要是为了解决多进程的渲染问题，因此它在设计上分两个端，分别是 client 端和 service 端。下图反映了 Chromium 中各种进程和 Command Buffer 中两个端的对应关系：

![command buffer service&client](/data/command_buffer_service_client.svg)

可以看到，Browser和Render进程都是 client 端， GPU 进程是 service 端。client 端负责调用 GL 命令来产生绘制操作，但是这些GL命令并不会真正执行而是被序列化为 Command Buffer 命令，然后通过 IPC 传输到 GPU 进程，GPU 进程负责反序列化 Command Buffer 命令并最终执行 GL 调用。

在 Chromium 的实现中，引入了更多的概念：

- 每个client端和server端之间都通过 `IPC channel (IPC::Channel`) 通道进行连接。
- 每个 IPC channel 可以有多个`调度组（scheduling groups）`，每个调度组称为一个 `stream`，每个 stream 有自己的调度优先级。
- 每个 stream 可以承载多个 `command buffer`。
- 每个 command buffer 都对应一个 GL context，在相同stream中的GL Context都属于同一个`share group`。
- 每个 command buffer 都包含一系列的 GL 命令。

下图反映了 context,commandbuffer,stream,channel 之间的关系：

![command buffer](/data/commandbuffer.svg)

> 关于 `share group` 参考 [OpengGL 中的同步及资源共享]({% post_url 2020-06-12-opengl-sync %})

下面是 Command Buffer 的模块依赖关系：

![command buffer](/data/command_buffer_components.svg)

content 模块通过调用 GL 或者 Skia 来产生 GL 命令， 然后 Command Buffer client 将这些 GL 命令序列化，然后通过 IPC 传输到了 Command Buffer service 端，service 将命令反序列化然后调用 `ui/gl` 模块执行真正的 GL 调用。

## Command Buffer 命令的传输方式

Command Buffer 定义了三种命令传输方式：

1. 命令和命令涉及到的数据都直接放在 Command Buffer 中传输，在多进程模式下 Command Buffer 本身位于共享内存中；
2. 命令放在 Command Buffer 中，数据放在共享内存中，在命令中引用该共享内存；
3. 先使用 `SetBucketSize` 命令在 service 进程中创建一个足够大的 `Bucket`，然后将数据的一部分放在共享内存中，然后使用 `SetBucketData` 命令将该共享内存中的数据放到 service 进程的 Bucket 中，然后再放一部分数据到共享内存，再使用 `SetBucketData` 命令将数据传输到 service 进程中，循环这个操作直到将所有的数据都放到 service 进程中，最后调用原本的 GL 命令并引用这个 Bucket 的 Id 。Bucket 机制主要用在共享内存不足以存放所有要传输的数据的时候。由于涉及到多次数据从共享内存拷贝到进程空间的操作，因此性能较低。

## Command Buffer 的具体实现

下面是 Command Buffer 及相关模块的类图（点击查看大图）：

[![command buffer](/data/command_buffer_gpu.svg)](/data/command_buffer_gpu.svg)

下面针对 client 端和 service 端对主要的流程进行介绍。

### client 端

client 端的核心接口是 `viz::(Raster)ContextProvider`，它负责创建并维护client端的Context，它的各种实现依赖另外的两个接口 `gpu::gles2::GLES2Interface` 和 `gpu::CommandBuffer`， 前者定义了所有client端的GL API，作为client端的 GL Context被使用，它内部大部分GL API 的定义是通过脚本自动生成的，后者在client端维护传输CommandBuffer所需要的内存。也就是说所有通过 `gpu::gles2::GLES2Interface` 接口调用的 GL 命令最终都被序列化到了 `gpu::CommandBuffer` 维护的内存中。

记作 `viz::(Raster)ContextProvider` = `gpu::gles2::GLES2Interface` + `gpu::CommandBuffer`。

`viz::(Raster)ContextProvider` 接口有很多实现，每个实现都用于不同的场景，下面给出了各种实现依赖的 `GLES2Interface` 以及 `CommandBuffer` 的组合：

- `viz::ContextProviderCommandBuffer` = (`gpu::webgpu::WebGPUImplementation` \| `gpu::raster::RasterImplementation` \| `skia_bindings::GLES2ImplementationWithGrContextSupport` \| `gpu::gles2::GLES2Implementation`) + `gpu::CommandBufferProxyImpl`
  
  用于正式产品中大部分的client端，除了android_webview。

- `viz::VizProcessContextProvider` = `skia_bindings::GLES2ImplementationWithGrContextSupport` + `gpu::InProcessCommandBuffer`

  用于viz进程内部，主要给DisplayCompositor使用，运行在service端。

- `viz::DirectContextProvider` = `gpu::gles2::GLES2Implementation` + `gpu::CommandBufferDirect`

  内部同时维护了GLES2的Impl和decoder，用于在一个线程中执行原本应该在多进程中执行的GL调用，它允许在单线中使用`GLES2Interface`接口，但是由于内部有GL的序列化和反序列化因此效率没有native GL高（在这种场景下是可以直接使用native GL的），因此在chromium中只有开启`GL Readback` 的时候才被用于 `SkiaOutputSurfaceImplOnGpu`，而该选项默认是关闭的，关闭后使用`Skia for Readback`。

- `android_webview::AwRenderThreadContextProvider` = `gpu::GLInProcessContext`(`skia_bindings::GLES2ImplementationWithGrContextSupport` + `gpu::InProcessCommandBuffer`)

  用于 android webview。

- `ui::InProcessContextProvider` = `gpu::GLInProcessContext`(`skia_bindings::GLES2ImplementationWithGrContextSupport` + `gpu::InProcessCommandBuffer`)

  用于在单进程中的测试或者demo程序。

这里我们主要关注 `viz::ContextProviderCommandBuffer` 类，网页/UI的绘制，栅格化(Raster)，视频渲染都会用到它。它提供`gpu::gles2::GLES2Implementation` 类作为client端的GLContext，所有通过该Context发起的GL调用都会通过 `gpu::gles2::GLES2CmdHelper` 接口被序列化到CommandBufer中，然后通过`gpu::GpuChannelHost`接口发送到service端。`gpu::GpuChannelHost`维护了到service端的IPC通道(Channel)，它是通过`viz::mojom::GpuSerice` mojo 接口建立起来的。关于Browser进程和Gpu进程之间的mojo通道的建立见其它文档。

Render进程也是client端，它和service之间IPC通道的建立不是通过`viz::mojom::GpuSerice`，而是通过Browser进程提供的`viz::mojom::Gpu`mojo接口做中间代理实现的，`viz::mojom::Gpu` 把Render的IPC建立请求转发到 GpuSerice 接口，从而为Render建立和service端的连接。这主要是为了安全考虑，Render进程由于包含网页内容，因此更加不可信。

一旦IPC通道建立成功之后，Render和Browser进程对于service端就没有任何差异了，因为它们都使用同样的`viz::ContextProviderCommandBuffer`类。

client端可以创建多个 `viz::ContextProviderCommandBuffer`，每个都拥有一个`stream id`,拥有相同`stream id`的`viz::ContextProviderCommandBuffer`属于同一个`stream`，每个`gpu::CommandBufferProxyImpl`都拥有一个不同的`route id`。

需要注意的是，client端并不会每产生一条CommandBuffer命令都马上发送到service端，只有当client调用 `Flush` 命令的时候，CommandBuffer中的数据才会被“告知”servcie端。这里说“告知”是因为 CommandBuffer 存在于共享内存中，所以不存在将共享内存发送到service的情况，所谓的发送其实只是告诉service有新的数据了，然后service会从共享内存读取新的命令。通知service有新命令的IPC消息为 `GpuCommandBufferMsg_AsyncFlush`。

### service 端

当 service 端收到 client 建立 IPC(channel) 通道请求的时候，service 端会通过 `gpu::GpuChannelManager` 类建立一个 `gpu::GpuChannel` 对象，它维护了到client端的IPC通道。

针对客户端的每个`stream`，service都会通过`gpu::Scheduler`创建一个`gpu::Scheduler::Sequence`,后续该stream上的IPC消息都会放入对应的Sequence，然后被Scheduler按照优先级进行调度。

针对客户端的每个CommandBuffer，`gpu::GpuChannel` 都会创建一个 `gpu::CommandBufferStub` 对象，用来接收由 CommandBufferProxyImpl 发送来的 CommandBuffer 命令。当收到 `GpuCommandBufferMsg_AsyncFlush` 命令的时候，会通过 `gpu::CommandBufferService::Flush` 方法调用 `gpu::DecoderContext::DoCommands` 方法进行命令的反序列化，参数校验以及执行。

有两个主要的反序列化实现类，`gpu::GLES2DecoderImpl` 和 `gpu::GLES2DecoderPassthoughImpl`。两者都用于`CommandBuffer`命令的解码（反序列化），类名不带Passthough表示在GL命令解码之后内部会对GL命令的参数进行校验，如果校验不通过会返回错误，类名带 Passthough 表示内部不进行进行GL命令参数的合法性校验，会直接返回成功。

另外，service 端并没有直接对接native GL，而是使用 `ui/gl` 模块来调用native GL。

## ui/gl 对 native GL 的抽象

由于不同平台支持的 native GL 差异很大，初始化方法也各不相同，因此为了屏蔽各个平台的细节，`ui/gl` 模块将 native GL 抽象为了4个主要接口：

- `gl::GLContext` 抽象了各平台的 GL Conetxt，负责初始化 Context 以及 Context 的切换。
- `gl::GLSurface` 抽象了各平台的GL渲染表面，一般各平台都会通过提供窗口的handle来提供渲染表面。
- `gl::GLApi` 该类由脚本生成，定义了所有的GL接口，用动态加载的方式对接native的GL API，所有对GL的调用都通过它发起。根据平台不同查找 native GL API 的动态库包括 libGLESv2.so,libGLESv2.so.2,libGLESv2.dll 等。
- `gl::EGLApi` 该类由脚本生成，定义了所有的EGL接口，用动态加载的方式对接native的EGL API，所有对EGL的调用都通过它发起。根据平台不同查找 EGL API 的动态库包括 libEGL.so,libEGL.so.1,libEGL.dll。在windows上还要加载 ddraw.dll 和 D3DCompiler_47.dll。

> Windows 平台上对 EGL 和 GLES 的支持是通过 `ANGLE` 项目实现的，详见 ANGLE 相关文档。

当 GPU 进程启动的时候会通过 `gpu::GpuInit` 来初始化 `ui/gl` 模块。

## 总结

Command Buffer 可以用于实现多进程的渲染架构，并且提供全平台支持。可以通过设置 `is_component_build=true` 来将 Command Buffer 模块编译为动态链接库，从而嵌入到自己的项目中。例如，Skia 项目就提供了对 Command Buffer 的支持。

-------------

参考文档：

- [gpu/command_buffer/docs/gles2_cmd_format_docs.txt](https://chromium.googlesource.com/chromium/src/+/master/gpu/command_buffer/docs/gles2_cmd_format_docs.txt)
- [GPU Command Buffer - The Chromium Projects](https://www.chromium.org/developers/design-documents/gpu-command-buffer)
