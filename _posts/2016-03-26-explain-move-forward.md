---
layout: post
title: "详解C++11中移动语义(std::move)和完美转发(std::forward)"
date:   2016-03-26
categories: Program-Languages
---

* content
{:toc}

#### 前言
所有的手段都是为了解决已存在的问题。std::move和std::forward是C++11中的特性，是为了解决C++98/C++0x中遗留的问题，虽从理解上来看比较复杂，却是较好的解决手段。

#### 左值(lvalue)与右值(rvalue)
左值与右值的概念其实在C++0x中就有了。概括的讲，凡是能够取地址的可以称之为左值，反之称之为右值，C++中并没有对左值和右值给出明确的定义，从其解决手段来看类似上面的定义，当然我们还可以定义为：有名字的对象为左值，没有名字的对象为右值。

	class A
	{};
	A a; // a为左值，因为其有明确名字，且对a进行 &a 是合法的。
	
	void Test(A a)
	{
	  __Test(a);
	}
	Test(A()); // A() 为右值，因为A()产生一个临时对象，临时对象没有名字且无法进行 &取址操作。
	/*
	对 __Test(a); 而言，这里的a为左值，因为这里的a对调用方而言是具名的，__Test内部对a也是可以进行 &取址操作的。
	但这个a在调用完毕后很快被销毁，毕竟它只是一个临时变量。
	*/

所以从上述的伪代码中还可以知道：因为函数内的嵌套调用，上一层的右值是可以“变成”左值以完成下层调用，意即：可以接受右值的右值引用本身却是个左值。（这里为了避免误导，伪代码中没有采用 void Test(A &a) 的形式，不过下面会讲到。）

#### 移动语义(std::move)
有这样一段代码：

    class A
    {
    public:
      A() :array(new int[3]{1, 2, 3})
      {
      }
      ~A()
      {
        if(nullptr != a)
        {
          delete [] a;
        }
      }
      A(const A& a)
      {
        std::cout << "Copy Construct" << endl;
      }
      private:
        int *array{nullptr};
      };
      
      int main()
      {
        A a1;
        A a2(a1);
        return 0;
      }
    
    /*******************
    输出：
    Copy Construct
    *******************/

上面的代码咋看上去没什么问题，但这代码还有优化空间：如果实际上array所指的是一个非常大的数组，那么这种拷贝构造和销毁的花销是非常大的，在某些应用场景下甚至无法接受这种大size对象的拷贝。我们有一个理想：任何大size的对象，能否在其进行拷贝（拷贝构造、拷贝赋值操作）时仅仅只拷贝其class内部的大size对象，这样就避免了大size对象的构造和销毁的额外花销，答案是肯定的。

    class A
    {
    public:
      A() :array(new int[3]{1, 2, 3})
      {
      }
      ~A()
      {
        if(nullptr != a)
        {
          delete [] a;
        }
      }
      A(A &&a)
      {
        array = a.array;
        a.array = nullptr;
        std::cout << "Memory Copy Construct" << endl;
      }
    public:
      int *array{nullptr};
    };
    
    A GetTempObj()
    {
      A a;
      std::cout << a.array << endl;
      return a;
    }
    
    int main()
    {
      A a(GetTempObj());  // 实际上这段代码并不标准，依托编译器实现
      std::cout << a.array << endl;
      return 0;
    }
    
    /****************************
    输出：
    00BEF138
    Memory Copy Construct
    00BEF138
    ****************************/

上面的代码中我们似乎实现了仅仅只拷贝了A中array所指的内存。GetTempObj()返回了一个临时对象（注：在VS2012以上编译器会进行RVO优化，RVO优化暂时不在本文讨论之列），这个临时对象用来初始化a对象。上面的代码打印了array的地址，从结果上看，我们实现了两个对象之间在拷贝构造出现时至只拷贝其大size内存。上面的代码适用于一个左值与一个右值之间的拷贝构造，我们称之为移动拷贝构造。细思一下，我们还想要：

    A a1;
    A a2(a1); // error： 'A::A(const A &)': attempting to reference a deleted function

上面的代码无法通过编译，需要我们显示的指明拷贝构造函数（因为我们实现了移动拷贝构造，编译器认为它不需要自动生成拷贝构造函数）才能通过。虽说如此，实际代码中，我们不可能总是靠GetTempObj()返回临时对象（右值）以实现移动拷贝构造，需要一种特别的解决手段。  

