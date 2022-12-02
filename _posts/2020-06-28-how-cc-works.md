---
title: How cc works
published: true
tags:
  - chromium
  - cc
  - graphics
  - raster
---

> Update:  
> 2022.11.18: 添加 blink 的 paint 堆栈；


官方的这篇 [How cc Works](https://chromium.googlesource.com/chromium/src/+/master/docs/how_cc_works.md) 文档写的很好，有顶层设计也有内部细节，可谓面面俱到了。只不过阅读门槛比较高，需要对 Chromium 的渲染有一定的了解才能较好的理解，因此才有此文。

强烈建议先学习 `viz` 再来学习 `cc`，这样会事半功倍，viz 的文档可以参考 [How viz works]({% post_url 2020-07-29-how-viz-works %})。

Chromium 的渲染流水线如下：

Blink ---> `Paint` -> `Commit` -> (`Tiling` ->) `Raster` -> `Activate` -> `Draw(Submit)` ---> Viz

Blink 对接 `cc` 的绘制接口进行 `Paint`，Paint 生成 cc 模块的**数据源**（`cc::Layer`），CC 将数据源进行合成，经过一系列过程最终在 `Draw` 阶段将合成的结果(`viz::CompositorFrame`)提交到 `Viz`。
也就是说，`Blink` 负责网页内容的绘制，`cc` 负责将绘制的结果进行合成并提交到 `Viz`。

`Blink` 负责浏览器网页部分的绘制，`//ui/views` 模块负责浏览器非网页部分 UI 的绘制，它们的绘制都对接 cc。如果我们扩展 `//ui/views` 的能力，就可以在 cc 之上建立一套新的 UI 框架，比如我们把这套新的 UI 框架叫做 `Flutter`。真正的 `Flutter` 当然没有这么简单，但其实 Flutter 最初是由 Chromium 的开发人员建立的（信息来自网络，未求证），所以 Flutter 渲染流水线或多或少的会吸取 Chromium 渲染的经验。最近 UC 内核团队开始投入 Flutter 的研究也算他们团队的一个新方向了。

## cc 的架构设计

cc 的设计相对简单，它不直接涉及跨进程操作（它依赖的 SharedImage 以及 Viz 负责夸进程处理），可以理解为一个简单的多线程异步流水线。运行在 Browser 进程中的 cc 负责合成浏览器非网页部分的 UI，运行在 Renderer 进程中的 cc 负责网页的合成。

在 [How cc Works](https://chromium.googlesource.com/chromium/src/+/master/docs/how_cc_works.md) 中给出的下图能很好的反应 cc 的核心逻辑：

![cc](/data/2020-08-19-17-32-43.png)

cc 的多线程体现在不同阶段运行在不同的线程中，`Paint` 运行在 Main 线程，`Commit`，`Activate`，`Submit` 运行在 Compositor 线程，而 `Raster` 运行在专门的 Raster 线程。

不同的 cc client 有自己的 paint 方式，比如 views 和 blink 都是 cc 的 client，他们有各自不同的 paint 逻辑，但是其他比如 commit，raster 等都是一样的逻辑，因此如果要学习 cc 可以从 views 入手，不必一开始就深入到 blink 中。

下面是基于这个 [demo](https://github.com/keyou/chromium_demo/blob/c/80.0.3987/demo_views/demo_views.cc) 的 cc 类图：

![cc](/data/cc.svg)

图中**蓝色**表示核心数据结构，用于存储数据，**橙色**表示核心类/接口，用于执行核心逻辑。

下面结合类图对 cc 流水线中的各阶段进行分析：

### Paint

Paint 用于产生 cc 的数据源，生成 `cc::Layer` 树。一个 `cc::Layer` 表示一个矩形区域内的 UI。它有很多子类，用于存储不同类型的 UI 数据，下面简单介绍一下：

- `cc::PictureLayer` 用于实现自绘型的 UI 组件，比如上层的各种 Button，Label 等都可以用它来实现。它允许外部通过实现 `cc::ContentLayerClient` 接口提供一个 `cc::DisplayItemList` 对象，它表示一个绘制操作的列表，记录了一系列的绘制操作，比如画线，画矩形，画圆等。通过 `cc::PaintCanvas` 接口可以方便的创建复杂的自绘 UI。`cc::PictureLayer` 还是**唯一**需要 Raster 的 `cc::Layer`，关于 Raster 见下文。它经过 cc 的流水线之后转换为一个或多个 `viz::TileDrawQuad` 存储在 `viz::CompositorFrame` 中。
- `cc::TextureLayer` 对应 viz 中的 `viz::TextureDrawQuad`，所有想要使用自己的逻辑进行 Raster 的 UI 组件都可以使用这种 Layer，比如 Flash 插件，WebGL等。
- `cc::SurfaceLayer` 对应 viz 中的 `viz::SurfaceDrawQuad`，用于嵌入其他的 CompositorFrame。Blink 中的 iframe 和视频播放器可以使用这种 Layer 实现。
- `cc::UIResourceLayer/cc::NinePatchLayer` 类似 TextureLayer，用于软件渲染。
- `cc::SolidColorLayer` 用于显示纯色的 UI 组件。
- `cc::VideoLayer` 以前用于专门显示视频，被 SurfaceLayer 取代。

`Blink` 和 `//ui/views` 等上层 UI 绘制引擎通过以上各种 `cc::Layer` 来描述 UI 并实现和 cc 的对接。由于 `cc::Layer` 本身可以保存 Child `cc::Layer`，因此给定一个 Layer 对象，它实际上表示一棵 `cc::Layer` 树，这个 Layer 树即主线程 Layer 树，因为它运行在主线程中，并且主线程有且只有一棵 `cc::Layer` 树。

下面是执行 Paint 的堆栈，这里是调用 `//ui/views` 组件的堆栈，如果是 Blink 则 aura 和 views 命名空间下的堆栈会换成 blink。

```c++
// 以下堆栈已经过时，主要的变动是 LayerTreeHostClient，这里仅用于解释原理。
libviews.so!views::View::PaintChildren(views::internal::RootView * this, const views::PaintInfo & paint_info) (/media/keyou/dev2/chromium64/src/ui/views/view.cc:1649)
libviews.so!views::View::Paint(views::internal::RootView * this, const views::PaintInfo & parent_paint_info) (/media/keyou/dev2/chromium64/src/ui/views/view.cc:1003)
libviews.so!views::View::PaintFromPaintRoot(views::internal::RootView * this, const ui::PaintContext & parent_context) (/media/keyou/dev2/chromium64/src/ui/views/view.cc:2101)
libviews.so!views::Widget::OnNativeWidgetPaint(views::Widget * this, const ui::PaintContext & context) (/media/keyou/dev2/chromium64/src/ui/views/widget/widget.cc:1232)
libviews.so!views::DesktopNativeWidgetAura::OnPaint(views::DesktopNativeWidgetAura * this, const ui::PaintContext & context) (/media/keyou/dev2/chromium64/src/ui/views/widget/desktop_aura/desktop_native_widget_aura.cc:1077)
libaura.so!aura::Window::Paint(aura::Window * this, const ui::PaintContext & context) (/media/keyou/dev2/chromium64/src/ui/aura/window.cc:916)
libaura.so!aura::Window::OnPaintLayer(aura::Window * this, const ui::PaintContext & context) (/media/keyou/dev2/chromium64/src/ui/aura/window.cc:1297)
libcompositor.so!ui::Layer::PaintContentsToDisplayList(ui::Layer * this, cc::ContentLayerClient::PaintingControlSetting painting_control) (/media/keyou/dev2/chromium64/src/ui/compositor/layer.cc:1258)
libcompositor.so!non-virtual thunk to ui::Layer::PaintContentsToDisplayList(cc::ContentLayerClient::PaintingControlSetting) (Unknown Source:0)
libcc.so!cc::PictureLayer::Update(cc::PictureLayer * this) (/media/keyou/dev2/chromium64/src/cc/layers/picture_layer.cc:140)
// ￪ 通过 LayerTreeHostClient 通知外部组件进行绘制
libcc.so!cc::LayerTreeHost::PaintContent(cc::LayerTreeHost * this, const cc::LayerList & update_layer_list) (/media/keyou/dev2/chromium64/src/cc/trees/layer_tree_host.cc:1402)
libcc.so!cc::LayerTreeHost::DoUpdateLayers(cc::LayerTreeHost * this) (/media/keyou/dev2/chromium64/src/cc/trees/layer_tree_host.cc:834)
libcc.so!cc::LayerTreeHost::UpdateLayers(cc::LayerTreeHost * this) (/media/keyou/dev2/chromium64/src/cc/trees/layer_tree_host.cc:703)
libcc.so!cc::SingleThreadProxy::DoPainting(cc::SingleThreadProxy * this) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:871)
libcc.so!cc::SingleThreadProxy::BeginMainFrame(cc::SingleThreadProxy * this, const viz::BeginFrameArgs & begin_frame_args) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:848)
```

blink 的如下：

```c++
// canvas 的绘制流程
#0  blink::HTMLCanvasElement::PaintInternal (this=0xd86688c4908, context=..., r=...) at ../../third_party/blink/renderer/core/html/canvas/html_canvas_element.cc:853
#1  0x00007fffddbd46d7 in blink::HTMLCanvasElement::Paint (this=0xd86688c4908, context=..., r=..., flatten_composited_layers=false) at ../../third_party/blink/renderer/core/html/canvas/html_canvas_element.cc:805
#2  0x00007fffde9a6881 in blink::HTMLCanvasPainter::PaintReplaced (this=0x7fff9b8dcaf0, paint_info=..., paint_offset=...) at ../../third_party/blink/renderer/core/paint/html_canvas_painter.cc:71
#3  0x00007fffde357262 in blink::LayoutHTMLCanvas::PaintReplaced (this=0x3be72c038290, paint_info=..., paint_offset=...) at ../../third_party/blink/renderer/core/layout/layout_html_canvas.cc:50
#4  0x00007fffdea9e583 in blink::ReplacedPainter::Paint (this=0x7fff9b8dcdc8, paint_info=...) at ../../third_party/blink/renderer/core/paint/replaced_painter.cc:159
#5  0x00007fffde3a476a in blink::LayoutReplaced::Paint (this=0x3be72c038290, paint_info=...) at ../../third_party/blink/renderer/core/layout/layout_replaced.cc:130
#6  0x00007fffdea3275d in blink::PaintLayerPainter::PaintFragmentWithPhase (this=0x7fff9b8dd570, phase=blink::PaintPhase::kForeground, fragment=..., context=..., clip_rect=..., painting_info=..., paint_flags=16496) at ../../third_party/blink/renderer/core/paint/paint_layer_painter.cc:707
#7  0x00007fffdea32d64 in blink::PaintLayerPainter::PaintForegroundForFragmentsWithPhase(blink::PaintPhase, WTF::Vector<blink::PaintLayerFragment, 1u, WTF::PartitionAllocator> const&, blink::GraphicsContext&, blink::PaintLayerPaintingInfo const&, unsigned int)::$_2::operator()(blink::PaintLayerFragment const&) const (this=0x7fff9b8dd078, fragment=...) at ../../third_party/blink/renderer/core/paint/paint_layer_painter.cc:766
#8  0x00007fffdea32930 in blink::ForAllFragments<blink::PaintLayerPainter::PaintForegroundForFragmentsWithPhase(blink::PaintPhase, WTF::Vector<blink::PaintLayerFragment, 1u, WTF::PartitionAllocator> const&, blink::GraphicsContext&, blink::PaintLayerPaintingInfo const&, unsigned int)::$_2>(blink::GraphicsContext&, WTF::Vector<blink::PaintLayerFragment, 1u, WTF::PartitionAllocator> const&, blink::PaintLayerPainter::PaintForegroundForFragmentsWithPhase(blink::PaintPhase, WTF::Vector<blink::PaintLayerFragment, 1u, WTF::PartitionAllocator> const&, blink::GraphicsContext&, blink::PaintLayerPaintingInfo const&, unsigned int)::$_2 const&) (context=..., fragments=WTF::Vector of length 1, capacity 1 = {...}, function=...) at ../../third_party/blink/renderer/core/paint/paint_layer_painter.cc:593
#9  0x00007fffdea31e68 in blink::PaintLayerPainter::PaintForegroundForFragmentsWithPhase (this=0x7fff9b8dd570, phase=blink::PaintPhase::kForeground, layer_fragments=WTF::Vector of length 1, capacity 1 = {...}, context=..., local_painting_info=..., paint_flags=16496) at ../../third_party/blink/renderer/core/paint/paint_layer_painter.cc:763
#10 0x00007fffdea31f57 in blink::PaintLayerPainter::PaintForegroundForFragments (this=0x7fff9b8dd570, layer_fragments=WTF::Vector of length 1, capacity 1 = {...}, context=..., local_painting_info=..., paint_flags=16496) at ../../third_party/blink/renderer/core/paint/paint_layer_painter.cc:745
#11 0x00007fffdea30c60 in blink::PaintLayerPainter::PaintLayerContents (this=0x7fff9b8dd570, context=..., painting_info_arg=..., paint_flags_arg=16496) at ../../third_party/blink/renderer/core/paint/paint_layer_painter.cc:506
#12 0x00007fffdea2fd67 in blink::PaintLayerPainter::Paint (this=0x7fff9b8dd570, context=..., painting_info=..., paint_flags=16496) at ../../third_party/blink/renderer/core/paint/paint_layer_painter.cc:106
#13 0x00007fffdea31c1d in blink::PaintLayerPainter::PaintChildren (this=0x7fff9b8dda40, children_to_visit=blink::kNormalFlowAndPositiveZOrderChildren, context=..., painting_info=..., paint_flags=16496) at ../../third_party/blink/renderer/core/paint/paint_layer_painter.cc:621
#14 0x00007fffdea30cde in blink::PaintLayerPainter::PaintLayerContents (this=0x7fff9b8dda40, context=..., painting_info_arg=..., paint_flags_arg=16880) at ../../third_party/blink/renderer/core/paint/paint_layer_painter.cc:517
#15 0x00007fffdea2fd67 in blink::PaintLayerPainter::Paint (this=0x7fff9b8dda40, context=..., painting_info=..., paint_flags=16880) at ../../third_party/blink/renderer/core/paint/paint_layer_painter.cc:106
#16 0x00007fffdea31c1d in blink::PaintLayerPainter::PaintChildren (this=0x7fff9b8dde28, children_to_visit=blink::kNormalFlowAndPositiveZOrderChildren, context=..., painting_info=..., paint_flags=416) at ../../third_party/blink/renderer/core/paint/paint_layer_painter.cc:621
#17 0x00007fffdea30cde in blink::PaintLayerPainter::PaintLayerContents (this=0x7fff9b8dde28, context=..., painting_info_arg=..., paint_flags_arg=416) at ../../third_party/blink/renderer/core/paint/paint_layer_painter.cc:517
#18 0x00007fffde9714fc in blink::CompositedLayerMapping::DoPaintTask (this=0x3f87ef625990, paint_info=..., graphics_layer=..., paint_layer_flags=416, context=..., clip=...) at ../../third_party/blink/renderer/core/paint/compositing/composited_layer_mapping.cc:1619
#19 0x00007fffde97291f in blink::CompositedLayerMapping::PaintContents (this=0x3f87ef625990, graphics_layer=0x3f87ef72c910, context=..., graphics_layer_painting_phase=26, interest_rect=...) at ../../third_party/blink/renderer/core/paint/compositing/composited_layer_mapping.cc:1922
#20 0x00007fffd907ebb3 in blink::GraphicsLayer::Paint (this=0x3f87ef72c910, pre_composited_layers=WTF::Vector of length 1, capacity 4 = {...}, benchmark_mode=blink::PaintBenchmarkMode::kNormal, interest_rect=0x0) at ../../third_party/blink/renderer/platform/graphics/graphics_layer.cc:371
#21 0x00007fffd90809c1 in blink::GraphicsLayer::PaintRecursively(blink::GraphicsContext&, WTF::Vector<blink::PreCompositedLayerInfo, 0u, WTF::PartitionAllocator>&, blink::PaintBenchmarkMode)::$_2::operator()(blink::GraphicsLayer&) const (this=0x7fff9b8de760, layer=...) at ../../third_party/blink/renderer/platform/graphics/graphics_layer.cc:294
#22 0x00007fffd907e4f1 in blink::ForAllGraphicsLayers<blink::GraphicsLayer, blink::GraphicsLayer::PaintRecursively(blink::GraphicsContext&, WTF::Vector<blink::PreCompositedLayerInfo, 0u, WTF::PartitionAllocator>&, blink::PaintBenchmarkMode)::$_2, blink::GraphicsLayer::PaintRecursively(blink::GraphicsContext&, WTF::Vector<blink::PreCompositedLayerInfo, 0u, WTF::PartitionAllocator>&, blink::PaintBenchmarkMode)::$_3>(blink::GraphicsLayer&, blink::GraphicsLayer::PaintRecursively(blink::GraphicsContext&, WTF::Vector<blink::PreCompositedLayerInfo, 0u, WTF::PartitionAllocator>&, blink::PaintBenchmarkMode)::$_2 const&, blink::GraphicsLayer::PaintRecursively(blink::GraphicsContext&, WTF::Vector<blink::PreCompositedLayerInfo, 0u, WTF::PartitionAllocator>&, blink::PaintBenchmarkMode)::$_3 const&) (layer=..., graphics_layer_function=..., contents_layer_function=...) at ../../third_party/blink/renderer/platform/graphics/graphics_layer.h:344
#23 0x00007fffd907e5be in blink::ForAllGraphicsLayers<blink::GraphicsLayer, blink::GraphicsLayer::PaintRecursively(blink::GraphicsContext&, WTF::Vector<blink::PreCompositedLayerInfo, 0u, WTF::PartitionAllocator>&, blink::PaintBenchmarkMode)::$_2, blink::GraphicsLayer::PaintRecursively(blink::GraphicsContext&, WTF::Vector<blink::PreCompositedLayerInfo, 0u, WTF::PartitionAllocator>&, blink::PaintBenchmarkMode)::$_3>(blink::GraphicsLayer&, blink::GraphicsLayer::PaintRecursively(blink::GraphicsContext&, WTF::Vector<blink::PreCompositedLayerInfo, 0u, WTF::PartitionAllocator>&, blink::PaintBenchmarkMode)::$_2 const&, blink::GraphicsLayer::PaintRecursively(blink::GraphicsContext&, WTF::Vector<blink::PreCompositedLayerInfo, 0u, WTF::PartitionAllocator>&, blink::PaintBenchmarkMode)::$_3 const&) (layer=..., graphics_layer_function=..., contents_layer_function=...) at ../../third_party/blink/renderer/platform/graphics/graphics_layer.h:351
#24 0x00007fffd907e328 in blink::GraphicsLayer::PaintRecursively (this=0x3f87ef72ca90, context=..., pre_composited_layers=WTF::Vector of length 1, capacity 4 = {...}, benchmark_mode=blink::PaintBenchmarkMode::kNormal) at ../../third_party/blink/renderer/platform/graphics/graphics_layer.cc:287
#25 0x00007fffdda32f32 in blink::LocalFrameView::PaintTree (this=0x361f26e9e768, benchmark_mode=blink::PaintBenchmarkMode::kNormal) at ../../third_party/blink/renderer/core/frame/local_frame_view.cc:3045
#26 0x00007fffdda3112a in blink::LocalFrameView::RunPaintLifecyclePhase (this=0x361f26e9e768, benchmark_mode=blink::PaintBenchmarkMode::kNormal) at ../../third_party/blink/renderer/core/frame/local_frame_view.cc:2778
#27 0x00007fffdda300dd in blink::LocalFrameView::UpdateLifecyclePhasesInternal (this=0x361f26e9e768, target_state=blink::DocumentLifecycle::kPaintClean) at ../../third_party/blink/renderer/core/frame/local_frame_view.cc:2578
#28 0x00007fffdda2ebc2 in blink::LocalFrameView::UpdateLifecyclePhases (this=0x361f26e9e768, target_state=blink::DocumentLifecycle::kPaintClean, reason=blink::DocumentUpdateReason::kBeginMainFrame) at ../../third_party/blink/renderer/core/frame/local_frame_view.cc:2464
#29 0x00007fffdda2e4b0 in blink::LocalFrameView::UpdateAllLifecyclePhases (this=0x361f26e9e768, reason=blink::DocumentUpdateReason::kBeginMainFrame) at ../../third_party/blink/renderer/core/frame/local_frame_view.cc:2207
#30 0x00007fffde8e1ce7 in blink::PageAnimator::UpdateAllLifecyclePhases (this=0x301f74fc1ad8, root_frame=..., reason=blink::DocumentUpdateReason::kBeginMainFrame) at ../../third_party/blink/renderer/core/page/page_animator.cc:131
#31 0x00007fffde8e7adc in blink::PageWidgetDelegate::UpdateLifecycle (page=..., root=..., requested_update=blink::WebLifecycleUpdate::kAll, reason=blink::DocumentUpdateReason::kBeginMainFrame) at ../../third_party/blink/renderer/core/page/page_widget_delegate.cc:72
#32 0x00007fffdf82b31a in blink::WebViewImpl::UpdateLifecycle (this=0x3f87ef6f4010, requested_update=blink::WebLifecycleUpdate::kAll, reason=blink::DocumentUpdateReason::kBeginMainFrame) at ../../third_party/blink/renderer/core/exported/web_view_impl.cc:1342
#33 0x00007fffddb824e0 in blink::WebViewFrameWidget::UpdateLifecycle (this=0x361f26e88578, requested_update=blink::WebLifecycleUpdate::kAll, reason=blink::DocumentUpdateReason::kBeginMainFrame) at ../../third_party/blink/renderer/core/frame/web_view_frame_widget.cc:97
#34 0x00007fffd9360776 in blink::WidgetBase::UpdateVisualState (this=0x3bb6cec174a0) at ../../third_party/blink/renderer/platform/widget/widget_base.cc:761
#35 0x00007fffd92e7375 in blink::LayerTreeView::UpdateLayerTreeHost (this=0x3bb6cf1e02e0) at ../../third_party/blink/renderer/platform/widget/compositing/layer_tree_view.cc:198
// ￪ 通过 LayerTreeHostClient 通知外部组件进行绘制
#36  0x00007fffe77df186 in cc::LayerTreeHost::RequestMainFrameUpdate (this=0x16aa67069220, report_cc_metrics=true) at ../../cc/trees/layer_tree_host.cc:305                                                                                    
#37  0x00007fffe78d65a5 in cc::ProxyMain::BeginMainFrame (this=0x16aa66d16e30, begin_main_frame_state=std::unique_ptr<cc::BeginMainFrameAndCommitState> containing = {...}) at ../../cc/trees/proxy_main.cc:258

```

### Commit

Commit 阶段的核心作用是将保存在 `cc::Layer` 中的数据提交（`PushPropertiesTo`）到 `cc::LayerImpl` 中。`cc::LayerImpl` 和 `cc::Layer` 一一对应，只不过运行在 Compositor 线程中（也称为 Impl 线程）。在 Commit 完成之后会根据需要创建 Tiles 任务，这些任务被 Post 到 Raster 线程中执行。

下面是 Commit 部分的堆栈：

```c++
case SchedulerStateMachine::Action::COMMIT:
执行 commit: 将 LayerTreeHost 的数据commit到 LayerTreeHostImpl，注意commit操作是在impl线程中执行的（如果是单线程则在和main同一个线程），commit的核心代码在LayerTreeHost中
libcc.so!cc::PictureLayer::PushPropertiesTo(cc::PictureLayer * this, cc::PictureLayerImpl * base_layer) (/media/keyou/dev2/chromium64/src/cc/layers/picture_layer.cc:52)
libcc.so!cc::PushLayerPropertiesInternal<std::__Cr::__wrap_iter<cc::Layer**> >(std::__Cr::__wrap_iter<cc::Layer**> source_layers_begin, std::__Cr::__wrap_iter<cc::Layer**> source_layers_end, cc::LayerTreeHost * host_tree, cc::LayerTreeImpl * target_impl_tree) (/media/keyou/dev2/chromium64/src/cc/trees/tree_synchronizer.cc:172)
libcc.so!cc::TreeSynchronizer::PushLayerProperties(cc::LayerTreeHost * host_tree, cc::LayerTreeImpl * impl_tree) (/media/keyou/dev2/chromium64/src/cc/trees/tree_synchronizer.cc:206)
libcc.so!cc::LayerTreeHost::FinishCommitOnImplThread(cc::LayerTreeHost * this, cc::LayerTreeHostImpl * host_impl) (/media/keyou/dev2/chromium64/src/cc/trees/layer_tree_host.cc:343)
libcc.so!cc::SingleThreadProxy::DoCommit(cc::SingleThreadProxy * this) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:196)
libcc.so!cc::SingleThreadProxy::ScheduledActionCommit(cc::SingleThreadProxy * this) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:912)
libcc.so!cc::Scheduler::ProcessScheduledActions(cc::Scheduler * this) (/media/keyou/dev2/chromium64/src/cc/scheduler/scheduler.cc:795)
libcc.so!cc::Scheduler::NotifyReadyToCommit(cc::Scheduler * this, std::__Cr::unique_ptr<cc::BeginMainFrameMetrics, std::__Cr::default_delete<cc::BeginMainFrameMetrics> > details) (/media/keyou/dev2/chromium64/src/cc/scheduler/scheduler.cc:179)
libcc.so!cc::SingleThreadProxy::DoPainting(cc::SingleThreadProxy * this) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:877)
libcc.so!cc::SingleThreadProxy::BeginMainFrame(cc::SingleThreadProxy * this, const viz::BeginFrameArgs & begin_frame_args) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:848)
+
Post Tiles任务（raster任务）到单独的Raster线程，注意这同时会在任务队列的最后放入结束任务，专门用于raster结束的通知，并且结束通知是运行在主线中
libcc.so!cc::SingleThreadTaskGraphRunner::ScheduleTasks(cc::TestTaskGraphRunner * this, cc::NamespaceToken token, cc::TaskGraph * graph) (/media/keyou/dev2/chromium64/src/cc/raster/single_thread_task_graph_runner.cc:68)
libcc.so!cc::TileTaskManagerImpl::ScheduleTasks(cc::TileTaskManagerImpl * this, cc::TaskGraph * graph) (/media/keyou/dev2/chromium64/src/cc/tiles/tile_task_manager.cc:32)
libcc.so!cc::TileManager::ScheduleTasks(cc::TileManager * this, cc::TileManager::PrioritizedWorkToSchedule work_to_schedule) (/media/keyou/dev2/chromium64/src/cc/tiles/tile_manager.cc:1108)
libcc.so!cc::TileManager::PrepareTiles(cc::TileManager * this, const cc::GlobalStateThatImpactsTilePriority & state) (/media/keyou/dev2/chromium64/src/cc/tiles/tile_manager.cc:548)
libcc.so!cc::LayerTreeHostImpl::PrepareTiles(cc::LayerTreeHostImpl * this) (/media/keyou/dev2/chromium64/src/cc/trees/layer_tree_host_impl.cc:762)
libcc.so!cc::LayerTreeHostImpl::NotifyPendingTreeFullyPainted(cc::LayerTreeHostImpl * this) (/media/keyou/dev2/chromium64/src/cc/trees/layer_tree_host_impl.cc:651)
libcc.so!cc::LayerTreeHostImpl::UpdateSyncTreeAfterCommitOrImplSideInvalidation(cc::LayerTreeHostImpl * this) (/media/keyou/dev2/chromium64/src/cc/trees/layer_tree_host_impl.cc:536)
libcc.so!cc::LayerTreeHostImpl::CommitComplete(cc::LayerTreeHostImpl * this) (/media/keyou/dev2/chromium64/src/cc/trees/layer_tree_host_impl.cc:454)
libcc.so!cc::SingleThreadProxy::DoCommit(cc::SingleThreadProxy * this) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:203)
libcc.so!cc::SingleThreadProxy::ScheduledActionCommit(cc::SingleThreadProxy * this) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:912)
libcc.so!cc::Scheduler::ProcessScheduledActions(cc::Scheduler * this) (/media/keyou/dev2/chromium64/src/cc/scheduler/scheduler.cc:821)
libcc.so!cc::Scheduler::NotifyReadyToCommit(cc::Scheduler * this, std::__Cr::unique_ptr<cc::BeginMainFrameMetrics, std::__Cr::default_delete<cc::BeginMainFrameMetrics> > details) (/media/keyou/dev2/chromium64/src/cc/scheduler/scheduler.cc:179)
libcc.so!cc::SingleThreadProxy::DoPainting(cc::SingleThreadProxy * this) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:877)
libcc.so!cc::SingleThreadProxy::BeginMainFrame(cc::SingleThreadProxy * this, const viz::BeginFrameArgs & begin_frame_args) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:848)
```

### Tiling+Raster

在 Commit 阶段创建的 Tiles 任务（`cc::RasterTaskImpl`）在该阶段被执行。Tiling 阶段最重要的作用是将一个 `cc::PictureLayerImpl` 根据不同的 scale 级别，不同的大小拆分为多个 `cc::TileTask` 任务。Raster 阶段会执行每一个 TileTask,将 DisplayItemList 中的绘制操作 Playback 到 viz 的资源中。
由于 Raster 比较耗时，属于渲染的性能敏感路径，因此Chromium在这里实现了多种策略以适应不同的情况。这些策略主要在两方面进行优化，一方面是 Raster 结果（也就是资源）存储的位置，一方面是 Raster 中 Playback 的方式。这些方案被封装在了 `cc::RasterBufferProvider` 的子类中，下面一一进行介绍：

- `cc::GpuRasterBufferProvider` 使用 GPU 进行 Raster，Raster 的结果直接存储在 SharedImage 中（可以理解为 GLTexture，详细信息见 [GPU SharedImage]({% post_url 2020-06-22-chromium-gpu-image-share %})）。
- `cc::OneCopyRasterBufferProvider` 使用 Skia 进行 Raster，结果先保存到 GpuMemoryBuffer 中，然后再将 GpuMemoryBuffer 中的数据通过 CopySubTexture 拷贝到资源的 SharedImage 中。GpuMemeoryBuffer 在不同平台有不同的实现，也并不是所有的平台都支持，在 Linux 平台上底层实现为 Native Pixmap（来自X11中的概念）,在 Windows 平台上底层实现为 DXGI，在 Android 上底层实现为 AndroidHardwareBuffer，在 Mac 上底层实现为 IOSurface。
- `cc::ZeroCopyRasterBufferProvider` 使用 Skia 进行 Raster，结果保存到 GpuMemoryBuffer 中，然后使用 GpuMemoryBuffer 直接创建 SharedImage。
- `cc::BitmapRasterBufferProvider` 使用 Skia 进行 Raster，结果保存到共享内存中。

Raster 最终会产生一个资源，该资源被（间接）记录在了 `cc::PictureLayerImpl` 中，他们会在最终的 Draw 阶段被放入 CompositorFrame 中。

除了 `cc::PictureLayer` 的 Raster 之外，在该阶段还会进行图片的解码，关于这部分的详细信息以后如果有时间再补充。

TODO： 补充 GpuMemoryBuffer 和 图片解码相关内容。

下面是抓取到的 `cc::RasterTaskImpl` 的执行流程：

```c++
执行tile任务，运行在单独的Raster线程（在SingleThreadTaskGraphRunner中执行Raster操作）这会将DisplayItemList中保存的绘制操作执行真实的绘制,绘制到内存中,注意这里采用的是OnCopyRasterBufferProvider。
libcc_paint.so!cc::PaintOpBuffer::Playback(const cc::PaintOpBuffer * this, SkCanvas * canvas, const cc::PlaybackParams & params, const std::__Cr::vector<unsigned long, std::__Cr::allocator<unsigned long> > * offsets) (/media/keyou/dev2/chromium64/src/cc/paint/paint_op_buffer.cc:2431)
libcc_paint.so!cc::DisplayItemList::Raster(const cc::DisplayItemList * this, SkCanvas * canvas, cc::(anonymous namespace)::DispatchingImageProvider * image_provider) (/media/keyou/dev2/chromium64/src/cc/paint/display_item_list.cc:81)
libcc.so!cc::RasterSource::PlaybackToCanvas(const cc::RasterSource * this, SkCanvas * raster_canvas, cc::(anonymous namespace)::DispatchingImageProvider * image_provider) (/media/keyou/dev2/chromium64/src/cc/raster/raster_source.cc:171)
libcc.so!cc::RasterSource::PlaybackToCanvas(const cc::RasterSource * this, SkCanvas * raster_canvas, const gfx::Size & content_size, const gfx::Rect & canvas_bitmap_rect, const gfx::Rect & canvas_playback_rect, const gfx::AxisTransform2d & raster_transform, const cc::RasterSource::PlaybackSettings & settings) (/media/keyou/dev2/chromium64/src/cc/raster/raster_source.cc:161)
libcc.so!cc::RasterBufferProvider::PlaybackToMemory(void * memory, viz::ResourceFormat format, const gfx::Size & size, size_t stride, const cc::RasterSource * raster_source, const gfx::Rect & canvas_bitmap_rect, const gfx::Rect & canvas_playback_rect, const gfx::AxisTransform2d & transform, const gfx::ColorSpace & target_color_space, bool gpu_compositing, const cc::RasterSource::PlaybackSettings & playback_settings) (/media/keyou/dev2/chromium64/src/cc/raster/raster_buffer_provider.cc:107)
libcc.so!cc::OneCopyRasterBufferProvider::PlaybackToStagingBuffer(cc::OneCopyRasterBufferProvider * this, cc::StagingBuffer * staging_buffer, const cc::RasterSource * raster_source, const gfx::Rect & raster_full_rect, const gfx::Rect & raster_dirty_rect, const gfx::AxisTransform2d & transform, viz::ResourceFormat format, const gfx::ColorSpace & dst_color_space, const cc::RasterSource::PlaybackSettings & playback_settings, uint64_t previous_content_id, uint64_t new_content_id) (/media/keyou/dev2/chromium64/src/cc/raster/one_copy_raster_buffer_provider.cc:347)
libcc.so!cc::OneCopyRasterBufferProvider::PlaybackAndCopyOnWorkerThread(cc::OneCopyRasterBufferProvider * this, gpu::Mailbox * mailbox, GLenum mailbox_texture_target, bool mailbox_texture_is_overlay_candidate, const gpu::SyncToken & sync_token, const cc::RasterSource * raster_source, const gfx::Rect & raster_full_rect, const gfx::Rect & raster_dirty_rect, const gfx::AxisTransform2d & transform, const gfx::Size & resource_size, viz::ResourceFormat resource_format, const gfx::ColorSpace & color_space, const cc::RasterSource::PlaybackSettings & playback_settings, uint64_t previous_content_id, uint64_t new_content_id) (/media/keyou/dev2/chromium64/src/cc/raster/one_copy_raster_buffer_provider.cc:270)
libcc.so!cc::OneCopyRasterBufferProvider::RasterBufferImpl::Playback(cc::OneCopyRasterBufferProvider::RasterBufferImpl * this, const cc::RasterSource * raster_source, const gfx::Rect & raster_full_rect, const gfx::Rect & raster_dirty_rect, uint64_t new_content_id, const gfx::AxisTransform2d & transform, const cc::RasterSource::PlaybackSettings & playback_settings, const GURL & url) (/media/keyou/dev2/chromium64/src/cc/raster/one_copy_raster_buffer_provider.cc:118)
libcc.so!cc::(anonymous namespace)::RasterTaskImpl::RunOnWorkerThread(cc::(anonymous namespace)::RasterTaskImpl * this) (/media/keyou/dev2/chromium64/src/cc/tiles/tile_manager.cc:125)
libcc.so!cc::SingleThreadTaskGraphRunner::RunTaskWithLockAcquired(cc::TestTaskGraphRunner * this) (/media/keyou/dev2/chromium64/src/cc/raster/single_thread_task_graph_runner.cc:154)
libcc.so!cc::SingleThreadTaskGraphRunner::Run(cc::TestTaskGraphRunner * this) (/media/keyou/dev2/chromium64/src/cc/raster/single_thread_task_graph_runner.cc:117)
libbase.so!base::DelegateSimpleThread::Run(base::DelegateSimpleThread * this) (/media/keyou/dev2/chromium64/src/base/threading/simple_thread.cc:98)
libbase.so!base::SimpleThread::ThreadMain(base::DelegateSimpleThread * this) (/media/keyou/dev2/chromium64/src/base/threading/simple_thread.cc:75)
libbase.so!base::(anonymous namespace)::ThreadFunc(void * params) (/media/keyou/dev2/chromium64/src/base/threading/platform_thread_posix.cc:81)
libpthread.so.0!start_thread (Unknown Source:0)
libc.so.6!clone (Unknown Source:0)
```

### Activate

在 Impl 端有三个 `cc::LayerImpl` 树，分别是 Pending,Active,Recycle 树。Commit 阶段提交的目标其实就是 Pending 树，Raster 的结果也被存储在了 Pending 树中。

在 Activate 阶段，Pending 树中的所有 `cc::LayerImpl` 会被复制到 Active 树中，为了避免频繁的创建 `cc::LayerImpl` 对象，此时 Pending 树并不会被销毁，而是退化为 Recycle 树。

和主线程 `cc::Layer` 树不同，`cc::LayerImpl` 树并不是自己维护树形结构的，而是由 `cc::LayerTreeImpl` 对象来维护 `cc::LayerImpl` 树的。三个 Impl 树分别对应三个 `cc::LayerTreeImpl` 对象。

下面是 Activate 阶段的执行堆栈：

```c++
case SchedulerStateMachine::Action::ACTIVATE_SYNC_TREE:
主要用于将pending_tree_ Sync 到 active_tree_ 上（在单线程中不使用pending_tree_;这个代码在LayerTreeHostImpl中；）
libcc.so!cc::LayerTreeHostImpl::ActivateSyncTree(cc::LayerTreeHostImpl * this) (/media/keyou/dev2/chromium64/src/cc/trees/layer_tree_host_impl.cc:2979)
libcc.so!cc::SingleThreadProxy::ScheduledActionActivateSyncTree(cc::SingleThreadProxy * this) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:917)
libcc.so!cc::Scheduler::ProcessScheduledActions(cc::Scheduler * this) (/media/keyou/dev2/chromium64/src/cc/scheduler/scheduler.cc:795)
libcc.so!cc::Scheduler::NotifyReadyToCommit(cc::Scheduler * this, std::__Cr::unique_ptr<cc::BeginMainFrameMetrics, std::__Cr::default_delete<cc::BeginMainFrameMetrics> > details) (/media/keyou/dev2/chromium64/src/cc/scheduler/scheduler.cc:179)
libcc.so!cc::SingleThreadProxy::DoPainting(cc::SingleThreadProxy * this) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:877)
libcc.so!cc::SingleThreadProxy::BeginMainFrame(cc::SingleThreadProxy * this, const viz::BeginFrameArgs & begin_frame_args) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:848)
```

### Draw

Draw 阶段并不执行真正的绘制，而是遍历 Active 树中的 `cc::LayerImpl` 对象，并调用它的 `cc::LayerImpl::AppendQuads` 方法创建合适的 `viz::DrawQuad` 放入 CompositorFrame 的 RenderPass 中。`cc::LayerImpl` 中的资源会被创建为 `viz::TransferabelResource` 存入 CompositorFrame 的资源列表中。至此一个 `viz::CompositorFrame` 对象创建完毕，最后通过 `cc::LayerTreeFrameSink` 接口将该 CompositorFrame 发送到给 viz 进程（GPU进程）进行渲染。

下面是 Draw 阶段的运行堆栈：

```c++
Deadline时间到，触发绘制(Draw/Submit)操作
Run Scheduler::OnBeginImplFrameDeadline
case SchedulerStateMachine::Action::DRAW_IF_POSSIBLE:
libservice.so!viz::DirectLayerTreeFrameSink::SubmitCompositorFrame(viz::DirectLayerTreeFrameSink * this, viz::CompositorFrame frame, bool hit_test_data_changed, bool show_hit_test_borders) (/media/keyou/dev2/chromium64/src/components/viz/service/frame_sinks/direct_layer_tree_frame_sink.cc:184)
libcc.so!cc::LayerTreeHostImpl::DrawLayers(cc::LayerTreeHostImpl * this, cc::LayerTreeHostImpl::FrameData * frame) (/media/keyou/dev2/chromium64/src/cc/trees/layer_tree_host_impl.cc:2294)
libcc.so!cc::SingleThreadProxy::DoComposite(cc::SingleThreadProxy * this, cc::LayerTreeHostImpl::FrameData * frame) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:691)
libcc.so!cc::SingleThreadProxy::ScheduledActionDrawIfPossible(cc::SingleThreadProxy * this) (/media/keyou/dev2/chromium64/src/cc/trees/single_thread_proxy.cc:902)
libcc.so!cc::Scheduler::DrawIfPossible(cc::Scheduler * this) (/media/keyou/dev2/chromium64/src/cc/scheduler/scheduler.cc:732)
libcc.so!cc::Scheduler::ProcessScheduledActions(cc::Scheduler * this) (/media/keyou/dev2/chromium64/src/cc/scheduler/scheduler.cc:838)
libcc.so!cc::Scheduler::OnBeginImplFrameDeadline(cc::Scheduler * this) (/media/keyou/dev2/chromium64/src/cc/scheduler/scheduler.cc:720)
```

## Scheduler 调度器

上面已经介绍了 cc 渲染流水线的各个过程，这些过程是由 `cc::Scheduler` 进行调度的。它控制 cc 在合适的时机执行合适的动作，内部维护了一个渲染的状态机，
其核心调度逻辑在 `cc::Scheduler::ProcessScheduledActions()` 中，代码如下：

```C++
// 2020年1月(v80.0.3987.158),有删减
void Scheduler::ProcessScheduledActions() {
  SchedulerStateMachine::Action action;
  do {
    action = state_machine_.NextAction();
    switch (action) {
      case SchedulerStateMachine::Action::NONE:
        break;
      case SchedulerStateMachine::Action::SEND_BEGIN_MAIN_FRAME:
        // 绘制新帧，这会触发cc embedder的绘制(Paint)，比如views::View::Paint()或者blink::GraphicsLayer::Paint()
        client_->ScheduledActionSendBeginMainFrame(begin_main_frame_args_);
        break;
      case SchedulerStateMachine::Action::COMMIT: {
        // 执行提交，也就是将cc::Layer的内容复制到cc::LayerImpl中
        client_->ScheduledActionCommit();
        break;
      }
      case SchedulerStateMachine::Action::ACTIVATE_SYNC_TREE:
        // 执行同步，也就是将pending_tree的内容同步到active_tree
        client_->ScheduledActionActivateSyncTree();
        break;
      case SchedulerStateMachine::Action::DRAW_IF_POSSIBLE:
        // 执行submit，创建CompositorFrame并提交到DisplayCompsitor
        DrawIfPossible();
        break;
      case SchedulerStateMachine::Action::BEGIN_LAYER_TREE_FRAME_SINK_CREATION:
        // 初始化，请求创建LayerTreeFrameSink
        client_->ScheduledActionBeginLayerTreeFrameSinkCreation();
        break;
      case SchedulerStateMachine::Action::PREPARE_TILES:
        // 执行Raster，并且准备Tiles
        client_->ScheduledActionPrepareTiles();
        break;
      }
    }
  } while (action != SchedulerStateMachine::Action::NONE);

  // 用于设定一个Deadline，在该Deadline之前都允许触发SEND_BEGIN_MAIN_FRAME事件，在Deadline触发后就不再允许触发该事件。
  // 它会Post一个延迟的OnBeginImplFrameDeadline事件，如果该事件执行了表示已经到了Deadline（状态机进入INSIDE_DEADLINE状态），
  // 它会设置一些状态禁止触发BeginMainFrame，然后进行下一次调度，这些调度都是需要在Deadline状态才能执行的操作，
  // 可能调度到DRAW_IF_POSSIBLE，这会触发client提交CF到viz，也可能调度到其他状态比如PREPARE_TILES等，当然也可能调度到NONE状态，
  // 在进入Deadline之后的所有操作都完成之后，调度器会进入空闲状态IDLE，此时就算退出Deadline了，后续就允许触发SEND_BEGIN_MAIN_FRAME事件了。
  ScheduleBeginImplFrameDeadline();

  // 用于检测是否需要触发SEND_BEGIN_MAIN_FRAME事件，如果需要它会Post HandlePendingBeginFrame事件，
  // 该事件执行后会使状态机进入INSIDE_BEGIN_FRAME状态，随后调度器会在合适的时机触发SEND_BEGIN_MAIN_FRAME事件，
  // 这会调用client进行绘制，在client绘制完成之后会请求调度器执行提交COMMIT以及激活ACTIVATE_SYNC_TREE。
  PostPendingBeginFrameTask();

  // 用于检测当前是否处于IDLE状态，如果是则将自己从 BFS 移除，否则则将自己 加入BFS。如果调度器将自己从BFS上移除，则会进入”休眠状态“，
  // 此时不会调度器不会占用CPU，如果后续外部需要更新显示的画面，可以通过cc::Scheduler::SetNeedsBeginFrame()
  // 来激活调度器（注意cc模块外可以通过cc::LayerTreeHost::SetNeedsCommit()来间接调用它，如果在cc::Layer中，
  // 则可以直接调用它的SetNeedsCommit成员函数，如果cc::Layer没有任何变化，只调用SetNeddsCommit() 不会激活调度器，必须有实际的变更或者标记区域demaged）。
  StartOrStopBeginFrames();
}
```

TODO： 补充 cc 在调度上的优化。

## 总结

这里只是简单介绍了 `cc` 的主要逻辑，还有很多周边模块没有提及，比如动画，UI 嵌套/独立渲染，图片解码等，所以再次推荐官方的 [How cc Works](https://chromium.googlesource.com/chromium/src/+/master/docs/how_cc_works.md) 文档。

-------

参考文档：

- [How cc Works](https://chromium.googlesource.com/chromium/src/+/master/docs/how_cc_works.md)