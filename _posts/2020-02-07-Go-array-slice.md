---
layout: post
title: "Go解密：数组、切片"
date: 2020-02-07
categories: Go
---

* content
{:toc}

最近在翻阅Go部分源代码，略有涉及到数组（array）和切片（slice）的实现，本文出自Arrays, slices (and strings): The mechanics of 'append'（https://blog.golang.org/slices） 的中文翻译版本，我并没有完全按照原文翻译，部分内容我重新做了解释以及加入个人理解，还有部分内容我做了适当的删除和简化，如果不当之处多多指正。  

<a href="#1">·介绍</a>  
<a href="#2">·数组</a>  
<a href="#3">·Slice：Slice Header</a>  
<a href="#4">·函数传递Slice</a>  
<a href="#5">·Slice 指针：方法接收器</a>  
<a href="#6">·make</a>  
<a href="#7">·copy</a>  
<a href="#8">·append：示例</a>  

#### <a name="1">介绍</a>  
编程语言中最常见的一个概念是数组，看起来是似乎很简单，但在将数组添加到编程语言时必须考虑许多问题，例如：  
1. 固定大小或可变大小？  
2. 大小是否是类型的一部分？  
3. 多维数组的模型？  
4. 空数组的意义？

这些问题的答案会影响数组是编程语言的众多特性之一还是其核心设计。  

在Go早期的开发阶段，设计数组前大约花了一年的时间来解决这些问题，关键之一是引入Slice：在一个固定大小的数组上有一个灵活且可扩展的数据结构。

#### <a name="2">数组</a>  
数组是Go中重要模块，像其他基础模块一样数组隐藏在一些可见组件之下。在谈及功能更加强大、突出的切片之前先简单说说数组。  

在Go程序中经常看不到数组，因为**数组的大小是数组类型的组成部分**，这点限制了书面表达能力。  

``` go
var buffer [256]byte
```
以上定义了数组变量```buffer```，类型是```[256]byte```，类型中描述了大小是256，从这里可以理解：```[256]byte```和```[512]byte```是不同的数组类型。  

与数组有关的数据是元素，数组在内存中的样子如下：  
``` go
buffer:byte byte byte ...... byte byte byte
```  
这个变量拥有256字节的数据，除此别无其它。可以通过下标访问其元素：```buffer[0],buffer[1]```等，如果索引值超过256访问数组元素会引起panic。  

#### <a name="3">Slice：Slice Header</a>  
恐怕这里有个疑问：Slice的应用场景是什么？只有理解Slice是什么和Slice能做什么才能准确使用。

Slice被描述为：**与Slice本身分开存储的数组的连续部分的数据结构，Slice不是数组，它描述一个数组。**

可以用如下方式定义Slice变量：  
``` go
var slice []byte = buffer[100:150]
var slice = buffer[100:150]
slice := buffer[100:150]
```  
所以Slice究竟是什么？在Go源代码目录下reflect\value.go这个文件中找到```sliceHeader```的定义：  
``` go
type sliceHeader struct {
    Data unsafe.Pointer
    Len  int
    Cap  int
}
```
于是我们可以暂且对Slice做如下的理解(伪代码，暂时忽略Cap变量)：  
``` go
slice := sliceHeader{
    Len: 50,
    Data: &buffer[100],
}
```
正如上，在数组上构建了一个Slice，同样可以在Slice上构建Slice：  
``` go
slice2 := slice[5:10]
```
根据我们对Slice的理解，slice2的范围是[5, 10)，放到原始数组上即[105, 110)，那么slice2的结构应该是这样子：  
``` go
slice2 := sliceHeader{
    Len: 5,
    Data: &buffer[105],
}
```
以上可以知道：slice和slice2仍然指向同一个底层buffer数组。  

现在尝试重新构建```slice```：重新截取一个Slice，并把新的Slice作为结果返回给原始Slice结构。  
``` go
slice = slice[5:10]
```
这种情况下，slice看起来和slice2一样，再截取一次：  
``` go
slice = slice[1:len(slice) - 1]
```
对应的sliceHeader：  
``` go
slice = sliceHeader{
    Len: 8,
    Data: &buffer[101]
}
```
可以联想到Slice的应用场景之一：**截取。**

#### <a name="4">函数传递Slice</a>
理解**Slice包含原始数组指针同时它又是一个值**这点很重要，Slice是一个包含了指针和长度的struct。

考虑如下的代码：  
``` go
func AddOneToEachElement(slice []byte) {
    for i := range slice {
        slice[i]++
    }
}

func main() {
    slice := buffer[10:20]
    for i := 0; i < len(slice); i++ {
        slice[i] = byte(i)
    }
    fmt.Println("before", slice)
    AddOneToEachElement(slice)
    fmt.Println("after", slice)
}
```
Slice的传递规则是值传递，值传递过程中拷贝的是```sliceHeader```结构，并未改变内部指针，该Slice和原Slice都指向同一个数组，当函数返回时候，原数组元素已被修改。  

``` go
func SubtractOneFromLength(slice []byte) []byte {
    slice = slice[0 : len(slice)-1]
    return slice
}

func main() {
    slice := buffer[10:20]
    for i := 0; i < len(slice); i++ {
        slice[i] = byte(i)
    }

    fmt.Println("Before: len(slice) =", len(slice))
    newSlice := SubtractOneFromLength(slice)
    fmt.Println("After:  len(slice) =", len(slice))
    fmt.Println("After:  len(newSlice) =", len(newSlice))
}
```
上面的代码意图对slice进行截取，但由于Slice是值传递，因为进入```SubtractOneFromLength```只是slice的一个拷贝值，所以前后slice的长度都不变。如果某个函数想修改Slice长度，一个可行的方法是把新的Slice作为结果参数返回。

