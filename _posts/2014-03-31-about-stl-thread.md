---
layout: post
title: "std中的std::thread"
categories: c++
---

C++11终于有了原生的thread了，我之前一直都是用的自己封装的_beginthreadex，在看了C++11的手册之后发现std::thread更加方便，而且还封装了std::mutex原子操作，看起来相当的方便。

在使用_beginthreadex的过程中需要传入五个参数，反正我个人觉得是很麻烦的，虽然使用上比较方便，但是在使用前的准备工作还是有点麻烦的，现在std::thread提供了更加简单方便的使用方式，所以决心要试试std::thread的使用如何。

环境：VS2013 + win8.1
使用std::thread需要包含<thread>头文件：  

``` c++
void ThreadFunction()  
{  
    std::cout << "this is test ThreadFunction" << endl;  
}  
  
int _tmain(int argc, _TCHAR* argv[])  
{  
    std::thread thread1(ThreadFunction);  
    thread1.join();  
  
    return 0;  
}  
```

在我们没有添加thread1.join()这句话时，运行之后AV了，看了thread的实现之后发现join实际上封装了WaitForSingleObject操作，即等待thread1线程执行完毕。一句代码就开了一个线程，看起来很方便，而且std::thread支持多种线程函数绑定，支持lambda表达式绑定。  

``` c++
auto fn = [](){std::cout << "this is lambda!" << endl; };  
std::thread thread2(fn);  
thread2.join(); 
```
 
同时也还支持std::function函数对象的绑定：  

``` c++
void TestFunctionAndBind()  
{  
    std::cout << "this is std::function<void ()> and std::bind!" << endl;  
}  
  
std::function<void()> fn1 = std::bind(TestFunctionAndBind);  
std::thread thread3(fn1);  
thread3.join();  
```

对仿函数对象也是支持的，想要的特性几乎都有：  

``` c++
struct TestFunctor  
{  
    void operator() ()  
    {  
        std::cout << "this is TestFunctor " << endl;  
    }  
};  
  
std::thread thread4((TestFunctor()));  
thread4.join(); 
```
 
std::thread使用起来确实是非常方便，支持多种线程函数的绑定。

实际上std::thread源自于boost中的thread库，我找来了boost的源代码作为对比，大部分的处理很类似。std::thread同样是在内部封装了CreateThread操作。而且STL中也提供了对了对应的std::mutex原子操作，暂时不知道std::thread的性能如何，明天有空的时候大致测试一下std::mutex的性能，详细看下std::thread中的实现。