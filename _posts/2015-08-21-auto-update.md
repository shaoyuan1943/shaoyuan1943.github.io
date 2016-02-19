---
layout: post
title: "重构更新机制"
date:   2015-08-21
categories: Game-Dev
---

* content
{:toc}

### 前言

在决定开始优化萌仙的时候，重构更新机制被放在最重要的地位，因为这套更新机制存在的麻烦远远大于好处。不支持版本全量更新，不支持非强制性更新，更新包以滚雪球的方式越滚越大，以至于到最后一个普通的更新包居然有40MB之多。想要在国外上线，这是必须要干掉的大麻烦之一。

增量更新：每次版本更新至更新当前版本内的修改内容。  
全量更新：以App版本为基点，更新所有已发布的全部版本。  
AppVersion：当前客户端安装包的版本号，即主版本。   
ResVersion：当前资源的版本号，即我们定义的日常更新版本，脚本、配置表、CCB、图片、音效等都定义为资源，即ResVersion。

### 方式

理想中的更新方式：  

	1、1.0 App版本
	2、1.0.1日常更新版本
	3、1.0.2日常更新版本
	4、1.0.3日常更新版本
	5、2.0 App版本
	6、2.0.1日常更新版本
	7、2.0.2日常更新版本
	8、2.0.3日常更新版本

上面1.0是AppVersion，1.0.1是ResVersion，1.0.1是保持主版本不变的情况下仅改动了相关游戏资源。实际上1.0版本是包括了打包时资源，因此1.0.1版本中的内容实际上就是与1.0版本中的资源作比较得出的差异，然后将这个差异更新到客户端。

那么我们需要：  
一个版本号配置文件update.ini，这个文件会在每次版本时修改，主要是记录当前客户端的ResVersion。对于这个文件，看起来应该像这样：  

	[Versions]
	AppVersion=1.0
	ResVersion=1.0.1

那么以正常流程来看，客户端的更新流程：  
![alt text](/img/2015-08-21-1.png)  

而更新包的流程大致如下：  
![alt text](/img/2015-08-21-2.png)  

这是简化后的流程，就客户端而言，复杂度降低了很多。

### 实现

#### 版本服务器

既然是更新，必然要涉及到版本服务器。我们对版本服务器的要求：  
1. 简单易用，不能过于复杂。  
2. 既能管理AppVersion又能管理ResVersion。  
3. 要适用于不同的游戏。  
4. 既能方便运维操作也要方便运营操作。  
5. 需要配备GUI操作后台。

综合上述需求，版本服务器的技术选型过程中我们采用了Java语言实现。没有太复杂的设计，简单但可靠的代码。

与常规的版本服务器不同的是：在进行版本管理时（增），我们并没有选择去比较所填入的版本号，而是以管理的时间为标记。当添加一个ResVersion时，我们此时并不检验这个ResVersion的正确性，仅检查这个ResVersion是否重复，然后根据添加这个版本的时间定义为一个版本。

客户端在请求服务端进行版本验证时，以客户端发过来的版本为基点，得到客户端当前版本发布的时间，然后以这个时间与当前时间做检查，检查这段时间内的版本，然后将版本的下载链接发给客户端，这样完成一次版本检查，接下来的全部工作交由客户端。

之所以不比较版本号，因为比较版本号并非最安全的操作，因为客户端甚至可以伪造一个版本号，版本发布的时间是留在版本服务器内的，所以我们以时间为检查标准。这样将版本号的配置交给人来进行，即使不小心配置错了也没关系，再添加一个ResVersion即可，丝毫不影响客户端的逻辑。版本号仅仅作为一个标记，是否有最新的版本是通过版本号对应的版本发布时间来决定的。

在这一点上它带来的优势远大于所带来的代价，且代价不会影响客户端。

逻辑：  
1. 客户端通过json协议与版本服务器交互检查版本。  
2. 客户端得到版本服务器返回的json串得到版本的下载地址。  
3. 维护一个下载队列，下载版本包。  
4. 解压版本包重新加载即可。  

在客户端上，我抛弃了cocos2dx里的AssetManager选择重写。cocos2dx::AssetsManager有一个致命的缺点：它开始更新与结束都是以UserDefault.xml中的变量来标记的，甚至把版本号也记录在这个文件里。这种做法非常不明智，一旦在更新过程中出错导致UserDefault.xml记录失败，意味着这个玩家在下一次更新时要付出上一次更新失败的修复成本。

所以新实现的AssetsManagerScene仅仅只继承了CCScene，然后通过scheduleUpdate去轮询当前更新检查的状态，大致代码实现：  

    void XAssetsManagerScene::update(float delta)
    {
    	CCNode::update(delta);
    	switch (_euState)
    	{
    		case eInit:
    		{
    			_strStorePath = CCFileUtils::sharedFileUtils()->getWritablePath();
    			Log(eLogDebug, "WritablePath:%s", _strStorePath.c_str());
    			_pProgressTipLabel->setString("wait...");
    			_pProgressLabel->setString("");
    			_euState = eCheck;
    			break;
    		}
    		case eCheck:
    		{
    			_euState = eWait;
    			checkUpdate();
    			break;
    		}
    		case eCheckResponse:
    		{
    			parseVersionResponse();
    			break;
    		}
    		case eNoForcedUpdate:
    		{
    			_euState = eWait;
    			MsgBoxBtn btnOK(_pConfig->getComfirmNotify().c_str(), this, callfunc_selector(XAssetsManagerScene::onConfirmForceUpdate));
    			MsgBoxBtn btnCancel(_pConfig->getCancelNotify().c_str(), this, callfunc_selector(XAssetsManagerScene::onCancelFroceUpdate));
    			XEnvironment::NotifyMsgBox(_pConfig->getUpdateNotify().c_str(), _pConfig->getForceUpdateNotify().c_str(), &btnCancel, &btnOK);
    			break;
    		}
    		case eStart:
    		{
    			download();
    			break;
    		}
    		case eDownloading:
    		{
    			char szBuffer[64];
    			snprintf(szBuffer, _countof(szBuffer), _pConfig->getDownloadTip().c_str(), _iCurrentDownIndex, _versionInfoMap.size());
    			_pProgressTipLabel->setString(szBuffer);
    
    			memset(szBuffer, 0, _countof(szBuffer));
    			snprintf(szBuffer, _countof(szBuffer), "%d %%", _iPercent);
    			_pProgressLabel->setString(szBuffer);
    			break;
    		}
    		case eDownloadCompleted:
    		{
    			uncompress();
    			break;
    		}
    		case eUncompressing:
    		{
    			_pProgressLabel->setString("");
    			_pProgressTipLabel->setString("uncompressing...");
    			break;
    		}
    		case eEnd:
    		{
    			onEnterGameScene();
    			break;
    		}
    		case eWait:
    			break;
    	default:
    		dealErrorMessage(_euState);
    		break;
    	}
    }

如何打更新包？

记录所有资源的MD5值，没发一次版本记录当前所有文件的MD5值，下一次做版本时取当前所有文件的MD5值与上一次版本的所有MD5值进行Compare，得到Diff差异，将差异文件以某种方式达成二进制更新包，客户端在下载完更新包之后按打包时间顺序加载。

由于项目的原因，有一些代码未能公开，有很多实现细节也未能一一详解，talk is cheap啊。有兴趣的童鞋可以一起交流。

