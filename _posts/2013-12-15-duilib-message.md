---
layout: post
title: "duilib里的消息流转"
categories: c++
---

Duilib中比较重要的一个方面就是窗口消息的流转，首先我们不谈渲染引擎。只针对三个类CWindowWnd，WindowImplBase，CPaintManagerUI。

首先我们确定一下几个消息的顺序WM_CREATE，WM_PAINT，
WM_CREATE消息是在调用了CreateWindowEx之后发送给窗口过程的，说明窗口已经创建完成了，可以再OnCreate函数里做其他操作或者创建子窗口。
在Create完成之后接下来我们调用ShowWindow来告诉windows要显示哪一个窗口，但是此时窗口并没有显示出来，需要手动调用UpdateWindow才能将窗口显示出来，最终将窗口显示出来了。UpdateWindow产生了WM_PAINT消息，这个消息发送到窗口过程。

接下来说明这三个类。CWindowWnd是窗口类的第一层封装，WindowImplBase是窗口类的第二层封装，继承自CWindowWnd，CPaintManagerUI是虚拟窗口构建器。
假设有：  

``` c++
class CFrame : public WindowImplBase
{};
CFrame pFrame = new CFrame();
pFrame->Create(...);  
::ShowWindow(...);
CPaintManagerUI::MessageLoop();
```

在pFrame->Create中，调用的实际上是底层CWindowWnd中的Create创建了一个逻辑窗口。Create逻辑窗口之后产生了WM_CREATE消息，发送到了底层的窗口过程__WndProc函数，但是__WndProc没有处理，而是通过虚函数HandlerMessage发送到了上一层，即WindowImplBase中，WindowImplBase中的HandlerMessage抓到了WM_CREATE消息，出发了OnCreate函数，在OnCreate函数中，调用了CDialogBuilder::Create创建逻辑控件（即加载xml文件的描述，此时WM_PAINT消息还没有来）。  

然后ShowWindow将这个窗口显示出来，此函数产生了WM_PAINT消息，是在CPaintManagerUI::MessageLoop中，MessageLoop将消息一层一层发送。同样由虚函数HandlerMessage将WM_PAINT消息送到了WindowImplBase中，WindowImplBase中的HandleMessage抓到了 WM_PAINT 消息，但是此消息又发送给了m_PaintManager.MessageHandler，然后在CPaintManagerUI中的MessageHandler中处理WM_PAINT消息，于是接下来开始绘图，渲染引擎开始工作，渲染引擎完成了渲染工作之后将绘制完成的图显示屏幕上，这样主窗口就开始显示出来了（由于逻辑窗口没有调用UpdateWindow所以逻辑窗口没有显示，正是DirectUI的巧妙处理的地方，但是即使显示出来了也看不到，因为引擎渲染出来的图始终覆盖在逻辑窗口之上的）。

这样就可以知道：  

自定义窗口中的HandleMessage是来处理我们应用所要处理的消息。
CPaintManagerUI中的MessageHandler就是DUILIB 库帮我们处理的消息及相关的处理函数，同时出发了渲染引擎进行窗口的绘制。
CWindowWnd::HandleMessage则将不处理的消息交给最下面（实际上也就是windows）进行处理。