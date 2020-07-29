---
title: Chromium GPU Workarounds
tags:
  - chromium
  - graphics
  - gpu
---

## GPU Workarounds

`//gpu` 模块为了兼容不同的GPU，维护了一长串 GPU 兼容(Workarounds)列表，这个列表记录在json文件中，例如：

```json
// file path: gpu/config/gpu_driver_bug_list.json
{
  "name": "gpu driver bug list",
  "entries": [
    ...
   {
      "id": 109,
      "cr_bugs": [449150, 514510],
      "description": "MakeCurrent is slow on Linux with NVIDIA drivers",
      "vendor_id": "0x10de",
      "os": {
        "type": "linux"
      },
      "gl_vendor": "NVIDIA.*",
      "features": [
        "use_virtualized_gl_contexts"
      ]
    },
    ...
  ]
}
```

> 该json文件的详细格式定义见 `gpu/config/gpu_control_list_format.txt`。

上面的 json 描述的意思是如果当前系统是`linux`，gl厂商是 NVIDIA，厂商id是 `0x10de` 时，需要启用 `use_virtualized_gl_contexts` 功能（使用虚拟Context）。

该json文件会在编译时被 `gpu/config/process_json.py` 转换成C++代码，并在程序运行时被保存在 `gpu::GpuFeatureInfo::enabled_gpu_driver_bug_workarounds` 中。
另外，在 `third_party/skia/src/gpu/gpu_workaround_list.txt` 和 `gpu/config/gpu_workaround_list.txt` 文件中记录了 Chromium 支持的所有的兼容列表。

可以通过命令行等方式来覆盖默认的兼容列表，比如启动时添加 `--use_virtualized_gl_contexts=0` 可以强制关闭虚拟Context的使用。

在 `chrome://gpu` 页面中可以查看当前应用的所有workarounds，下面是在我电脑上的检测结果：

Driver Bug Workarounds

- adjust_src_dst_region_for_blitframebuffer
- clear_uniforms_before_first_program_use
- disable_discard_framebuffer
- exit_on_context_lost
- force_cube_complete
- init_gl_position_in_vertex_shader
- init_vertex_attributes
- pack_parameters_workaround_with_pack_buffer
- reset_base_mipmap_level_before_texstorage
- scalarize_vec_and_mat_constructor_args
- unpack_alignment_workaround_with_unpack_buffer
- unpack_overlapping_rows_separately_unpack_buffer
- use_virtualized_gl_contexts
- disabled_extension_GL_KHR_blend_equation_advanced
- disabled_extension_GL_KHR_blend_equation_advanced_coherent
- disabled_extension_GL_MESA_framebuffer_flip_y

可以看到在我的电脑上启用了 `use_virtualized_gl_contexts` 功能。

## 背景信息

`//gpu` 模块提供的核心功能是 CommandBuffer，包括 CommandBuffer 的夸进程 IPC 通信，GL 命令序列化以及 GPU 的兼容性及性能测试。此外，它还负责收集并向其他模块提供当前系统的GPU的硬件信息，扩展信息，驱动信息等。对接 skia ganesh 也在这里实现。
