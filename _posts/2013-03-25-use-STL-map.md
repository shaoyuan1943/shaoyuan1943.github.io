---
layout: post
title: "使用std::map发现的一个容易忽略的小问题"
categories: c++
---

最近在使用的std::map的过程中发现了一个小问题，折腾了半天之后才发现自己忽略了一个地方。相信也有人遇到和我一样的情况，在这里做个记号，将问题和解决办法和大家共享一下。这里有一个函数，暂且叫做getMap()，是Doc类中的一个类成员函数。getMap返回一个std::map，具体的类型定义如下：

``` c++
std::map<const char*,const char*>map_;  
```

此map有一堆键值：```family_seniority=shen```;键值的类型都是```const char*```。于是我在main函数里试图调用getMap取到这个map，进而取到这个map里面的family_seniority键的值。但是，就在这里问题来了，运行之后我的程序崩溃了，然后单步调试请看下面的图。  
![alt text](/img/2013-03-25-1.jpg)

事实证明我想要的这个std::map已经成功返回了，而且还带入了我想要的值，接下来我试图把这个index为0的一对键值取出来： 

``` c++ 
const char* strss = map_["family_seniority"];
```
  
然后我 ```std::cout << strss << std::endl;```尝试输出，结果程序崩溃了，于是我把断点打在这句输出的代码上，调试的结果大吃一惊：  
![alt text](/img/2013-03-25-2.jpg)

怎么map里面有两对键值了，怎么会这样？我一下子蒙了，我没有做任何操作啊。我在调用getMap()方法之后获得map之后就立即```const char* strss = map_["family_seniority"];```有了这句，再下面一句就是输出，应该就是这三句之间的某句代码有问题了。我尝试进入```getMap()```函数，发现map返回的时候也是只有一对键值，那么问题肯定就是出现在:  

``` c++
const char* strss = map_["family_seniority"];  
```

这句代码上。但是我想了之后，这只是一个简单的取键值得操作，怎么会出现这个问题，但是我查看了文档之后才发现，当使用```operator[]```操作取键值时会自动在整个map里查找一次，如果发现了有family_seniority这个键，就返回这个键相对应的值，如果没有则在map里插入这个键。下面是重载[]符号的源代码：  

``` c++
T& operator [] (const key_type& k)  
{  
    return (*((insert(value_type(k,T()))).first)).second;  
}  
```

也就是说，一旦使用了[]符号，就会调用此重载，调用insert语句，键就是插进去的键，而键值则是一个T类型的临时对象，这样也就解释了family_seniority这个键被添加进去了，但是键值是不存在的，因为char()这个临时对象是未知的。这样也就解释了```const char* strss = map_["family_seniority"];```这句之后为什么会有两对键值了。  

问题又来了，我的map里明明是有这个family_seniority键的呀，怎么还会插入一条呢？于是我再检查了下我的map定义：```std::map<const char*,const char*>map_;```恍然大悟，我的模板参数是const ```char*```,是指针。也就是说实际上在map里面把指针地址作为了键，并没有把指针所指的内容family_seniority作为键。从第二张图看到，两个键的地址为```0x000eb3c8,0x00cb1dd8```，这根本就是两个不同的地址，再```operator[]```内的查找过程中，```0x000eb3c8 != 0x00cb1dd8```。这样问题的原因也就找到了，由于是将字符串指针作为了地址，因此当使用```operator[]```取```family-seniority```值，只是在map里查找地址，发现没有所要取的地址，于是就插入了一条，这样也就导致了strss为```0x00000000```。  

再往更深层次追求就发现，上面的这个理由略微有点牵强，为什么呢？因为我们知道，map是一个模板类，这个模板类有四个参数，请看map的申明：  

``` c++
template<class _Kty,  
    class _Ty,  
    class _Pr = less<_Kty>,  
    class _Alloc = allocator<pair<const _Kty, _Ty> > >  
    class map  
        : public _Tree<_Tmap_traits<_Kty, _Ty, _Pr, _Alloc, false> >  
    {   // ordered red-black tree of {key, mapped} values, unique keys  
    };  
```

