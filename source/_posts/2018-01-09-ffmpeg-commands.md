---
title: FFmpeg常用命令
date: 2018-01-09 13:44:15
tags: FFmpeg
categories: Technology
---
记录FFmpeg常用的命令，留作备忘。

#### 1. 将本地文件推流至RTMP Server，并循环播放
```
ffmpeg -stream_loop -1 -re -i ./file.mp4 -c copy -f flv "rtmp://localhost:1935/live/video live=1 timeout=5"
```

#### 2. 将音频流转发到其他RTMP Server
```
ffmpeg -probesize 1M -analyzeduration 5M "rtmp://localhost:1935/live/audio1 live=1 timeout=5" -c copy -f flv "rtmp://localhost:1935/live/audio2 live=1 timeout=5"
```

#### 3. 将视频缩放至指定分辨率（640x320），宽高比不一致时填充黑边
```
ffmpeg -i ./file.mp4 -vf "scale=iw*min(640/iw\,360/ih):ih*min(640/iw\,360/ih),pad=640:360:(640-iw)/2:(320-ih)/2" -c:v libx264 -crf 18 -c:a aac -f mp4 ./output.mp4
```

#### 4. `-vf`和`-filter_complex`的区别：
   + `-vf`只能构建简单的`filter graph`，它只允许有一个输入和一个输出，即：`ffmpeg`的`-i`参数只能有一个。但是，可以在`-vf`中添加`amovie`或`movie`插件实现多输入，例如：
     ```
     ffmpeg -i ./video.mp4 -vf "movie=./image.jpg,scale=102x58[watermark];[in]scale=872x480[out];[out][watermark]overlay=0:422" -c:v libx264 -c:a aac -f mp4 ./output.mp4 -y 
     ```

   + `-filter_complex`可以构建复杂的`filter graph`，因此可以设置多输入和多输出，例如：
     ```
     ffmpeg -i ./video.mp4 -i ./logo1.jpg -i ./logo2.jpg -filter_complex "[0]scale=654:360[video0];[1]scale=39:36[watermark1];[video0][watermark1]overlay=19:14[video1];[2]scale=39:36[watermark2];[video1][watermark2]overlay=327:251[video2]" -map [video2] -map 0:a? -c:v libx264 -c:a aac -f mp4 ./output.mp4 -y
     ```

     > **注意：**`-map 0:a?`后面的`?`表示如果音频流不存在，则该`-map`参数可以被忽略，ffmpeg可以继续执行后续动作。另外，`?`只能忽略不存在的`stream`，如果`input`不存在仍然会导致ffmpeg退出，即`0`必须是有效的`input file`。

## 5. Speeding up / slowing down video，使用`setpts`插件，而且该插件仅修改视频速率，不会改变音频速率，因此可以使用`-an`静音。
   + 2倍速率。加速后FFmpeg将抛弃部分frame，为了避免丢帧，可以将frame rate设置为原来2倍（如：原视频是15 fps）：
     ```
     ffmpeg -i input.mkv -r 30 -filter:v "setpts=0.5*PTS" output.mkv
     ```

   + 0.5倍速率（慢速）。设置`PTS`值大于1即可达到慢速效果：
     ```
     ffmpeg -i input.mkv -filter:v "setpts=2.0*PTS" output.mkv
     ```

   > **注意1：**使用`setpts`插件加速视频时，FFmpeg需要快速的读取文件，因此不能与`-re`（按照视频帧率读取内容）选项共用。
   >
   > **注意2：**使用`setpts`插件将导致视频重编码，因此，使用时建议添加`-c:v libx264 -crf 18`参数提升编码质量。

### 6. Speeding up / slowing down audio，使用`atempo`插件。该插件只能设置`[0.5 - 2.0]`范围的值，如果想设置更大范围的速率，可以使用`chain filter`方式。
   + 4倍速率。
     ```
     ffmpeg -i input.mkv -filter:a "atempo=2.0,atempo=2.0" -vn output.mkv
     ```

   + 同时调整视频和音频的速率（2倍速率）：
     ```
     ffmpeg -i input.mkv -filter_complex "[0:v]setpts=0.5*PTS[v];[0:a]atempo=2.0[a]" -map "[v]" -map "[a]" output.mkv
     ```

#### 7. 视频显示大小由分辨率和宽高比决定，ffmpeg提供宽高值，SAR和DAR参数控制视频显示大小。
   + DAR - Display Aspect Ratio 显示横纵比。最终显示的图像在长度单位上的横纵比。
   + SAR - Sample Aspect Ratio 采样横纵比。表示横向像素点数和纵向像素点数的比值。
   + `DAR = HORIZONTAL_RESOLUTION / VERTICAL_RESOLUTION * SAR`
   + **注意：**如果视频不含DAR和SAR信息，使用ffprobe查看则显示为`1:0`。

将test.mp4按照原显示纵横比等比缩放至640x360（16:9），如果纵横比不一致则添加黑边。

```
ffmpeg -i test.mp4 -vf "[0]scale=iw*min(sar\,1)*min(640/iw/min(sar\,1)\,360/ih*max(sar\,1)):ih/max(sar\,1)*min(640/iw/min(sar\,1)\,360/ih*max(sar\,1)),pad=640:360:(640-iw)/2:(360-ih)/2[video0]" -map [video0] -map 0:a? -c:v h264 -b:v 120k -c:a aac -b:a 64k -g 30 -max_muxing_queue_size 512 -y out.mp4
```

#### 8. 使mp4支持渐进式下载，使用`-movflags faststart`命令将metadata（moov）放在data之前，或者使用`-movflags frag_keyframe`以fragmented（moof）方式组织box

```
file -> file:
ffmpeg -i test.mp4 -c copy -movflags faststart test_progress.mp4
ffmpeg -i test.mp4 -c copy -movflags frag_keyframe test_fragmented.mp4

stream -> segments:
ffmpeg -i rtmp://localhost/live/test -c copy -f segment -segment_time 5 -segment_format_options movflags=frag_keyframe test%d.mp4
ffmpeg -i rtmp://localhost/live/test -c copy -f segment -segment_time 5 -segment_format_options movflags=frag_keyframe test%d.mp4
```
#### 9. 提取视频关键帧

```
ffmpeg -v debug -i video.webm -vf "select=eq(pict_type\,I)" -vsync vfr thumb%04d.jpg -hide_banner 2> log.txt
```

+ -vsync vfr: This is a parameter that tells the filter to use a variable bitrate video synchronization. If we do not use this parameter ffmpeg will fail to find only the keyframes and shoud extract other frames that can be not processed correctly.
+ -v debug: You can use the detail log to find the time stamp of the key frame.