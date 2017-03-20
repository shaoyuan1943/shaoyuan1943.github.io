---
layout: post
title: "FaceUI-Dev 提高控件属性解析时的性能"
date:   2017-03-20
categories: Win32-UI-Dev
---

在duilib中，控件的属性解析都是以字符串进行的，控件内部会对属性的key（name）进行比较。字符串比较的效率远低于数字比较，所以在FaceUI中拒绝了以字符串比较来获取控件属性的方式，而是利用C++11中```constexpr```关键字实现编译器计算hash，从而节省在运行期进行比较或计算带来的效率问题。

在C++11标准中，新增了```constexpr```关键字。这个关键字明确表示：计算的值不会被改变并且在编译期就能得到计算结果。所以我们可以利用```constexpr```关键字在编译期间计算出属性名的hash，然后在运行期```switch-case```语句即可快速匹配出对应的属性值。

在duilib中是这样子的：  

``` c++
void Control::SetAttribute(LPCTSTR key, LPCTSTR value)
{
    if(_tcscmp(key, L"pos") == 0)
    {}

    if(_tcscmp(key, L"padding") == 0)
    {}

    if(_tcscmp(key, L"name") == 0)
    {}

    // ...
}
```
在FaceUI中是这样子的：  

``` c++
void Control::SetAttribute(LPCTSTR key, LPCTSTR value)
{
    switch(HASH(key))
    {
        case HASH_CT(L"pos"):
            break;
        case HASH_CT(L"padding"):
            break;
        case HASH_CT(L"name"):
            break;
        // ...
    }
}
```

其中```HASH_CT```就是在编译器就可以计算出对应的哈希值，正式利用了```constexpr```关键字。代码是这样子的：  

``` c++
class FACE_API CompileHash final
{
public:
    using hash_t = std::uint64_t;

    constexpr static hash_t prime = 0x2;
    constexpr static hash_t basis = 0xB;

    template<typename T>
    static std::uint64_t Hash(T str)
    {
        std::uint64_t ret{ basis };
        while (*str)
        {
            ret ^= *str;
            ret *= prime;
            str++;
        }

        return ret;
    }

    template<typename T>
    constexpr static std::uint64_t HashInCompileTime(T str, std::uint64_t last_value = basis)
    {
        return *str ? HashInCompileTime(str + 1, (*str ^ last_value) * prime) : last_value;
    }
};
#define HASH        CompileHash::Hash<const wchar_t*>
#define HASH_CT     CompileHash::HashInCompileTime<const wchar_t*>
#define HASH_A      CompileHash::Hash<const char*>
#define HASH_CT_A   CompileHash::HashInCompileTime<const char*>
```

其中```_A```是多字节版本。```constexpr```修饰```HashInCompileTime```，可以认为此时它是一个常量表达式，它将编译器就可以计算具体值。