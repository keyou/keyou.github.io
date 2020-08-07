---
title: Chromium Native GL (//ui/gl)
published: false
categories:
  - chromium
  - graphics
tags:
  - chromium
  - graphics
---

```c++
driver_->fn.glClearFn(mask);
libgl_wrapper.so!gl::GLApiBase::glClearFn(gl::RealGLApi * this, GLbitfield mask) (/media/keyou/nvmedev/chromium64/src/ui/gl/gl_bindings_autogen_gl.cc:3066)
libgl_wrapper.so!gl::RealGLApi::glClearFn(gl::RealGLApi * this, GLbitfield mask) (/media/keyou/nvmedev/chromium64/src/ui/gl/gl_gl_api_implementation.cc:414)
libgl_wrapper.so!gl::TraceGLApi::glClearFn(gl::TraceGLApi * this, GLbitfield mask) (/media/keyou/nvmedev/chromium64/src/ui/gl/gl_bindings_autogen_gl.cc:6364)
glClear(GL_COLOR_BUFFER_BIT);
```
