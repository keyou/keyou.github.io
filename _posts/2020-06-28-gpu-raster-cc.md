---
title: Chromium Raster - CC
published: false
tags:
  - chromium
  - cc
  - graphics
  - raster
---

Chromium 的渲染流水线（CC的渲染流水线）如下：

`Paint` -> `Commit` -> `Tiling` -> `Raster` -> `Activate` -> `Draw(Submit)` ~~~> Viz

这里我们主要分析 `Tiling` 和 `Raster` 过程。

## Tiling

`Tiling` 中文意思是平铺，在 Chromium 中表示将 Layer 拆分成多个 Tiles，每个 Tile 表示一个矩形区域，该区域可以独立进行 Raster。



## Raster

`Raster` 表示栅格化，用来将绘制操作转换为可渲染的像素，比如位图Bitmap或者Texture。

该步骤的输入是 `cc::DisplayItemList`，它表示一系列的绘制操作。
