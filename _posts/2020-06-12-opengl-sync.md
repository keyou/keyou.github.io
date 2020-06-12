---
title: OpengGL Synchronization
published: false
categories: 
  - opengl
tags:
  - opengl
---

> 这里记录关于 OpenGL 同步的方方面面。

## 为什么需要同步

根本原因是 OpenGL 绘制API的执行是异步的，如果是同步的就没有这个话题了，设计为异步API可以提高性能。这里的异步指的是一个GL API调用结束并不表示它已经被GPU执行了。应用对GL API的调用（称为一个GL命令）会先被GPU驱动程序缓存在内存中，然后在某一个时机驱动程序再把GL命令发送到GPU硬件中，GPU硬件中有个命令队列，GPU会从这个队列中取出命令进行执行。所以一个GL命令会经过2次缓存，一次在GPU驱动程序中，一次在GPU硬件中。正常情况下处于缓存中的命令什么时候被实际执行应用是不知道的，但是如果应用需要使用这些GL命令执行的结果了，比如把渲染的结果作为位图读到内存，这个时候就必须要保证所有的GL绘制命令都被执行了才可以，否则读到的位图就是不完整的。这里就需要同步机制了。但在这个举例中

> 详细信息参考官方解读： [Synchronization - OpenGL Wiki](https://www.khronos.org/opengl/wiki/Synchronization)

