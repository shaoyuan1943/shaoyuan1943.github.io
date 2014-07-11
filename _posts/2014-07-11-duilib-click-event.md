---
layout: post
title: "DuiLib中事件分开处理的一个方法"
categories: c++
---

这两天帮一个童鞋做PC客户端的界面，这个童鞋对客户端界面不太熟悉，又想快点搞定。于是我就想到了用DuiLib帮他做。

好久没有碰DuiLib了，也不知道DuiLib现在发展什么样子了，作者已经很久没有更新了，反正我是改了一套DuiLib自己用的爽，顺便把改了之后的DuiLib的相关东西放到github上去了。

DuiLib里有一个很蛋疼的就是，窗口消息都会分发到窗口中```void Notify(DuiLib::TNotify& msg);```函数中，某一个按钮的事件反应这么写的：  

``` c++
void MainWindow::Notify(DuiLib::TNotify& msg)
{
	if(msg.sType == L"click")
	{
		if(msg.pSender->GetName() == L"sys_btn_close")
		{
			// to do anything...
		}
	}
	else if(msg.sType == L"windowinit")
	{
		// to do anything...
	}
}
```

可以想象如果事件多的话按照上面的方式这个```Notify(TNotify& msg)```将会有多长，即使在```Notify(TNotify& msg)```中做消息转发也需要非常多的```if...else...```，这是非常蛋疼的，今天再次用到的时候就想着能不能有简单的办法将处理消息转发出去(以前都是通过类型和控件名转发到真正处理的函数中)，我是这么干的，伪代码如下：   

``` c++
// 头文件中
typedef std::function<void(DuiLib::TNotifyUI& msg)> ClickEvent;

// 方便操作的宏
#define MakeClickEventMap(eventmap, name, event) \
	eventmap.insert(std::pair<std::wstring, ClickEvent>(name, event))

// 窗口类的变量
std::map<std::wstring, ClickEvent> EventMap;

// 在窗口的InitWindow函数中初始化这个事件列表
MakeClickEventMap(this->EventMap, L"sys_btn_close", std::bind(&MainWindow::CloseWindow, this, std::tr1::placeholders::_1));

void MainWindow::Notify(DuiLib::TNotify& msg)
{
	if(msg.sType == L"click")
	{
		wstring name = msg.pSender->GetName();
		EventMap[name](msg);
	}
}
```  

一般来说，程序里需要处理的最多就是```click```消息，然后还有```windowinit```消息，这里就可以单独为消息类型定义事件```map```，反正也不多，这样整个```Notify(TNotify& msg)```函数就可以大大减小了。

这里只是试了一种能够大幅度减少```Notify(TNotify& msg)```代码的方法，同时用到C++ 11特性，做起来我觉得还是蛮漂亮的。