在这种场景下，std::move应运而生。std::move在名字上有一定的误导性，但它**不移动任何东西**。  

新的标准认为：移动构造函数和移动拷贝函数应对左值和右值有不同的处理，而std::move就是其统一的解决手段。std::move可以将一个左值转化为右值引用以实现移动拷贝（下面会说move如何转换的），因此对于两个左值之间的拷贝，我们可以这样：

    A a1;
    A a2(std::move(a1)); // correctly

所以，实现一个具有移动拷贝构造和移动拷贝函数的类：

    class A
    {
    public:
      A() :array(new int[3]{1, 2, 3})
      {
      }
    
      ~A()
      {
        if(nullptr != a)
        {
          delete [] a;
        }
      }

      A(A &&a)
      {
      }
    
      A& operator = (A &&rhs)
      {
        return *this;
      }
    };

事实上，为了保证移动语义的传递，我们在编写移动构造函数的时候，应该总是记得使用std::move转换拥有形如堆内存、文件句柄等资源的成员为右值，这样一来，如果成员支持移动构造的话，就可以实现其移动语义。而即使成员没有移动构造函数，那么接受常量左值的构造函数版本也会轻松地实现拷贝构造（没有移动函数的话，std::move会默认进行拷贝操作）而不会引起大的问题。

现在有这样的代码：

    A Get()
    {
      return A();
    }
    
    A a(A());
    A a1(Get());

事实上，a和a1都不会出发移动函数（移动构造和移动拷贝），因为A()和Get()返回的临时对象，是一个右值。那么能否对这种临时对象施加std::move以实现调用移动函数呢？这里就涉及到了左值引用和右值引用。

##### 右值引用
左值引用与右值引用大可参照左值与右值。我们知道可以使用std::move将一个左值转换为右值引用以便调用移动函数。那么问题来了，右值引用可以使用std::move吗？答案是肯定的。具体来讲，左值引用和右值引用是可以相互绑定的，但它们遵循规则：

1. 非const左值引用只能绑定到非const左值；  
2. const左值引用可绑定到const左值、非const左值、const右值、非const右值；  
3. 非const右值引用只能绑定到非const右值，但不适用于函数模板的形参；  
4. const右值引用可绑定到const右值和非const右值，它没有现实意义（**毕竟右值引用的初衷在于移动语义，而移动就意味着修改**）；

我们想让a和a1实现移动函数的调用，从上面来上看，我们需要将A()和Get()的返回值转换成非const右值才会实现移动函数的调用，答案就是：std::move。

    template<typename T>  
    inline typename std::remove_reference<T>::type&&  
    move(T&& t)  
    { 
      return static_cast<typename std::remove_reference<T>::type&&>(t); 
    }  

根据模板参数推导规则，当传入参数是一个左值的时候，T会被推导为T&，于是T& + T&&推导为T&，实际上也就是move(&)，当传入参数是一个右值T&&的时候，根据 T&& + T&&推导为T&&，所以无论如何，move最终一定是返回了T&&。对于类型推导： 

> 当函数模板的模板参数为T而函数形参为T&&（右值引用）时适用本规则。若实参为左值 A& ，则模板参数 T 应推导为引用类型 A& 。（根据引用
  折叠规则， A& + && => A&， 而T&& <=> A&，故T <=> A& ）若实参为右值 A&& ，则模板参数 T 应推导为非引用类型 A 。（根据引用折叠规则， A或A&& + && => A&&， 而T&& <=> A&&，故T <=> A或A&&，这里强制规定T <=> A）。

所以我们的代码需要进行修改以调用A的移动构造函数，移动拷贝函数也同理如此：

    A a(std::move(A()));
    A a1(std::move(Get()));

还记得C++98/0x中的auto\_ptr吗？auto\_ptr在"拷贝"的时候其实并非严格意义上的拷贝。"拷贝"是要保留源对象不变，并基于它复制出一个新的对象出来。但auto\_ptr的"拷贝"却会将源对象"掏空"，只留一个空壳，实际上是一次资源所有权的转移，但auto\_ptr的危险之处在于看上去应该是复制，实际上确实转移。auto\_ptr调用被转移过的的成员函数将会导致不可预知的后果, C++11中被unique\_ptr替换，而unique\_ptr就是用move语义实现的。

