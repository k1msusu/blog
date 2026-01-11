---
layout: post
title:	"Wecome to http!!"
date:	2026-01-05 1:07:50 +0800
categories: jekyll update
---
## ffmpeg实战 Cheat sheet

### 一、视频信息探查

```
# 腾讯转码生成的 15s 竖屏视频，分辨率，帧数，声道，主流组合
ffmprobe .\2062650031.mp4 解析
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from '.\2062650031.mp4':  # 识别容器格式
  Metadata:  # 文件级描述标签
    major_brand     : isom  # mp4 的主格式标识
    minor_version   : 512  # 格式的小版本号，核心格式标识
    compatible_brands: isomiso2avc1mp41  # 文件兼容的格式标识
    encoder         : Lavf58.20.100  # 生成的视频编码器版本
    description     : Tencent CAPD MTS (T-tech)  # 视频的转码服务来源
  Duration: 00:00:15.21, start: 0.000000, bitrate: 2029 kb/s  # 核心全局信息: 时长，起始时间戳，整体码率（视频+音频）
  Stream #0:0[0x1](und): Video: h264 (High) (avc1 / 0x31637661), yuv420p(tv, bt709, progressive), 540x960, 1973 kb/s, 30 fps, 30 tbr, 15360 tbn (default)  # 视频流的编号，视频的语言标识，变频编码格式和编码档次，编码的封装标识，视频的像素格式+色彩标准+扫描方式，视频分辨率，视频流的单独码率，视频帧数，实际帧率，时间基，默认视频流
    Metadata:
      handler_name    : VideoHandler  # 视频流处理器
      vendor_id       : [0][0][0][0]  # 厂商ID
  Stream #0:1[0x2](und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, mono, fltp, 48 kb/s (default)  # 音频编码格式+音频档次，封装标识，音频采样率，声道数，音频采样格式，音频流单独码率，默认音频流
    Metadata:
      handler_name    : SoundHandler
      vendor_id       : [0][0][0][0]
```

```
# 默认转码
ffprobe .\1111.avi
Input #0, avi, from '.\1111.avi':
  Metadata:  # 转码是重新编码+重新封装
    software        : Lavf62.3.100  # 视频编码器版本升级
  Duration: 00:00:15.28, start: 0.000000, bitrate: 978 kb/s  # 多了 7 帧，帧对齐误差，总码率腰斩
  Stream #0:0: Video: mpeg4 (Simple Profile) (FMP4 / 0x34504D46), yuv420p, 540x960 [SAR 1:1 DAR 9:16], 900 kb/s, 30 fps, 30 tbr, 30 tbn  # mpeg4 是非常老旧的编码格式，Simple Profile 是最低档次，进行 AVI 转码时的默认视频编码
  Stream #0:1: Audio: mp3 (mp3float) (U[0][0][0] / 0x0055), 44100 Hz, mono, fltp, 64 kb/s # 音频编码格式改变为默认编码
```

```
# 简易查询
ffprobe -v error -show_entries stream=index,codec_name,width,height,r_frame_rate -of  json .\2062650031.mp4
```

### 二、转封装 vs 转码

```
ffmpeg -i in.mp4 -c copy out.mkv  # 继承原来的编码格式，差异在于封装开销
ffmpeg -i in.mp4 -c:v libx264 -b:v 2000k -c:a aac out.mp4
ffmpeg -i in.mp4 -vf scale=1280:720 -b:v 2000k -c:a aac out_720p  # 更改分辨率
ffmpeg -i in.mp4 -ss 00:00:05 -vframes 1 out.jpg  # 截图
ffmpeg -i in.mp4 -vn -c:a copy out.acc  # 提取音频
```

### 三、HLS 多分辨率 <索引>

```
# 单分辨率 HLS
ffmpeg -i in.mp4 -vf scale=1280:720 -b:v 2000k -c:a aac -f hls -hls_time 6 -hls_list_size 0 -hls_playlist_type vod out_720p.m3u8
```

```
# 多分辨率，三次转换
# -hls_list_size 0 播放列表中保留的分片数量无限制
ffmpeg -i in.mp4 -vf scale=1920:1080 -b:v 5000k -c:a aac -f hls -hls_time 6 -hls_list_size 0 -hls_playlist_type vod out_1080p.m3u8
ffmpeg -i in.mp4 -vf scale=1280:720 -b:v 3000k -c:a aac -f hls -hls_time 6 -hls_list_size 0 -hls_playlist_type vod out_720p.m3u8
ffmpeg -i in.mp4 -vf scale=854:480 -b:v 1500k -c:a aac -f hls -hls_time 6 -hls_list_size 0 -hls_playlist_type vod out_480p.m3u8
# 手动编写 master.m3u8 内容
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2"
out_1080p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720,CODECS="avc1.640020,mp4a.40.2"
out_720p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1500000,RESOLUTION=854x480,CODECS="avc1.64001e,mp4a.40.2"
out_480p.m3u8
```

```
# 多分辨率，单次转换
ffmpeg -i in.mp4 -c:a aac -hls_time 6 -hls_list_size 0 -hls_playlist_type vod -hls_segment_filename "hls_%v_%d.ts" -var_stream_map "v:0,a:0 v:1,a:1 v:2,a:2" -master_pl_name "master.m3u8" -vf "scale=1920:1080[v0];scale=1280:720[v1];scale=854:480[v2]" -map "[v0]" -b:v:0 5000k -map "[v1]" -b:v:1 3000k -map "[v2]" -b:v:2 1500k -map a:0 out_%v.m3u8
```

### 四、批量处理 & 自动化

```
# 批量转码
for file in *.mp4; do
    ffmpeg -i "$file" -c:v libx264 -b:v 2000k -c:a acc "out_$file"
done
# 批量生成多分辨率 HLS
for file in *.mp4; do
    ffmpeg -i "$file" -vf scale=1920:1080 -b:v 5000k -c:a aac -f hls -hls_time 6 -hls_list_size 0 -hls_playlist_type vod "${file%.mp4}"_1080p.m3u8
    ffmpeg -i "$file" -vf scale=1280:720 -b:v 3000k -c:a aac -f hls -hls_time 6 -hls_list_size 0 -hls_playlist_type vod "${file%.mp4}"_720p.m3u8
    ffmpeg -i "$file" -vf scale=854:480 -b:v 1500k -c:a aac -f hls -hls_time 6 -hls_list_size 0 -hls_playlist_type vod "${file%.mp4}"_480p.m3u8
done
```

### 五、性能优化

```
# 控制画质
- crf 23
-preset fast
# 多线程
- threads 4
# 硬件加速
- hwaccel cuda -c:v h264_nvenc
# shell 脚本 -> 对象存储 -> 数据库映射目录
# 直播 : 有滑动窗口、list_size > 0 、会丢失前面的数据段
# 点播 : 无滑动窗口、list_size = 0 、必须全部保留 ts
================================
# 一个正确的 HLS 转码格式 采用一条滑动的 LIVE 和一条不滑动的 DVR
ffmpeg -i in.mp4
-c:v libx264 -g 150 -keyint_min 150 - scenecut 0 -vf scale=1280:720 -b:v 3000k -c:a acc -f hls -hls_time 6 -hls_playlist_type event -hls_list_size 0 720.m3u8
# scenecut 强制场景切换 I 帧
# -g 25 * 6 = 150 每 6 s 一个 I 帧 ， 25 fps ，结合后强制 150 帧一个 I 帧

```
