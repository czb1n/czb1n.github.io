---
layout: post
author: czb1n
title:  "音频处理基础"
date:   2018-12-18 09:57:00
categories: [Audio]
tags: [Server, Audio, FFmpeg]
---

## 一、基础概念

- 采样频率（Sampling Rate），单位时间内采集的样本数，是采样周期的倒数，指两个采样之间的时间间隔。
> 采样频率必须至少是信号中最大频率分量频率的两倍，否则就不能从信号采样中恢复原始信号，这其实就是著名的香农采样定理。  
> CD音质(一般的音频)采样率为 44.1 kHz，人耳只能听到20Hz到20khz范围的声音。

- 量化深度，表示一个样本的二进制的位数，即样本的比特数。

- 声道数，记录声音时，如果每次生成一个声波数据，称为单声道；每次生成两个声波数据，称为双声道(立体声)。

- 文件大小（B）=采样频率（Hz）×录音时间（S）×（量化深度/8）× 声道数
> 如：录制1分钟采样频率为44.1KHz，量化深度为16位，立体声的声音（CD音质），文件大小为：
44.1×1000×60×(16/8)×2=10584000B≈10.09M

- 单位
  - dBSPL，通常所说的dB，使用声压作为被测量，选择20μPa作为基准值。
  - dBm，使用功率作为被测量，选择1mW作为基准值。
  - dBu，使用电压作为被测量，选择0.775V作为基准值。
  - dBV，和dBu一样，使用电压作为被测量，选择1V作为基准值。
  - dBFS，和上面的量都不相同，上面的量都是测量模拟值的，dBFS是测量数字音频的，其选择的基准值为sample的最大值为0dBFS，其他的值都为负值。

- PCM（Pulse Code Modulation）编码，即通过脉冲编码调制方法生成数字音频数据的技术或格式，是一种无损编码格式，是音频模拟信号数字化的一种方法，需要经过采样、量化和编码过程，以实现音频模拟信号数字化。

- 参考
> http://blog.jianchihu.net/pcm-volume-control.html

## 二、FFmpeg

- 参考
> https://trac.ffmpeg.org/wiki/Concatenate  
> https://blog.csdn.net/leixiaohua1020/article/details/15811977

### 2.1、基础说明

- ffmpeg 输入文件由 -i 参数指定，可以指定多个。

###  2.2、Filters

> 官网: http://www.ffmpeg.org/ffmpeg-filters.html  
命令参数为: -filter_complex

- Filter中需要使用到输入文件时则需要用到 **[]** 来指定流操作。

1. 输入文件中的某个流
```
[{输入文件编号}:{音频流a或视频流v}:{多个流中的第几个流}]
如: [0:a:0]
```
2. 流名称，一般为组合命令过程中的输出流。
```
[{流名称}]
如: [czb]
```

- 操作格式

> 一个 **Filter** 操作包含3个部分:

1. 输入流
2. 指令
3. 输出流

> 指令中的参数以冒号 **(:)** 隔开。

如: "[in] amix=inputs=2:duration=first [out]"

> 多个 Filter 可以组合使用。以分号 **(;)** 隔开。

#### 音频混合: amix

``` Java
inputs: 一共有多少个输入，默认为2
duration: 输出的时长定义枚举
    longest 等于最长的输入的长度 (默认值)
    shortest 等于最短的输入的长度
    first 等于第一个输入的长度
dropout_transition: 结束时淡出的持续时间(秒) 默认为 2 秒
```

#### 音频循环: aloop

``` Java
loop: 循环次数, -1 表示无限循环, 默认 为 0
size: 样本的最大的大小, 默认为 0
start: 循环的第一个样本, 默认为0
```
> 一般来说循环时 size 要足够大, 否则没有办法循环, 一般直接设为 2e+9

#### 拼接: concat

``` Java
n: 输入的个数
v: 输出视频流的数量
a: 输入音频流的数量
```

#### 音量调节

``` Java
[in] volume=X [out]

X值为调节的倍数
如原始音量为1, 增大一倍为2, 缩小一倍为0.5
```

### 2.3 其他

- 获取音频文件信息
``` Java
ffprobe -v quiet -print_format json -show_format -show_streams {file_name}
```
- 获取ffmpeg支持的音视频格式
``` Java
ffmpeg -formats
```
- 按照长度分割音频
``` Java
ffmpeg -i {输入音频} -f segment -segment_time {每一段的长度} -ac 1 -ar 16000 {输出音频 如:audio%02d.wav}
```
- 音频拼接
``` Java
ffmpeg -i {输入文件0} ... -i {输入文件n} -filter_complex "[{输入文件编号}:{音频流a或视频流v}:{第几个流}] ...  concat=n={一共多少个输入}:v={每个输入有多少个视频流输出}:a={每个输入有多少个音频流输出} {输出流的名称}" -map "{输出流的名称}" {输出文件}
如: ffmpeg -i part1.m4a -i part2.m4a -i part3.m4a -filter_complex "[0:a:0] [1:a:0] [2:a:0] concat=n=3:v=0:a=1 [czb]" -map "[czb]" output.m4a
```

## 三、pydub

### 3.1、安装

> https://github.com/jiaaro/pydub  
> pip install pydub

### 3.2、基础使用

- 主要使用的是**SegmentAudio**类
``` Python
from pydub import AudioSegment
```
- 引入文件
``` Python
ogg_version = AudioSegment.from_ogg("never_gonna_give_you_up.ogg")
flv_version = AudioSegment.from_flv("never_gonna_give_you_up.flv")
mp4_version = AudioSegment.from_file("never_gonna_give_you_up.mp4", "mp4")
wma_version = AudioSegment.from_file("never_gonna_give_you_up.wma", "wma")
aac_version = AudioSegment.from_file("never_gonna_give_you_up.aiff", "aac")
```
- 音量调节可以直接使用+/-来操作增减db

- 导出音频
``` Python
{AudioSegment Object}.export("{filename}.mp3", format="mp3", bitrate="{bitrate}")
```

