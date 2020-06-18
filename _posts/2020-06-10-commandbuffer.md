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

下面是 Command Buffer 相关的类图（点击可查看大图）：

[![command buffer](/data/command_buffer_gpu.svg)](/data/command_buffer_gpu.svg)

下面做简要的介绍：

TODO: 介绍大体流程

-------------

参考文档：

- [gpu/command_buffer/docs/gles2_cmd_format_docs.txt](https://chromium.googlesource.com/chromium/src/+/master/gpu/command_buffer/docs/gles2_cmd_format_docs.txt)
- [GPU Command Buffer - The Chromium Projects](https://www.chromium.org/developers/design-documents/gpu-command-buffer)
