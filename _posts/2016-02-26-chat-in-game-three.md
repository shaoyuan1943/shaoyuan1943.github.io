---
layout: post
title: "AMR音频在Android和iOS上的应用"
date:   2016-02-26
categories: Android-iOS
--- 

之前已经确定游戏内语音聊天Android与iOS平台上采用AAC音频格式文件交互。这几日又查了仔细思索了下，感觉采用AAC格式的音频文件大小还是略大，在一定程度对网络交互还是有影响的。再次查了下资料，发现采用AMR格式的文件大小比AAC更小，经过测试之后发现的确如此，10s左右的语音采用AMR格式的文件大小可以稳定在15k左右。

现实情况Android支持AMR，但是iOS在4.3.x版本之后不再支持AMR，剔除了AMR的硬解。但通过opencore这个codec依旧可以在iOS平台上对AMR进行软解，以实现AMR转换到其他的格式可以播放。

最终确定采用AMR作为两平台的语音交互格式，不同的是，在iOS平台上需要进行WAV和AMR之间的转换。好在libopencore可以解决这个事情。

在折腾过程中也踩到一个坑，我把Android上录制的AMR放到iOS上进行转换播放时会出现问题，无法进行转换。尝试调整了各种参数也无法解决，偶然想到是不是因为两者解码时的文件参数不一样，于是下载了MediaInfo对比了Android和iOS上录制的AMR文件，发现两者参数不一样，于是我尝试去调整一下Android上录制AMR的参数：

    recorder.setOutputFormat(MediaRecorder.OutputFormat.AMR_NB);
    recorder.setAudioEncoder(MediaRecorder.AudioEncoder.AMR_NB);
    recorder.setAudioChannels(1);

一定要输出和编码格式都是AMR\_NB格式的，才能在两者正常通用。

由于Android原本就支持AMR的录制和播放，iOS上并不支持，所以需要用到第三方库libopencore对AMR文件进行进行转换方能播放。

所以我封装了一个在iOS上录制和播放的实现，并且支持了WAV和AMR文件之间的转换，实现文件在[**这里**](https://github.com/shaoyuan1943/AMRVoiceRecorder)。

