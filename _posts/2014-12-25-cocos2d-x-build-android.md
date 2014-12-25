---
layout: post
title: "cocos2d-x 3.x版本Android编译配置"
categories: cocos2d-x
---

3.x版本与2.x有很大的不同，这里暂且不谈代码层面上的变化，这里记录一下3.x版本在Android下的编译。2.x版本中，Android的编译是通过shell脚本进行的(build\_native.sh)，但是在3.x中修改成了python文件(build\_native.py)。我认为将3.x的编译脚本改成py可以说是cocos的有一大进步。sh配置方式太难以找错了，而脚本py方式很容易找错，通过查看代码可以知道编译经过几个步骤，每个步骤有哪些操作等。在便捷性和复杂度中间得到了平衡。

因为改成了python脚本，所以编译过程中我们实际需要注意几个环境变量：  
1. ANDROID\_SDK\_ROOT  
2. NDK\_ROOT  
3. NDK\_TOOLCHAIN\_VERSION  

只要把这几个配置好了编译过程基本没有问题，即使有问题也是很好找出来的。py脚本中大概有几个重要的函数，这里做下笔记：  
1. get\_num\_of\_cpu()，获取当前机器上的CPU核心数。  
2. check\_environmen\_variables_sdk()，检查ANDROID\_SDK\_ROOT这个环境变量是否可用。   
3. check\_environment\_variables()，检查 NDK\_ROOT这个环境变量是否可用。(我发现这两个名字取得很不明了)   
4. select\_toolchain\_version()，获取toolchain的版本号。   
5. copy\_files(src, dst)，拷贝文件。  
6. copy\_resources(app_android_root)，拷贝资源文件。  
7. build(ndk\_build\_param,android\_platform,build\_mode)，编译函数，这里调用do\_build()，真正的编译工作在do\_build里做。  

Android.mk文件的配置

1. 由于cocos2d-x是C++写的，所以需要将源代码暴露给NDK。  
2. 配置游戏所需链接的库。  

![alt text](/img/2014-12-25.png)