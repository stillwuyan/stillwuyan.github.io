---
title: FFprobe常用命令
date: 2018-01-09 13:53:17
tags: FFprobe
categories: Technology
---

记录FFprobe常用的命令，留作备忘。

#### 1. 查看视频容器的信息，使用`-show_format`：
```
ffprobe -v quiet -hide_banner -show_format test.mp4
```

#### 2. 查看视频流的信息，使用`-show_streams`：
```
ffproge -v quiet -hide_banner -show_streams test.mp4
```

#### 3. 如果不使用`-hide_banner`参数，输出内容中将包含程序编译信息：
```
ffprobe version N-86994-g92da230 Copyright (c) 2007-2017 the FFmpeg developers
  built with gcc 7.1.0 (GCC)
  configuration: --enable-gpl --enable-version3 --enable-cuda --enable-cuvid --enable-d3d11va --enable-dxva2 --enable-libmfx --enable-nvenc --enable-avisynth --enable-bzlib --enable-fontconfig --enable-frei0r --enable-gnutls --enable-iconv --enable-libass --enable-libbluray --enable-libbs2b --enable-libcaca --enable-libfreetype --enable-libgme --enable-libgsm --enable-libilbc --enable-libmodplug --enable-libmp3lame --enable-libopencore-amrnb --enable-libopencore-amrwb --enable-libopenh264 --enable-libopenjpeg --enable-libopus --enable-librtmp --enable-libsnappy --enable-libsoxr --enable-libspeex --enable-libtheora --enable-libtwolame --enable-libvidstab --enable-libvo-amrwbenc --enable-libvorbis --enable-libvpx --enable-libwavpack --enable-libwebp --enable-libx264 --enable-libx265 --enable-libxavs --enable-libxvid --enable-libzimg --enable-lzma --enable-zlib
  libavutil      55. 74.100 / 55. 74.100
  libavcodec     57.102.100 / 57.102.100
  libavformat    57. 76.100 / 57. 76.100
  libavdevice    57.  7.100 / 57.  7.100
  libavfilter     6. 99.100 /  6. 99.100
  libswscale      4.  7.102 /  4.  7.102
  libswresample   2.  8.100 /  2.  8.100
  libpostproc    54.  6.100 / 54.  6.100
```

#### 4. 如果不使用`-v quiet`参数，输出内容中将包含调试信息。调试信息的详细程度也可以通过`-v info`/`-v verbose`/`-v debug`等具体参数来调节：
```
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from 'test.mp4':
  Metadata:
    major_brand     : isom
    minor_version   : 1
    compatible_brands: isomavc1
    creation_time   : 2016-02-19T08:21:53.000000Z
  Duration: 02:30:48.75, start: 0.000000, bitrate: 3248 kb/s
    Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p, 1280x720 [SAR 1:1 DAR 16:9], 3052 kb/s, 25 fps, 25 tbr, 25k tbn, 50 tbc (default)
    Metadata:
      creation_time   : 2016-02-19T08:21:53.000000Z
      handler_name    : 264:fps=25.0@GPAC0.5.1-DEV-rev4929
    Stream #0:1(und): Audio: aac (LC) (mp4a / 0x6134706D), 48000 Hz, stereo, fltp, 191 kb/s (default)
    Metadata:
      creation_time   : 2016-02-19T07:16:54.000000Z
      handler_name    : Sound Media Handler
```

#### 5. 使用`-print_format`参数可以设置输出内容的格式化形式，如`json`、`xml`或`csv`等：
```
ffprobe -v quiet -hide_banner -show_format -pretty -print_format json=compact=0 test.mp4
{
    "format": {
        "filename": "test.mp4",
        "nb_streams": 2,
        "nb_programs": 0,
        "format_name": "mov,mp4,m4a,3gp,3g2,mj2",
        "format_long_name": "QuickTime / MOV",
        "start_time": "0:00:00.000000",
        "duration": "2:30:48.746667",
        "size": "3.422155 Gibyte",
        "bit_rate": "3.248636 Mbit/s",
        "probe_score": 100,
        "tags": {
            "major_brand": "isom",
            "minor_version": "1",
            "compatible_brands": "isomavc1",
            "creation_time": "2016-02-19T08:21:53.000000Z"
        }
    }
}
```

#### 6. 使用`-show_frames`参数可以显示各条流的逐帧信息，使用`-select_streams`参数可以选择查看哪条流：
```
ffprobe -v quiet -hide_banner -show_frames -select_streams video test.mp4 | more            
[FRAME]
media_type=video
stream_index=0
key_frame=1
pkt_pts=150545
pkt_pts_time=1.672722
pkt_dts=150545
pkt_dts_time=1.672722
best_effort_timestamp=150545
best_effort_timestamp_time=1.672722
pkt_duration=N/A
pkt_duration_time=N/A
pkt_pos=564
pkt_size=22251
width=202
height=360
pix_fmt=yuv420p
sample_aspect_ratio=N/A
pict_type=I
coded_picture_number=0
display_picture_number=0
interlaced_frame=0
top_field_first=0
repeat_pict=0
[/FRAME]
[FRAME] 
media_type=video
stream_index=0
key_frame=0
pkt_pts=162818
pkt_pts_time=1.809089
pkt_dts=162818
pkt_dts_time=1.809089
best_effort_timestamp=162818
best_effort_timestamp_time=1.809089
pkt_duration=N/A
--More--
```

#### 7. 使用`-show_entries`参数可以筛选显示的`entries` 和`sections`。因此如果知道`entries`和`sections`，可以直接使用该参数筛选出想要的信息：
```
ffprobe -v quiet -hide_banner -select_streams video -show_entries format:frame=media_type,pict_type test.mp4 | more
[FRAME]
media_type=video
pict_type=I
[/FRAME]
[FRAME]
media_type=video
pict_type=P
[/FRAME]
[FRAME]
media_type=video
pict_type=P
[/FRAME]
[FRAME]
media_type=video
pict_type=P
[/FRAME]
[FRAME]
media_type=video
pict_type=P
[/FRAME]
[FRAME]
media_type=video
pict_type=B
[/FRAME]
...
[FORMAT]
filename=/home/gztv/data/test/dst/live/201704/05/abcd1234/478k/abcd1234.m3u8
nb_streams=2
nb_programs=1
format_name=hls,applehttp
format_long_name=Apple HTTP Live Streaming
start_time=1.649500
duration=102.275014
size=1646
bit_rate=128
probe_score=100
[/FORMAT]
--More--
```