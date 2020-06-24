---
title: 常用 GL extensions
published: false
tags:
  - graphics
  - opengl
  - egl
  - opengles
---

OpenGL 扩展的常见[前缀](https://www.opengl.org/archives/resources/features/OGLextensions/)：

- `ARB`: Extensions officially approved by the OpenGL Architecture Review Board
- `ANDROID`: Android 提供的扩展
- `ANGLE`: [ANGLE](https://github.com/google/angle) 项目提供的扩展
- `OES`: OpenGL ES 扩展
- `KHR`: Khronos-approved (ARB, OES, or KHR vendor suffixed) extensions
- `EXT`: Extensions agreed upon by multiple OpenGL vendors
- `HP`: Hewlett-Packard
- `IBM`: International Business Machines
- `INTEL`: Intel
- `NV`: NVIDIA Corporation
- `MESA`: Brian Paul’s freeware portable OpenGL implementation
- `SGI`: Silicon Graphics
- `SGIX`: Silicon Graphics (experimental)
- `WIN`: Microsoft

其中 `ARB`,`OES`,`KHR`,`EXT` 扩展大多可以跨平台，其他扩展大多都是平台相关的。

> OpenGL 其他前缀见 [OpenGL Registry](https://www.khronos.org/registry/OpenGL/extensions/)，EGL 其他扩展前缀见 [EGL Registry](https://www.khronos.org/registry/EGL/extensions/).

[Android 提供的EGL扩展](https://www.khronos.org/registry/EGL/extensions/ANDROID/)：

- EGL_ANDROID_blob_cache.txt
- EGL_ANDROID_create_native_client_buffer.txt
- EGL_ANDROID_framebuffer_target.txt
- EGL_ANDROID_front_buffer_auto_refresh.txt
- EGL_ANDROID_get_frame_timestamps.txt
- EGL_ANDROID_get_native_client_buffer.txt
- EGL_ANDROID_GLES_layers.txt
- EGL_ANDROID_image_native_buffer.txt
- EGL_ANDROID_native_fence_sync.txt
- EGL_ANDROID_presentation_time.txt
- EGL_ANDROID_recordable.txt

[Android 提供的OpenGL扩展](https://www.khronos.org/registry/OpenGL/extensions/ANDROID/)：

- ANDROID_extension_pack_es31a.txt