##### std::move的关键字：效率优化
我们首先是要解决一些问题的。std::move的出现就是解决在效率上的问题，减少不必要的拷贝。如果存在这样一个移动构造函数的话，所有源对象为临时对象的拷贝构造行为都可以简化为移动式(move)构造。对于普通的string类型而言，std::move和copy construction之间的效率差是节省了一次O(n)的分配操作，一次O(n)的拷贝操作，一次O(1)的析构操作（被拷贝的那个临时对象的析构）。这里的效率提升是显而易见且显著的。

#### 完美转发(std::forward)

程序员在不写高度泛型代码的时候，完美转发的好处可能还体会不到，在C++98/0x时代，当我们需要写泛型代码的时候，可能会遇到函数调用的一些问题。

    void __Test(int &t)
    {
    }
    
    template<typename T>
    void Test(const T &t)
    {
      // do other things... 
      __Test(t);
    }
    
    int i = 0;
    Test(i);

上述的代码将会无法通过编译，究其原因，t在进入Test之后为const T&，而调用\_\_Test之后是T&，模板无法从const T&推导为T&。针对这个问题，我们有两种方法解决：

    template<typename T>
    void Test(const T& t)
    {
      // do other things...
      __Test(const_cast<T&>(t));
    }

经过修改后，Test可以编译通过。我们通过const\_cast去掉了const属性，仔细想想，用const\_cast合适吗？显然这里用const\_cast破坏了函数的健壮性。有没有其他的方法解决呢？我们特化模板表以适当的类型或重载调用函数解决这个问题：

    void __Test(const int &t)
    {}
    
    // or
    void __Test(int & t)
    {}
    // or
    template<typename T>
    void Test(T &t)
    {}
    
    // or
    template<typename T>
    void Test(T &t)
    {}

这样问题顺利解决，无论是我们调用Test(i)还是Test(5)都可以顺利通过编译。

现实世界里的代码不会这么简单，上面的测试代码只涉及到了一个参数，假如某一模板函数有两个参数遇到这种情况之后怎么办？C++0x/98下我们可能会写出这样的代码：

    template<typename T>
    void Test(T &t1, T &t2)
    {
      __Test(t1, t2);
    }
    template<typename T>
    void Test(const T &t1, const T &t2)
    {
      __Test(t1, t2);
    }
    template<typename T>
    void Test(const T &t1, T &t2)
    {
      __Test(t1, t2);
    }
    template<typename T>
    void Test(T &t1, const T &t2)
    {
      __Test(t1, t2);
    }

当我写完这一段示例代码，我的第一感觉是：这简直就是一段毫无技术含量的活，程序员万万不能将时间浪费在这一段代码上面。当我们有两个参数的模板函数会有4个特化.问题不在这里，因为参数类型不同，Test最终将参数转发到\_\_Test中了，意味着什么？意味着我们还要为\_\_Test重载不同参数类型的版本。两个参数能重载出4个，如果有三个参数，四个参数甚至五个参数的时候，\_\_Test究竟需要多少个重载函数才能满足？这样就是为什么VC9的std::bind里面5个参数就重载了63个函数，多么庞大的数量，而这些全是靠人力堆积起来的。

要么将模板函数Test数量减少，要么将被转发函数\_\_Test数量减少，程序员应该将大脑和手用在需要的地方。

现在我们有了左值和右值得概念，这种问题有比较好的解决手段。

    template<typename T>
    struct RemoveReference
    {
      typedef T Type;
    };
    
    template<typename T>
    struct RemoveReference<T&>
    {
      typedef T Type;
    };
    
    template<typename T>
    struct RemoveReference<T&&>
    {
      typedef T Type;
    };
    
    template<typename T>
    T&& ForwardValue(typename RemoveReference<T>::Type&& value)
    {
      return (T&&)value;
    }
    
    template<typename T>
    T&& ForwardValue(typename RemoveReference<T>::Type& value)
    {
      return (T&&)value;
    }

有了上面的代码之后（实际上也就是std::forward的大致实现），然后我们的代码就简单多了：

    template<typename T>
    void Test(T &&t1, T &&t2)
    {
      __Test(ForwardValue<T>(t1), ForwardValue<T>(t2));
    }

无论我们的\_\_Test接受什么形式的参数，即使有多个形式的\_\_Test，都可以调用到正确的\_\_Test中，从而不再需要特化多个模板函数，这就是完美转发。