#### <a name="5">Slice 指针：方法接收器</a>
另一个修改Slice的方法是以指针方式传递，上一节的代码可以改成这种：  
``` go
func PtrSubtractOneFromLength(slicePtr *[]byte) {
    slice := *slicePtr
    *slicePtr = slice[0 : len(slice)-1]
}

func main() {
    slice := buffer[10:20]
    for i := 0; i < len(slice); i++ {
        slice[i] = byte(i)
    }

    fmt.Println("Before: len(slice) =", len(slice))
    PtrSubtractOneFromLength(&slice)
    fmt.Println("After:  len(slice) =", len(slice))
}
```
这种方法有点累赘，多了一个临时变量做中转，对于要修改Slice的函数来说，使用指针传递也是比较常见的方式。还有一种方式：  
``` go
type path []byte

func (p *path) TruncateAtFinalSlash() {
    i := bytes.LastIndex(*p, []byte("/"))
    if i >= 0 {
        *p = (*p)[0:i]
    }
}

func (p path) ToUpper() {
    for i, b := range p {
        if 'a' <= b && b <= 'z' {
            p[i] = b + 'A' - 'a'
        }
    }
}

func main() {
    pathName := path("/usr/bin/tso")
    pathName.TruncateAtFinalSlash()
    fmt.Printf("%s\n", pathName)

    pathName1 := path("/usr/bin/tso")
    pathName1.ToUpper()
    fmt.Printf("%s\n", pathName1)
}

// output:
/usr/bin
/USR/BIN/TSO
```  
如果我们将```TruncateAtFinalSlash```改为value receiver会发现并没有改变原数组，而```ToUpper```无论是value receiver还是point receiver都会改变原数组。这也是Slice有趣的一点，Slice在函数传递中是值传递（拷贝变量值，内部指针仍旧指向原数组），若```TruncateAtFinalSlash```为value receiver，在进行```p = (p)[0:i]```操作时，p将会是一个新的Slice而不是pathName，而在```ToUpper```中，以Slice方式操作底层原数组，无论是哪种receiver都将改变原数组。  

#### <a name="6">容量</a>
正如前面所说，```sliceHeader```中还有一个```Cap```变量，这个变量存储了Slice的容量，记录数组实际使用了多少的空间，这是```Len```能达到的最大值。看看这样的代码：  
``` go
func main() {
	var array [10]int
	for i := 0; i < 10; i++ {
		array[i] = i
	}

	slice := array[6:10]
	fmt.Printf("%v, %v, %v, %p\n", slice, cap(slice), len(slice), &slice[0])

	slice = append(slice, 11)
	fmt.Printf("%v, %v, %v, %p\n", slice, cap(slice), len(slice), &slice[0])

	slice[0] = 12
	fmt.Printf("%v, %v, %v, %p\n", slice, cap(slice), len(slice), &slice[0])

	fmt.Println(array)
}

// out put
[6 7 8 9], 4, 4, 0xc00006a0d0
[6 7 8 9 11], 8, 5, 0xc00007c100
[12 7 8 9 11], 8, 5, 0xc00007c100
[0 1 2 3 4 5 6 7 8 9]
```
于是我们知道，当向Slice追加元素导致```Cap```大于```Len```时会创建一个Cap大于原数组的新数组（首元素地址不一致），并将值拷贝进新数组，之后再改变Slice元素值时改变的是新创建的数组（切断与原数组的引用关系）。

#### <a name="7">make</a>
根据Slice的定义：```Cap```限制了Slice的增长，当想增大Slice到大于本身容量时，推荐的做法是创建新的数组，然后把Slice数据拷贝到新数组。使用```make```创建一个新的数据并创建一个Slice。```make```有三个参数：Slice类型，初始长度和容量，用于存储Slice数据的数组长度，默认情况下，  
``` go
func main() {
    slice := make([]int, 10, 15)
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
    newSlice := make([]int, len(slice), 2*cap(slice))
    for i := range slice {
        newSlice[i] = slice[i]
    }
    slice = newSlice
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
}

// output
len: 10, cap: 15
len: 10, cap: 30
```
```make```创建了一个新的Slice，然后将原数据拷贝至新的Slice（所指向的数组）。

#### <a name="7">copy</a>
Go有个内建的```copy```函数，参数是两个Slice，将第二个Slice的数据拷贝到第一个Slice中：  
``` go
func main() {
    slice := make([]int, 10, 15)

    newSlice := make([]int, len(slice), 2*cap(slice))
    copy(newSlice, slice)
}
```
对于Slice的```copy```而言，有一点比较绕口：copy复制的元素数量是两个Slice中长度最小的那个，一定程度上节约效率。  

有一种常见的情况：原Slice和目的Slice出现交叉（在C++中我们常叫地址重叠），但copy操作任然能正常进行，这意味着```copy```可以用于单个Slice移动元素。  
``` go
// 向Slice指定有效位置插入元素，Slice必须有空间可以增加新的元素
func Insert(slice []int, index, value int) []int {
    // 增加1个元素的空间
    slice = slice[0:len(slice)+1]
    // 使用copy移动Slice前半部分
    copy(slice[index + 1:], slice[index:])
    // 存放新的值
    slice[index] = value

    return slice
}
```
```Insert```函数完成向Slice中插入值，值得注意的是，函数必须返回Slice（前面有奖过为什么）。

#### <a name="8">append：示例</a>
这里留一个问题：如何自实现Slice ```append```函数？