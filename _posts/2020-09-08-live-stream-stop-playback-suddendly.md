---
title: flv.js 直播异常卡死问题分析
published: false
tags:
  - livestream
  - media
---

业务中使用 zego SDK 推流到 zego 服务器，然后 zego 服务器推流到 CDN 服务器，最后客户端使用 flv.js 播放直播流。当推流端丢包率较高的时候（15%）会导致 flv.js 播放卡死。下面记录了该问题的处理过程。

首先确认视频流是否有问题？使用 mpv 播放器播放直播流是正常的，不会出现卡死现象，说明直播流是没有问题的。
