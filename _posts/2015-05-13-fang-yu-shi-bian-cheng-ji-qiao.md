---
layout: post
title: "编程中的防御式技巧"
categories: c++
---

现代软件开发过程中，遇到各种由于代码编写不当带来的错误，如何提高代码的健壮性是串通开发始终的命题。现代软件工程中倡导防御式编程，何谓防御式编程，说通了就是在将可能出现的错误造成的影响控制在有限的范围内，提高软件的健壮性。防御即为抵挡，将可能出现的错误扼杀在摇篮里或完全控制错误造成的影响，将影响减至最小。

以异常处理为例，我们对于异常一般的处理如下：  

	try
	{
		// 想干嘛干嘛		
	}
	catch(Exception& e)
	{
		// 善后处理
	}

话题回到Lua里来，脚本语言很灵活，但这种灵活的代价是易出错。游戏中Lua脚本的存在感非常强，归于其简单易扩展弹性高。

	-- test.lua
	GlobalValue = nil;
	function SetGlobalValue(data)
		GlobalValue = data;		
	end

	function DoOtherthings()
		local times = GlobalValue.Times;
	end

上面的代码，看起来是没有问题的，代码可以跑的不错，但不是最佳。游戏中一般都需要热更新功能，以便随时可以在不停机的情况下解决错误。

一旦热更新之后，上述的代码将会有问题。如果SetGlobalValue在主程序的生命周期内只会调用一次，那么热更新之后势必要重新解释一遍test.lua文件，那么问题就来了，GlobalValue将会为nil值。接下来的逻辑如果没有判断GlobalValue是否为nil值，将会导致错误出现。正确的代码应该是：  

	-- test.lua
	GlobalValue = GlobalValue or nil;


