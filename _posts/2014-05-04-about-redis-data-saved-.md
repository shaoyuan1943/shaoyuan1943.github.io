---
layout: post
title: "Redis数据存储"
date:   2014-05-04
categories: Lua
---


今天在和运营那边联调web端的数据接口时，被一个问题扛了，仔细翻阅了配置文件之后才知道是怎么回事，看来对redis不熟悉真的是硬伤啊。

web那边会调用我暴露出来的网关接口，然后传输相关的数据过来，采用json交互方式。问题是：json数据过来了，然后我存redis数据库，然后我重启服务器，服务器会在重启的时候就去数据库get数据，每一次都成功save了，但是重启之后数据库里又没有这个key的数据，让我好生烦恼。

遇到问题了之后，我在想save的时候是不是会有个缓冲时间，因为redis是先将数据save到memory的，会不会有一个缓冲时间。然后我去问了别人，给出的回答是会立即存入数据库。这下否定我的猜想了，然后没办法，我去翻阅了一下redis.conf文件：

> Save the DB on disk:

> save <>seconds<> <>changes<>

> Will save the DB if both the given number of seconds and the given number of write operations against the DB occurred.In the example below the behaviour will be to save:
after 900 sec (15 min) if at least 1 key changed
after 300 sec (5 min) if at least 10 keys changed
after 60 sec if at least 10000 keys changed

> Note: you can disable saving at all commenting all the "save" lines.

> It is also possible to remove all the previously configured save
points by adding a save directive with a single empty string argument
like in the following example:

> save ""

看到这里我就明白了，将数据保存到文件中是需要配置的，然后我看了一下配置的时间和更改配置，发现配置的都是很大的，于是我尝试改小一点再测试就ok了。

最终的解决方法居然在这里，虽然我觉得应该存数据的时候有时间限制，但是没有找到证据论证，没想到在 redis.conf里有答案，大致看了一下这个配置文件，发现关于redis的大部分配置在这里都有说明，有时间了可以把这个翻译一下备以后查阅。