其中第三个参数_Pr默认为less。map实际上是有一个排序功能的，这个排序默认为小于排列，也就是根据键从小到大排列，```class _Pr = less<_Key>```则说明map是默认小于排列的，为了验证，我写了一个测试东东：  

``` c++
std::map<int ,int> map_int_;  
map_int_[2] = 2;  
map_int_[1] = 1;  
map_int_[3] = 3;
```
  
上面的代码上，我是按照2、1、3这个顺序往map中插入的，然后调试：   
![alt text](/img/2013-03-25-3.jpg) 

发现键的顺序已经是从小到大排列了，也就说说明map确实是默认小于排列的。但是这仅仅都是针对那些可以比较的数据类型，虽然指针也是可比较的，但是我们想比较的是指针所指向的内容，若与传入的键命相等，则取出该键的值，这样在```operator[]```过程中就涉及到一个查找一个一比较过程，而且要求比较的不能是地址，而是地址中的内容。从map的申明可以看出，_Pr是给了默认值，在map内部有：  

``` c++
typedef _Pr key_compare;
typedef _Tree<_Tmap_traits<_Kty, _Ty, _Pr, _Alloc, false> > _Mybase;
```

再看map中的函数：  

``` c++
		map()  
        : _Mybase(key_compare(), allocator_type())  
        {   // construct empty map from defaults  
        }  
  
    map(const _Myt& _Right)  
        : _Mybase(_Right)  
        {   // construct map by copying _Right  
        }  
  
    explicit map(const key_compare& _Pred)  
        : _Mybase(_Pred, allocator_type())  
        {   // construct empty map from comparator  
        }  
  
    map(const key_compare& _Pred, const allocator_type& _Al)  
        : _Mybase(_Pred, _Al)  
        {   // construct empty map from comparator and allocator  
        }  
  
    template<class _Iter>  
        map(_Iter _First, _Iter _Last)  
        : _Mybase(key_compare(), allocator_type())  
        {   // construct map from [_First, _Last), defaults  
        this->insert(_First, _Last);  
        }  
  
    template<class _Iter>  
        map(_Iter _First, _Iter _Last,  
            const key_compare& _Pred)  
        : _Mybase(_Pred, allocator_type())  
        {   // construct map from [_First, _Last), comparator  
        this->insert(_First, _Last);  
        }  
  
    template<class _Iter>  
        map(_Iter _First, _Iter _Last,  
            const key_compare& _Pred, const allocator_type& _Al)  
        : _Mybase(_Pred, _Al)  
        {   // construct map from [_First, _Last), comparator, and allocator  
        this->insert(_First, _Last);  
        }  
  
    _Myt& operator=(const _Myt& _Right)  
        {   // assign by copying _Right  
        _Mybase::operator=(_Right);  
        return (*this);  
        }  
  
    map(_Myt&& _Right)  
        : _Mybase(_STD move(_Right))  
        {   // construct map by moving _Right  
        }  
  
    _Myt& operator=(_Myt&& _Right)  
        {   // assign by moving _Right  
        _Mybase::operator=(_STD move(_Right));  
        return (*this);  
        }  
  
    mapped_type& operator[](key_type&& _Keyval)  
        {   // find element matching _Keyval or insert with default mapped  
        iterator _Where = this->lower_bound(_Keyval);  
        if (_Where == this->end()  
            || this->comp(_Keyval, this->_Key(_Where._Mynode())))  
            _Where = this->insert(_Where,  
                _STD pair<key_type, mapped_type>(  
                    _STD move(_Keyval),  
                    mapped_type()));  
        return ((*_Where).second);  
        }  
  
    void swap(_Myt& _Right)  
        {   // exchange contents with non-movable _Right  
        _Mybase::swap(_Right);  
        }  
  
    void swap(_Myt&& _Right)  
        {   // exchange contents with movable _Right  
        _Mybase::swap(_STD move(_Right));  
        }  
```

