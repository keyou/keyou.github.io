---
title: 直播流延迟分析
tags:
  - livestream
---

直播流要经过3个端： `推流端` -> `CDN` -> `拉流端`。

直播中可能出现下面三种现象：

现象1：拉流端即使刷新也无法和推流端的时间进行同步  

问题在推流端： 推流端的逻辑为 `编码`->`缓存`->`推流`，如果推流速度小于编码码率，产生缓存堆积。  
验证方法：在推流端设置设备出站带宽限制，使用 ffmpeg 推流，观察推流 speed 参数，使 speed < 1x，此时推流速度小于编码比特率，此时拉流则可复现改种现象。

现象2：播放卡顿，重新拉流后延时（几乎）消失；  
问题在拉流端： 拉流端的逻辑为 `拉流`->`缓存`->`解码/播放`，如果拉流速度小于解码/播放速度，由于没有足够的数据进行解码，导致播放卡顿，卡顿的时间会累积到整体延时上，如果重新拉流则会直接拉取到最新的数据从而延时消失。如果拉流速度大于解码/播放速度，则数据堆积在播放端的缓存中，此时通过播放器倍速播放或者跳转到缓存末端可以降低延时，此时不需要重新拉流。  
验证方法：在拉流端设置设备入站带宽限制，使拉流速度小于视频比特率，此时拉流则可复现改种现象。暂停视频播放一段时间，可以模拟缓存堆积问题。  

CDN  
CDN的逻辑为 `接收流`->`缓存`->`发送流`，当客户端从CDN拉流时，CDN会为每个拉流连接维护一个缓存，该缓存记录了当前拉流端的播放进度。如果发送流的速度小于接收流的速度，则会产生CDN缓存堆积（该缓存可以大于几分钟），此时如果拉流端重新建立一条拉流连接重新拉流则会拉到CDN中的最新数据。

现象3：播放花屏  
花屏有两种可能，一种是数据帧不完整，一种是编码比特率过低。在全链路使用TCP的情况下，只有当编码比特率对于当前画面过低时才能导致花屏。当链路中有部分使用UDP时，会由于丢包导致数据帧不完整，从而导致花屏，此时可以通过丢弃不完整的数据帧来避免花屏。

综上，要想实现不卡顿，低延迟，需要尽可能使推流端的编码码率和播放端的拉流速度匹配。要想实现不花屏，需要保证不渲染不完整的数据帧或者保证编码比特率可以满足当前画面。

建议在应用中监控推流、拉流以及视频的码率，从而判断当前的延迟是由哪部分产生的。

其他：

- 丢包会极大影响 TCP 的可用带宽；
- 使用 flv.js 播放视频有三个缓存，分别是 `IOController` 中的 `_stashBuffer`，它缓存直接从网络上获取到的数据，通过将 `enableStashBuffer` 设为 false 可以关闭主动缓存，除非数据无法完整解码；然后是 `MSEController` 中 `_pendingSegments`，它缓存传给 SourceBuffer 的数据，一般不会产生堆积，它会尽最大努力将数据放入SourceBuffer；最后是浏览器video本身的缓存，位于浏览器中的缓存可以通过跳转或者倍速播放进行追帧。

使用 mpv 的低延时模式播放直播流，播放后使用 `I` 显示 OSD 信息：

```sh
mpv --profile=low-latency --no-cache --untimed <flv address>
```

![mpv-delay](/data/mpv-delay.png)

使用 ffmpeg 推流：

```sh
$ ffmpeg -re -i 41ts.mp4 -c:v copy -c:a copy -f flv -y rtmp://...
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from '41ts.mp4':
  Metadata:
    major_brand     : isom
    minor_version   : 512
    compatible_brands: isomiso2avc1mp41
    encoder         : Lavf57.83.100
  Duration: 00:04:11.47, start: 0.000000, bitrate: 1071 kb/s
    Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p, 856x480 [SAR 254:255 DAR 13589:7650], 935 kb/s, 23.98 fps, 23.98 tbr, 24k tbn, 47.95 tbc (default)
    Metadata:
      handler_name    : VideoHandler
    Stream #0:1(und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, stereo, fltp, 131 kb/s (default)
    Metadata:
      handler_name    : SoundHandler
Output #0, flv, to 'rtmp://...':
  Metadata:
    major_brand     : isom
    minor_version   : 512
    compatible_brands: isomiso2avc1mp41
    encoder         : Lavf57.83.100
    Stream #0:0(und): Video: h264 (High) ([7][0][0][0] / 0x0007), yuv420p, 856x480 [SAR 254:255 DAR 13589:7650], q=2-31, 935 kb/s, 23.98 fps, 23.98 tbr, 1k tbn, 24k tbc (default)
    Metadata:
      handler_name    : VideoHandler
    Stream #0:1(und): Audio: aac (LC) ([10][0][0][0] / 0x000A), 44100 Hz, stereo, fltp, 131 kb/s (default)
    Metadata:
      handler_name    : SoundHandler
Stream mapping:
  Stream #0:0 -> #0:0 (copy)
  Stream #0:1 -> #0:1 (copy)
Press [q] to stop, [?] for help
frame= 1096 fps= 24 q=-1.0 size=    5155kB time=00:00:45.60 bitrate= 925.9kbits/s speed=   1x
```
