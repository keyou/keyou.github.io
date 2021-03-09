---
title: 【draft】ATW Rendering 渲染优化
tags:
  - xr
  - graphics
  - rendering
---

我们期望在每一次屏幕刷新前，系统都可以拿到最新的画面，在每一个 VSYNC 中，ATW 线程都至少要根据最新的 Pose 数据生成一副画面。

我们期望每一帧都像下图这种情况：

![atw-frame-loss](/data/atw-frame-expect.png)

而实际上可能出现下面这种情况：

![atw-frame-loss](/data/atw-frame-loss.png)

TODO：完善文档
