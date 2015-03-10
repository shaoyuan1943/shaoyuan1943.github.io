---
layout: post
title: "C++中的错误与异常处理"
categories: c++
---

最近在优化萌仙客户端的代码，萌仙的C++代码有一个特点就是函数以返回值标识是否调用至理想结果，参数则分为输入参数与输出参数，且C++代码中没有使用异常机制。这样做的好处是C++代码很简单，坏处是有了问题无法简单调试，程序堆栈不明显。

正好我最近在着手DuiLib 2.0的计划，DuiLib 1.0对错误与异常的处理方式比较少，甚至连错误码也没有特定的处理，所以想在DuiLib 2.0中加上对异常的支持。前段时间看了Pongba的文章，Pongba的文章很不错，文笔独到，见解犀利。有之启发，所以根据自己积累的经验理解错误与异常处理。

###什么是错误###
>调用方违反被调用函数的precondition、或一个函数无法维持其理应维持的invariants、或一个函数无法满足它所调用的其它函数的precondition、或一个函数无法保证其退出时的postcondition；以上所有情况都属于错误。

如下面的代码：

``` c++
FILE* file = fopen(strFileName.c_str(), "rb");
UCHAR fileHead[9] = { 0 };
fread(fileHead, 1, 9, file);
fclose(file);
```

上面的代码中```fread```函数的precondition是```fopen```正常被调用，一旦行为无法正常进行，```fread```函数调用出错，条件将得不到满足，这就是错误。在如下面的代码：  

``` c++
FILE* file = fopen(strFileName.c_str(), "rb");
UCHAR fileHead[9] = { 0 };
fread(fileHead, 1, 8, file);
fclose(file);
//
DoOtherthings(fileHead);
```

上面的代码中，定义了长度为9的数组，但是只读取了文件的前8个字节，然后调用```DoOtherthings(fileHead)```，可能出现的情况是```DoOtherthings()```得不到条件满足，无法得到理想结果。这里出现错误的原因是```fileHead```只读取了8个字节，而实际上程序是按照9个字节进行处理的，这里属于程序不严谨，属于bug。

Bug是否属于错误呢？引Pongba的话：  
>编程bug不属于错误。

Bug属于程序员在编码期间人为造成的。但是这里应该分情况而定，如果```DoOtherthings()```属于第三方提供的函数，那么这里理应抛出异常作为错误处理，否则应该当场debug。所以程序不严谨导致的bug不是错误，即使因为这而给它做了异常处理，也需要考量一下，错误应该是程序无法规范的行为，而这里显然程序是可以规范的，bug应该被消灭在编码、review和QA阶段。