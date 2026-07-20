+++
date = '2026-07-20T09:16:02+08:00'
draft = false
title = '从 Linux PC 跑通音频播放开始'
+++

# 嵌入式音视频入门：从 Linux PC 跑通音频播放开始

刚开始学习嵌入式音视频时，最容易迷茫的问题是：到底应该先学什么？是先看 FFmpeg？还是先看 ALSA？还是直接上开发板？

我的建议是：**不要一开始就上开发板，也不要直接啃大项目。先在 Linux PC 上把最小音视频链路跑通。**

比如：

```text
摄像头采集 -> 视频编码 -> 保存 MP4
麦克风采集 -> 音频编码 -> 保存 WAV/AAC
音频文件 -> 解码成 PCM -> ALSA 播放
```

这篇文章先总结音频部分，尤其是如何播放 `wav / flac / aac / m4a` 这类音频文件。

## 一、ALSA 是干什么的？

ALSA 是 Linux 下常用的音频接口，全称是 Advanced Linux Sound Architecture。

它主要负责和声卡设备打交道，比如：

```text
录音：从麦克风读取 PCM 数据
播放：把 PCM 数据写入声卡
```

这里有一个很重要的点：

> ALSA 只能播放 PCM，不能直接播放 flac、aac、m4a 这种压缩格式。

也就是说，如果你有一个 `test.flac` 或 `test.m4a` 文件，不能直接把文件内容丢给 ALSA 播放。

正确流程是：

```text
音频文件
  ↓
FFmpeg 解封装
  ↓
FFmpeg 解码
  ↓
得到 PCM 数据
  ↓
必要时重采样 / 转格式
  ↓
ALSA 播放 PCM
```

所以实际项目里通常是：

```text
FFmpeg 负责解码
ALSA 负责播放
```

## 二、为什么 WAV 可以特殊处理？

`wav` 是一种容器格式，里面经常装的是 PCM 数据。

所以理论上，播放 WAV 可以自己解析 WAV 头，然后把后面的 PCM 数据送给 ALSA。

但是新人阶段不建议一开始就分开处理：

```text
wav 走一套逻辑
flac 走一套逻辑
aac 走一套逻辑
m4a 走一套逻辑
```

这样很容易把自己绕进去。

更推荐统一使用 FFmpeg：

```text
不管输入 wav / flac / aac / m4a
全部交给 FFmpeg 解码成 PCM
然后交给 ALSA 播放
```

这样代码结构更清晰。

## 三、播放音频的整体流程

播放 `wav / flac / aac / m4a` 的通用流程如下：

```text
1. 打开音频文件
2. 读取媒体信息
3. 找到音频流
4. 找到对应解码器
5. 打开解码器
6. 解码 packet 得到 frame
7. 把 frame 转成 ALSA 支持的 PCM 格式
8. 调用 ALSA 播放
9. 播放结束后释放资源
```

对应到 FFmpeg 和 ALSA API，大概是：

```c
avformat_open_input();          // 打开输入文件
avformat_find_stream_info();    // 读取媒体信息
av_find_best_stream();          // 找到音频流

avcodec_find_decoder();         // 找解码器
avcodec_alloc_context3();       // 创建解码器上下文
avcodec_parameters_to_context();// 拷贝流参数
avcodec_open2();                // 打开解码器

swr_alloc_set_opts2();          // 创建重采样上下文
swr_init();                     // 初始化重采样

snd_pcm_open();                 // 打开 ALSA 播放设备
snd_pcm_hw_params_*();          // 设置播放参数
snd_pcm_prepare();              // 准备播放

av_read_frame();                // 读取压缩数据
avcodec_send_packet();          // 送入解码器
avcodec_receive_frame();        // 取出解码后的 PCM frame
swr_convert();                  // 转成目标 PCM 格式
snd_pcm_writei();               // 写入声卡播放
```

## 四、为什么需要重采样？

FFmpeg 解码出来的音频格式不一定是 ALSA 当前配置的格式。

比如一个文件可能是：

```text
采样率：44100 Hz
声道数：1
采样格式：float planar
```

但我们希望 ALSA 固定播放：

```text
采样率：48000 Hz
声道数：2
采样格式：S16_LE
```

所以中间需要 `libswresample` 做转换。

常见目标格式可以先固定成：

```text
采样率：48000
声道数：2
格式：S16_LE
布局：stereo
```

这样 ALSA 配置也比较简单。

## 五、ALSA 播放 PCM 的基本流程

如果只看 ALSA，本质流程是：

