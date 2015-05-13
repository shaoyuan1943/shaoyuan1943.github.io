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

一旦try语句中出现任何异常，程序将会跳转至catch子句中，我们在catch子句中做代码上的善后处理，比如资源释放。

	// 1.
	delete p;	// a
	p = nullptr;	// b

	// 2.
	try
	{
		delete p;	// c
		p = nullptr;
	}
	catch(Exception& e)
	{
		// 善后处理
	}

一旦p其实已经被delete过，那么a代码将会出现内存访问异常，如不出意外，程序会abort。这在某些情况下是难以接受的，用户或许正在操作某种数据，一旦abort意味着用户数据将会丢失，损失是惨重的。有了try...catch语句之后，意味着一旦程序abort将会有一个专门的地方留给我们做善后操作，资源施放、数据落盘等。

说明：实际上对于p还有更好的处理。

在C++代码中，防御式的编程有更明显的体现，在很多开源库中都会见到不少的do...while的写法，cocos2d-x中对于do...while的使用非常多。do...while写法在防御式编程中也是一个常用的手段。

	bool Fuck()
	{
		bool bRet = false;
		int *pAssHole = new int;
	
		bRet = fuck1();
		if(!bRet) 
		{	
			delete pAssHole;
			return bRet;
		}
	
		bRet = fuck2();
		if(!bRet) 
		{	
			delete pAssHole;
			return bRet;
		}
	
		bRet = fuck3();
		if(!bRet) 
		{	
			delete pAssHole;
			return bRet;
		}
	}

上面的代码冗余，非常不灵活，这样的代码往往隐藏着错误。如果我们采用do...while的写法：  

	bool Fuck()
	{
		bool bRet = false;
		int *pAssHole = new int;
	
		do
		{
			bRet = fuck1();
			if(!bRet) 
				break;
		
			bRet = fuck2();
			if(!bRet) 
				break;
		
			bRet = fuck3();
			if(!bRet) 
				break;
		}while(0)
	
		delete pAssHole;
	}

我们明白do...while(0)肯定会执行一次，一旦某个if不成立，只需break跳出做其他处理即可。很大程序上减少代码冗余，提高程序健壮性。

	Sprite* sprite = new Sprite;
	CC_SAFE_DELETE(sprite);

```CC_SAFE_DELETE```的实现：  

	#define CC_SAFE_DELETE(p) do { if(p) { delete (p); (p) = 0; } } while(0)

也是采用了do...while手法。但是这种手法因人而已，总的来说，它是非常好的手段。这里的do...while有啥意义呢？  
	
	// a.
	#define SAFE_DELETE(p) delete p; p = NULL;
	if(nullptr != p) 
		SAFE_DELETE(p)
	else
		// 想干嘛干嘛
	
	// b.
	#define SAFE_DELETE(p) {delete p; p = NULL;}
	if(nullptr != p) 
		SAFE_DELETE(p);
	else
		// 想干嘛干嘛

如果不适用do...while，那么显然a代码无法通过编译，因为else无对应的if语句。b代码同样无法通过编译(加分号的习惯很严重:))。

当然，每个block之前强制加上```{}```可以解决这个问题，这就是仁者见仁智者见智的问题了。



