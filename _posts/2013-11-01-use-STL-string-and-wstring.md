---
layout: post
title: "关于STL中string的一些使用感受"
categories: c++
---

最近写的刺猬苹果助手已经发布了，慢慢的已经有一些开始使用了，大家如果感兴趣可以私信我发客户端你们使用。这个客户端是duilib下写的，其中用到的字符串主要还是STL中的string和wstring，虽然duilib中自带了CStdString，但是在阅读CStdString源代码的过程中发现CStdString的实现略微有些不当，在使用过程中也是不甚方便。  

duilib中默认就开辟了63长度的字符空间，用的是malloc，显然这个适用于短字符。但是在free string的时候只是直接将buffer的第一位设置为空了，这里也就造成了63长度中的空间浪费。用的少还好说，但是用的多了那么CStdString显然就是一个不太好的选择，而且STL提供了大而全string模板，所以我就选择用string和wstring了。  

以STL中string和wstring作为主要字符串使用还是第一次，不得不说，用的很爽昂。曾经某些人在知乎上讨论string和wstring的效率问题，首当其冲就是string慢的问题。  

在string是以写时复制的，也就是说string内部主要操作都是以拷贝实现的，有下面一段例子：  

``` c++
string strA = "shaoyuan";  
string strB = strA;  
const char* chA = strA.c_str();  
const char* chB = strB.c_str();  
string strC = strA.replace(1,3,"xyn");  
const char* chC = strC.c_str();  
``` 

strA与strB所占内存地址不同，chA和chB的占用地址不同。也就是说strB = strA这句，实际是上是将strA中的内容拷贝到了strB的区域，这样也就解释了strA和strB在返回内存时地址不同。同样，replace操作不仅仅更改了源字符串，而且也拷贝了一份替换后的字符串作为返回，就这么几个操作string内部就有好几个拷贝操作，效率略低的原因就在这里。慢通常是因为赋值/拷贝构造和operator+这类新申请存储或改变内部存储布局的操作，这里面有new和memcpy，相对慢。但是还有一个很重要的原因就是使用者的问题，尽可能用const&传参，减少栈上的string对象创建和string对象修改。当然啦我有点比较反感的就是，有些人自己没有怎么使用就一个劲的说效率问题，即使现在大面积使用，尽可能的优化，我像效率问题还是可以稳定在接受范围之内的。  

string在使用上来的感觉就是非常方便，由于我的客户端大部分都是比较短的字符，所以在string的使用上也没有太多的估计，而且是能使用临时对象的地方都用临时对象代替了，在作为参数时也是尽可能的以&的方式传递了。唯一比较长的可能就是直接使用string开辟内存空间存放png图片的二进制数据，但是也只是作为block变量，过了有效区域直接回收了。