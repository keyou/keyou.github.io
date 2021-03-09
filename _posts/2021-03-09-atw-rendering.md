---
title: [Draft] `ATW Rendering 渲染优化
tags:
  - xr
  - graphics
  - rendering
---

// TODO：完善文档

我们期望在每一次屏幕刷新前，系统都可以拿到最新的画面，在每一个 VSYNC 中，ATW 线程都至少要根据最新的Pose数据生成一副画面。

![atw-frame-loss](/data/atw-frame-expect.png)
![atw-frame-loss](/data/atw-frame-loss.png)

## 总结


-----
参考文档：

