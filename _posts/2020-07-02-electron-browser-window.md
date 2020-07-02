---
title: 在 Electron 中嵌入自定义图层
categories:
  - chromium
  - electron
tags:
  - chromium
  - electron
---

在使用 Electron 开发客户端程序的时候，会遇到有些内容的渲染比较消耗性能，或者有时需要在 Electron 中嵌入第三方界面的情况，这类需求可以通过在 Electron 中嵌入新的图层来实现。

## Electron 包含的图层

下面这段 js 代码创建一个窗口并加载指定网页：

```js
const {app, BrowserWindow} = require('electron')
app.whenReady().then(() => {
  // 创建并显示窗口
  const mainWindow = new BrowserWindow()
  // 加载网页
  mainWindow.loadURL("http://example.com")
})
```

下面是 `new BrowserWindow()` 创建并显示窗口的主要流程：

```c++
// 说明：
// js 开头表示是 js 调用,其它是 C++ 调用
// !!! 表示对接 Chromium 的关键逻辑

js require("electron") // 获取 BrowserWindow 类
js new BrowserWindow()
BrowserWindow::New()
  BrowserWindow::BrowserWindow()
    TopLevelWindow::TopLevelWindow()
      NativeWindow::Create()
        NativeWindowViews::NativeWindowViews()
          NativeWindow::NativeWindow()
            views::Widget:Widget() // !!!
          electron::RootView::RootView() : views::View // 创建 NativeWindowViews 的 rootview
          views::Widget::Init() // !!!
            NativeWindowViews::GetContentsView/CreateClientView()
            views::View::AddChildView() // !!! 将 NativeWindowViews 的 rootview 添加到 widget
      NativeWindow::InitFromOptions()
    elctron::api::WebContents::Create()
      elctron::api::WebContents::WebContents() // 包装 InspectableWebContents
        content::WebContents::Create() // !!!
        CommonWebContentsDelegate::InitWithWebContents()
          InspectableWebContents::Create()
            InspectableWebContentsImpl::InspectableWebContentsImpl()
              InspectableWebContentsViewViews::InspectableWebContentsViewViews() : views::View // 承载一个网页，包装 WebView
                views::WebView() // !!!
                views::WebView::SetWebContents() // !!! 负责网页的呈现
                views::View::AddChildView(webview) // !!!
    WrappableBase::InitWithArgs()
      js new WebContentsView(webContents)
        WebContentsView::New()
          WebContentsView::WebContentsView() // 包装 InspectableWebContentsViewViews
      js BrowserWindow::setContentView(webContentsView)
        TopLevelWindow::SetContentView()
          NativeWindowViews::SetContentView()
            views::View::AddChildView() // !!! 将 InspectableWebContentsViewViews 添加到 rootview
    NativeWindow::InitFromOptions()
      NativeWindowViews::Show()
        views::internal::NativeWidgetPrivate::Show() // !!! 显示窗口
```

最终创建出来的 view 层级如下（省略了chromium内部的View）：

```c++
views::Widget
  electron::RootView
    electron::InspectableWebContentsViewViews
      views::WebView
```

## 嵌入自定义 View

从上面的介绍可以看到，如果想要在网页上添加一个 view 图层，可以添加到 `electron::InspectableWebContentsViewViews` 中，类似下面这样：

```c++
views::Widget
  electron::RootView
    electron::InspectableWebContentsViewViews
      MyView1
      views::WebView
      MyView2
```

`MyView1` 位于 `views::WebView` 之下，用来在网页下面渲染内容，`MyView2` 位于 `views::WebView` 之上，用来在网页上面渲染内容。在项目中根据需要选择。