```c
snd_pcm_open();                  // 打开播放设备
snd_pcm_hw_params_malloc();      // 创建硬件参数对象
snd_pcm_hw_params_any();         // 初始化参数
snd_pcm_hw_params_set_access();  // 设置访问方式
snd_pcm_hw_params_set_format();  // 设置采样格式
snd_pcm_hw_params_set_channels();// 设置声道数
snd_pcm_hw_params_set_rate();    // 设置采样率
snd_pcm_hw_params();             // 应用参数
snd_pcm_prepare();               // 准备播放

while (...) {
    snd_pcm_writei();            // 写 PCM 数据到声卡
}

snd_pcm_close();                 // 关闭设备
```

播放和录音很像。

区别是：

```c
// 播放
snd_pcm_open(&pcm, "default", SND_PCM_STREAM_PLAYBACK, 0);
snd_pcm_writei(pcm, buffer, frames);

// 录音
snd_pcm_open(&pcm, "default", SND_PCM_STREAM_CAPTURE, 0);
snd_pcm_readi(pcm, buffer, frames);
```

## 六、最小播放器架构

第一版不要上线程，先写单线程版本。

建议结构：

```text
audio_player/
├── main.c
├── audio_decode.c
├── audio_output.c
└── CMakeLists.txt
```

逻辑可以先写成：

```text
main.c:
  解析输入文件路径
  初始化 FFmpeg 解码器
  初始化 ALSA 播放设备
  循环读取、解码、转换、播放
  释放资源
```

核心循环大概是：

```c
while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index != audio_stream_index) {
        av_packet_unref(pkt);
        continue;
    }

    avcodec_send_packet(codec_ctx, pkt);

    while (avcodec_receive_frame(codec_ctx, frame) == 0) {
        swr_convert(...);
        snd_pcm_writei(pcm_handle, pcm_buffer, out_samples);
    }

    av_packet_unref(pkt);
}
```

## 七、编译依赖

Ubuntu 上可以安装：

```bash
sudo apt install ffmpeg \
libavformat-dev libavcodec-dev libavutil-dev \
libswresample-dev libasound2-dev
```

编译时链接：

```bash
gcc player.c -o player \
-lavformat -lavcodec -lavutil -lswresample -lasound
```

运行：

```bash
./player test.wav
./player test.flac
./player test.aac
./player test.m4a
```

注意：你之前写的 `acc` 一般应是 `aac`。

## 八、后面怎么加线程？

第一版单线程播放跑通后，再升级成线程模型。

可以设计成：

```text
解码线程：
  读取文件
  解码音频
  转成 PCM
  放入 PCM 队列

播放线程：
  从 PCM 队列取数据
  调用 snd_pcm_writei()
  写入声卡
```

结构类似：

```c
pthread_create(&decode_thread, NULL, decode_thread_func, NULL);
pthread_create(&play_thread, NULL, play_thread_func, NULL);

pthread_join(decode_thread, NULL);
pthread_join(play_thread, NULL);
```

但新人阶段不要急着加线程。先把最小链路跑通：

```text
文件 -> FFmpeg 解码 -> PCM -> ALSA 播放
```

这个链路通了，再谈暂停、继续、进度条、倍速、seek、音量控制、线程队列。

## 九、推荐学习顺序

如果目标是做嵌入式音视频应用，建议路线是：

```text
1. ALSA 录音：麦克风 -> PCM
2. ALSA 播放：PCM -> 声卡
3. FFmpeg 解码：wav/flac/aac/m4a -> PCM
4. FFmpeg 编码：PCM -> AAC
5. FFmpeg 封装：AAC -> m4a/mp4
6. V4L2 摄像头采集
7. 音视频一起录制成 MP4
8. 加线程、队列、同步
9. 移植到开发板
```

不要一开始就看 VLC、OBS、GStreamer 这种大项目。它们很强，但对新人来说信息密度太高。

更适合先看：

```text
ALSA 官方 pcm.c 示例
FFmpeg 官方 doc/examples
简单 V4L2 抓图项目
```

## 十、总结

播放 `wav / flac / aac / m4a` 的关键不是 ALSA 本身，而是要理解音频链路：

```text
文件格式 != PCM 数据
压缩音频必须先解码
ALSA 只负责播放 PCM
FFmpeg 负责解封装和解码
```

最实用的第一版目标是：

```text
输入任意音频文件
用 FFmpeg 解码成 48000Hz / stereo / S16_LE PCM
用 ALSA 播放出来
```

只要这个播放器能跑起来，你就已经掌握了嵌入式音频应用里非常核心的一条链路。下一步再把它和录音、摄像头、MP4 封装结合起来，就可以慢慢形成一个完整的音视频小项目。
