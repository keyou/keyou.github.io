---
title: ffmpeg 常用命令
tags:
  - ffmpeg
---

记录常用的 ffmpeg/ffprobe 命令，持续更新。

```sh
# 查看选定视频轨道中每一个 packet 信息及每一个 frame 信息
ffprobe -show_frames -show_packets -pretty -select_streams v:0 input.mp4

# 查看视频每一帧的类型及序号
ffprobe -show_frames -pretty -show_entries frame=pict_type,coded_picture_number -of default=noprint_wrappers=1 input.mp4

# 转码 baseline 24fps 24(max GOP) 200kbps(ABR)
ffmpeg -i input.mp4 -c:v libx264 -profile:v baseline -level:v 3.0 -r 24 -g 24 -b:v 200k -c:a copy output.mp4

# 转码 baseline 24fps 24(max GOP) 200kbps(ABR+VBV)
ffmpeg -i input.mp4 -c:v libx264 -profile:v baseline -level:v 3.0 -r 24 -g 24 -b:v 200k -maxrate 200k -bufsize 400k -c:a copy output.mp4

# 给视频画面添加时间戳
ffmpeg -i input.mp4  -vf drawtext="fontfile=/Library/Fonts/DroidSansMono.ttf:fontsize=72:fontcolor='white':boxcolor=0x000000AA:box=1:x=w/2-text_w/2:y=80:timecode='00\:00\:00\:00':rate=23.976:text=''" output.mp4

# 循环推 rtmp 流
ffmpeg -re -stream_loop -1 -i input.mp4 -c:v copy -c:a copy -f flv -y rtmp://output

# 音视频合成
ffmpeg -i input.mp4 -i input.aac -c:v libx264 -profile:v baseline -r 15 -g 15 -c:a copy -bsf:a aac_adtstoasc -shortest black2.flv
```

----

参考链接:

- [ffmpeg Documentation](https://ffmpeg.org/ffmpeg.html)
- [Understanding Rate Control Modes (x264, x265, vpx)](https://slhck.info/video/2017/03/01/rate-control.html)
- [EncodingForStreamingSites – FFmpeg](https://trac.ffmpeg.org/wiki/EncodingForStreamingSites)
- [ffprobe Documentation](https://ffmpeg.org/ffprobe.html)
- [HANDY TIP: Using FFprobe for stream analysis : Compression Techniques](https://forums.creativecow.net/thread/20/871651)
- [FFprobeTips – FFmpeg](https://trac.ffmpeg.org/wiki/FFprobeTips)
- [ffmpeg - How to identify i-frame from idr-frame in the ffprobe show frames output? - Video Production Stack Exchange](https://video.stackexchange.com/questions/19250/how-to-identify-i-frame-from-idr-frame-in-the-ffprobe-show-frames-output)
- [mp4 - Repeat/loop Input Video with ffmpeg? - Video Production Stack Exchange](https://video.stackexchange.com/questions/12905/repeat-loop-input-video-with-ffmpeg)