所谓完美转发：无论目的调用函数需要哪种类型的参数都可以正确调用到我们想要的那个函数里。上main的示例代码中出现了&&，暂且我们把&&这个叫做右值引用吧。C++11中实现完美转发是依靠的类型推导和引用折叠。类型推导不用多说，STL中的容器广泛使用了类型推导，那引用折叠式什么？

引用折叠规则就是函数接受参数形式与传入参数形式之间进行引用简化，具体编译器定义了这样一条规则：

	1. T& + & => T&
	2. T&& + & => T&
	3. T& + && => T&
	4. T&& + && => T&&

上面的规则中，前者代表接受类型，后者代表进入类型，=>表示引用折叠之后的类型，即最后被推导决断的类型。

现在我们用一段实例代码来看看引用折叠是怎么运作的。

    template <typename T> 
    struct Name;
    
    template <> 
    struct Name<string> 
    {
      static const char * get() 
      {
        return "string";
      }
    };
    
    template <> 
    struct Name<const string> 
    {
      static const char * get() 
      {
        return "const string";
      }
    };
    
    template <> 
    struct Name<string&> 
    {
      static const char * get() 
      {
        return "string&";
      }
    };
    
    template <> 
    struct Name<const string&> 
    {
      static const char * get() 
      {
        return "const string&";
      }
    };
    
    template <> 
    struct Name<string&&> 
    {
      static const char * get() 
      {
        return "string&&";
      }
    };
    
    template <> 
    struct Name<const string&&> 
    {
      static const char * get()
      {
        return "const string&&";
      }
    };
    
    template <typename T> 
    void quark(T&& t) 
    {
      cout << "**********************************" << endl;
      cout << "t: " << t << endl;
      cout << "T: " << Name<T>::get() << endl;  // -->A
      cout << "T&&: " << Name<T&&>::get() << endl;  // -->B
      cout << endl;
    }
    
    string strange() 
    {
      return "strange()";
    }
    
    const string charm() 
    {
      return "charm()";
    }
    
    
    int main()
    {
      string up("up");
      const string down("down");
      
      quark(up);	// -->1
      quark(down);	// -->2
      quark(strange());	// -->3
      quark(charm());		// -->4
      
      return 0;
    }

从上面的实例代码大致可以知晓引用折叠的运作。上述代码运行结果：

    **********************************
    t: up
    T: string&
    T&&: string&
    
    **********************************
    t: down
    T: const string&
    T&&: const string&
    
    **********************************
    t: strange()
    T: string
    T&&: string&&
    
    **********************************
    t: charm()
    T: const string
    T&&: const string&&

1. 在调用quack时，up被推导为string&类型。在quack中，T会被推导为string&类型，根据 T& + && -> T&规则，也就是说进入到quack中，T是string&，所以A调用了string&的版本。仍然是上述规则，在执行B时，T依旧被转换和推导为string&，所以就走到了相应的版本中。
2. 因为down具有const属性，所以表现形式与1一样，唯一的区别是A和B均调用到具有const属性的版本。
3. 这里有点特殊，因为strange()返回的是临时对象，类型为string，因此进入quack之后，T仍旧为string类型，因此A最终会进入string的调用版本，同理T&&就是string&&，会进入对应的版本中。
4. 这里有点特殊，因为charm()返回的是临时对象，类型为string，因此进入quack之后，T为string类型，由于有const属性，所以A和B都会调用到const属性的版本。

上面的代码中我们给出了std::forward的大致实现，其实说白了，就是利用引用折叠规则保留参数原始类型，拒绝编译器的类型推导，以达到将参数完美转发到目的函数中。

之所有存在完美转发，其问题实质是：模板参数类型推导在转发过程中无法保证左右值的引用问题。而完美转发就是在不破坏const属性的前提下通过增加左右值引用概念和新增参数推导规则解决这个问题。

##### std::forward的关键字：解决
在模板内，如果我们需要将一组参数原封不动的传递给另外一个参数，在没有完美转发的情况下，考虑参数会有多种类型的重载，因此在没有完美转发的情况下，重载函数个数将会达到2^n个，多么庞大的人工量。当使用std::forward辅以模板参数推导规则以保持参数属性不变，实现完美转发节省了大量的工作。

**注：**以上代码均在VS2015 + Win10上测试。