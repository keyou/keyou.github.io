---
title: How viz works
published: false
categories:
  - chromium
  - electron
tags:
  - chromium
  - electron
---

```c++
Display::DrawAndSwap
  DirectRender::DrawFrame
    DirectRender::DrawRenderPassAndExecuteCopyRequests
      DirectRender::DrawRenderPass
        SkiaRender::FinishDrawingQuadList
          SkiaOutputSurfaceImpl::SubmitPaint
            if 在绘制root，则设定 deferred_framebuffer_draw_closure(简称DFDC) 为 SkiaOutputSurfaceImplOnGpu::FinishPaintCurrentFrame,
            else ScheduleGpuTask(SkiaOutputSurfaceImplOnGpu::FinishPaintRenderPass)
  SkiaRender::SwapBuffers
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

