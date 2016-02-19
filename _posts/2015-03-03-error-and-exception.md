---
layout: post
title: "C++中的错误与异常处理"
date:   2015-03-03
categories: Program-Languages
---

* content
{:toc}

### 前言

最近在优化萌仙客户端的代码，萌仙的C++代码有一个特点就是函数以返回值标识是否调用至理想结果，参数则分为输入参数与输出参数，且C++代码中没有使用异常机制。这样做的好处是C++代码很简单，坏处是有了问题无法简单调试，程序堆栈不明显。

正好我最近在着手DuiLib 2.0的计划，DuiLib 1.0对错误与异常的处理方式比较少，甚至连错误码也没有特定的处理，所以想在DuiLib 2.0中加上对异常的支持。前段时间看了Pongba的文章，Pongba的文章很不错，文笔独到，见解犀利。有之启发，所以根据自己积累的经验理解错误与异常处理。

### 什么是错误

>调用方违反被调用函数的precondition、或一个函数无法维持其理应维持的invariants、或一个函数无法满足它所调用的其它函数的precondition、或一个函数无法保证其退出时的postcondition；以上所有情况都属于错误。

如下面的代码：

    FILE* file = fopen(strFileName.c_str(), "rb");
    UCHAR fileHead[9] = { 0 };
    fread(fileHead, 1, 9, file);
    fclose(file);

上面的代码中```fread```函数的precondition是```fopen```正常被调用，一旦行为无法正常进行，```fread```函数调用出错，条件将得不到满足，这就是错误。在如下面的代码：  

    FILE* file = fopen(strFileName.c_str(), "rb");
    UCHAR fileHead[9] = { 0 };
    fread(fileHead, 1, 8, file);
    fclose(file);
    // ...
    DoOtherthings(fileHead);

上面的代码中，定义了长度为9的数组，但是只读取了文件的前8个字节，然后调用```DoOtherthings(fileHead)```，可能出现的情况是```DoOtherthings()```得不到条件满足，无法得到理想结果。这里出现错误的原因是```fileHead```只读取了8个字节，而实际上程序是按照9个字节进行处理的，这里属于程序不严谨，属于bug。

Bug是否属于错误呢？引Pongba的话：  
>编程bug不属于错误。

Bug属于程序员在编码期间人为造成的。但是这里应该分情况而定，如果```DoOtherthings()```属于第三方提供的函数，那么这里理应抛出异常作为错误处理，否则应该当场debug。所以程序不严谨导致的bug不是错误，即使因为这而给它做了异常处理，也需要考量一下，错误应该是程序无法规范的行为，而这里显然程序是可以规范的，bug应该被消灭在编码、review和QA阶段。
  
### 如何处理

> 严谨的错误处理要求不要忽视和放过任何一个错误，要么当即处理，要么转发给调用者，层层往上传播。任何被忽视的错误，都迟早会在代码接下去的执行流当中引发其他错误，这种被原始错误引发的二阶三阶错误可能看上去跟root cause一点关系都没有，造成bugfix的成本剧增，这是项目快速的开发步调下所承受不起的成本。

错误是在程序发布之前不能忽视的。这意味着需要不断的仔细的检查代码与QA，力争做到解决全部错误，很遗憾，这点我们现在做不到。这是认为因素决定，一定人为影响因子降低，付出的代价将会在以后有更大的代价。

检查错误意味着我们需要不断的review代码，这种“不断的”是需要付出时间与人力成本的，且在代码层面需要人为的处理与转发错误。这样造成的后果是代码膨胀与混乱。正如Pongba所言，使用异常来作为错误的处理机制。

除了上面所提出的使用异常处理机制来处理错误，现实程序中还有另外一种处理错误的方式，错误码。这也就是萌仙C++代码中的使用方式。由于cocos2d-x中的FileUtils不够用，因此我们针对我们自己的文件系统重写设计了一个FileAssist类：  

    struct XFileAssist : public XFileHelper
    {
    	virtual BOOL GetDirList(std::list<std::string>& dirList, 
    							const char szDir[], 
    							BOOL bRecursion = false, 
    							BOOL bRetRelativePath = false);
    
    	virtual BOOL GetFileList(std::list<std::string>& fileList, 
    							 const char szDir[], 
    							 BOOL bRecursion = false, 
    							 BOOL bRetRelativePath = false);
    
    	virtual CODE ReadFileData(BYTE *pbOutput, 
    							  size_t* puSize, 
    							  const char szFileName[], 
    							  size_t uExtSize = 0);
    	// ...
    }

FileAssist类的设计中，表明了函数是否调用至理想结果以及对错误返回码的处理。```GetDirList```和```GetFileList```以返回值表明函数是否调用至理想结果。```ReadFileData```因为考虑在获取文件数据过程中，因为会对数据进行较多的处理，所以以返回码来标识函数的调用状态。但是有几个问题：  

1. 函数的返回值被占用。    
2. 以```GetDirList```为例，是以dirList还是以返回值作为成功依据？我想这会根据写代码的人不同而产生歧义，虽然这两种都可以作为成功依据。  
3. CODE有多个值，QA和review的人都需要知道CODE下每个值分别是什么意思？  
4. CODE是否会向调用方传递？调用方接收到CODE后是否会再向上一层调用方传递？每一层都需要传递处理，造成调用层次高耦合。  
5. 无论CODE返回什么东西，我们都无法忽略CODE。  
6. CODE能够承载的程序错误信息有限，万一出现CODE无法描述的情况怎么办？  


以上的问题都是在程序过程中可能出现的问题，且可能会依赖代码的膨胀而膨胀。简而言之，我们需要对错误的处理方式是：  

1. 函数签名足够简单，意义要明确。  
2. 尽量少的嵌套与上层传递处理。
3. 具有程序可恢复性，即使在无法恢复的时候可以获得更多的Error Info。  

  
### 异常处理机制的弊端

异常就是一把双刃剑，有好处当然也会有坏处。全面使用异常处理当然也需要更高的技能。异常处理机制要求RAII范式，如果强烈要求是异常安全的代码，需要大量的额外机制去配合。

使用异常要意味要正确决定异常抛出点与捕获点，这在项目里也会作为成本存在。过多且不合适的异常抛出点反而会造成代码混乱。异常处理只用作错误，一旦项目庞大了之后，还有一种可能出现的情况就是用异常做正常的跳转！

异常处理的另外一大难题就是资源。如下面的代码：

    try
    {
    	A *pA = new A;
    	// ...
    	DoThings(pA);
    	// ...
    	delete pA;
    }
    catch(...)
    {
    	// ...
    }

上面的伪代码中，在try块中new了一个对象传递给```DoThings```做其他事情，一旦```DoThings```抛出了异常并且被catch块捕获到了，有两种情况：

1. 无法恢复的错误，接下来便在catch块中拦下dump core文件便于开发人员调试，然后会调用abort终止程序，pA所指对象会跟随程序的终止而回收。
2. 如果是可以恢复的错误，在catch块中被恢复了，然后程序继续下去。这时候会发生什么？内存泄漏！因为```delete pA;```根本执行不到。同理，打开文件，网络socket链接，数据库连接等，都有可能发生异常，一旦抛出异常后就需要对失败后的资源占用进行处理，否则，后果不预知。