---
layout: post
title: "iOS上录制AAC格式音频"
date:   2016-02-23
categories: Android-iOS
---

今天搞定了iOS上录制AAC音频，并对Android和iOS上录制AAC音频做了效率测试，结果是在iOS上录制出来的AAC音频文件质量和文件大小满意，Android上录制出来的文件会比iOS上的大20kb左右。

测试前提：Android上录制AAC音频有FAAC库和VoAAC库支持，iOS则是由```AVAudioRecord```实现的。

测试结果：

* Android-FAAC：录制22s左右大小是121836字节，音频质量尚可，CPU占用最高到5%左右，一旦录制时间变长，内存占用升高。
* Android-VoAAC：录制22s左右大小是88580字节，音频质量尚可，CPU占用在2%~4%左右，内存占用明显小于1024kb。
* iOS：录制22s左右大小是68973字节，音频质量较高。

测试机器：SN3，iTouch 5  
测试结果上来看，主要差别在Android无论我怎么录制，文件大小都比iOS大20kb左右。由于Android对AAC格式的支持并不完美，而iOS上则是对AAC完美支持且在系统层面有一定的优化，我猜想差异应该由系统层面对AAC格式的编解码支持度不一样导致，好在差异在我的接受范围内，如果有谁知道AAC在Android和iOS平台上的为什么会有大小差异的话烦请告知一下。

目前Android上测试了几部高端机器，表现均在预料范围内，iOS设备测试了iTouch 5 和iPhone 5，表现都还不错。现在唯一有点担心的是Android略微低端的机器是否可以正常录制和播放AAC，还有一些ROM改动比较大的机器，比如MIUI，Flyme等，不确定运行这些ROM机器是否可以正常录制，有空了再做一次全面的测试。

接下来开始进入具体的模块设计了。。
  
### 其中使用范例在[**这里**](https://github.com/shaoyuan1943/iOSRecordAAC.git)