当map构造的时候调用了```_Tree<_Tmap_traits<_Kty, _Ty, _Pr, _Alloc, false> >```这个模板类，在operator[]中，有一句代码：  

``` c++
this->comp(_Keyval, this->_Key(_Where._Mynode()));
```

这个comp函数是```_Tree<_Tmap_traits<_Kty, _Ty, _Pr, _Alloc, false> >```这个模板类中的，于是找到这个模板类的代码，代码比较短，我就一次性全贴出来:  

``` c++
template<class _Kty, // key type  
    class _Ty,  // mapped type  
    class _Pr,  // comparator predicate type  
    class _Alloc,   // actual allocator type (should be value allocator)  
    bool _Mfl>   // true if multiple equivalent keys are permitted  
    class _Tmap_traits  
        : public _Container_base  
    {   // traits required to make _Tree behave like a map  
public:  
    typedef pair<_Kty, _Ty> _Val_type;  
    typedef _Kty key_type;  
    typedef pair<const _Kty, _Ty> value_type;  
    typedef _Pr key_compare;  
  
    typedef typename _Alloc::template rebind<value_type>::other  
        allocator_type;  
  
    enum  
        {   // make multi parameter visible as an enum constant  
        _Multi = _Mfl};  
  
    _Tmap_traits()  
        : comp()  
        {   // construct with default comparator  
        }  
  
    _Tmap_traits(_Pr _Parg)  
        : comp(_Parg)  
        {   // construct with specified comparator  
        }  
  
    class value_compare  
        : public binary_function<value_type, value_type, bool>  
        {   // functor for comparing two element values  
        friend class _Tmap_traits<_Kty, _Ty, _Pr, _Alloc, _Mfl>;  
  
    public:  
        bool operator()(const value_type& _Left,  
            const value_type& _Right) const  
            {   // test if _Left precedes _Right by comparing just keys  
            return (comp(_Left.first, _Right.first));  
            }  
  
        value_compare(key_compare _Pred)  
            : comp(_Pred)  
            {   // construct with specified predicate  
            }  
  
    protected:  
        key_compare comp;   // the comparator predicate for keys  
        };  
  
    template<class _Ty1,  
        class _Ty2>  
        static const _Kty& _Kfn(const _STD pair<_Ty1, _Ty2>& _Val)  
        {   // extract key from element value  
        return (_Val.first);  
        }  
  
    _Pr comp;   // the comparator predicate for keys  
    };  
```

在map中传入的_Pr在模板中被typedef 了```key_compare```;然后_Pr 定义了一个comp对象（从字面上来理解就是比较），而该模板内部重载了```operator()```，这个函数内部调用了```comp()```;显然在这个comp对象中肯定也有一个对()进行重载的函数。也就说，在查找过程，传入的键与map已经存在的键进行比较的函数实际就是map的父类中的comp对象中的```operator()```重载函数，只不过这个值被默认设置为less了。  

再回来遇到的问题，问题的根本原因就是因为只是对指针做了比较，而并没有对指针的内容进行比较。但是默认的只是取了&进行比较，怎样才能实现指针内容的比较呢？可以尝试传入一个_Pr类型进去呢，在_Pr类里面进行()重载，这样不就可以调用外部的comp进行指针的内容比较了么？代码如下：  

``` c++
class sort  
{  
public:  
    bool operator()(const char* str1, const char* str2)  
    {  
        return strcmp(str1, str2) < 0;  
    }  
};  
//std::map<const char*,const char*,sort>map_;   
std::map<const char*,const char*,sort> map_ = pDoc->getMap();  
const char* strss = map_["family_seniority"];  
std::cout << strss << std::endl;  
```

OK！成功取到了family_seniority键的值！虽然不是很大的问题，但是我觉得还是一个容易忽略的地方，一旦std::map中的键位指针时，此时就要考虑默认的less能否将指针的内容取出进行比较，从而顺利取到键值，答案是不确定的。此方法还使用当键位结构体或者类时，也可以传入外部的自实现比较函数，从而达到目的。