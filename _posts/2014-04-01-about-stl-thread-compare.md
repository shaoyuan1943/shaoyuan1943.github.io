---
layout: post
title: "std::thread的性能对比"
date:   2014-04-01
categories: C++
---

今天上班特意来早了点，准备测试一下std::thread中mutex的性能。线程中频繁开关锁是一个内耗比较大的操作，线程间的调度同样也是消耗大的操作，虽然在小项目中使用应该没有多大的问题，一旦项目大了之后这样的内耗也是灰常明显的，虽然我一直都是在写客户端滴，但是对于各原子操作的性能方面东西心里还是有必要清楚一点的。这篇文章主要是测试了一下mutext、关键段```CRITICAL_SECTION```，API提供的Mutex三种原子操作的性能。关键段是用户模式下的同步方式，算是比较开的，mutex属于内核同步方式，相比关键段而言在开关锁都比较慢，且有较大的消耗，当然其实用性那就不同了，有得必有失嘛，所以在线程同步中选用哪种方式create thread，哪种方式同步线程还是一件值得考虑的事情。
测试环境：VS2013 + Win8.1
测试代码：  

    std::mutex mutexLock;  
    CRITICAL_SECTION csLock;  
    HANDLE hMutex = nullptr;  
      
    void TestMutex()  
    {  
        clock_t start = clock();  
        for (int i = 0; i < 1000000; i++)  
        {  
            mutexLock.lock();  
            int k = i;  
            mutexLock.unlock();  
        }  
        std::cout << "mutex耗时1:" << clock() - start << endl;  
    }  
      
    void TestMutex2()  
    {  
        clock_t start = clock();  
        for (int i = 0; i < 1000000; i++)  
        {  
            ::EnterCriticalSection(&csLock);  
            int k = i;  
            ::LeaveCriticalSection(&csLock);  
        }  
        std::cout << "关键段耗时2:" << clock() - start << endl;  
    }  
      
    void TestMutex3()  
    {  
        clock_t start = clock();  
        for (int i = 0; i < 1000000; i++)  
        {  
            ::WaitForSingleObject(hMutex, INFINITE);  
            int k = i;  
            ::ReleaseMutex(hMutex);  
        }  
        std::cout << "mutex耗时3:" << clock() - start << endl;  
    }  
      
    std::thread thread5(TestMutex);  
    thread5.join();  
      
    ::InitializeCriticalSection(&csLock);  
    std::thread thread6(TestMutex2);  
    thread6.join();  
      
    hMutex = ::CreateMutex(nullptr, false, nullptr);  
    std::thread thread7(TestMutex3);  
    thread7.join();  

刚开始我以为相互之间的时间差应该不会太大，但是运行结果出来之后还是在意料之外的，

![alt text](/img/2014-04-01-1.jpg)

Mutex的耗时在意料之中，但是关键段与std::mutex的耗时还是相差很大的，后者是前者的9倍多，性能问题也是比较明显的。我没有看到std::mutex是如何实现的，但是其底层实现应该还是调用系统的API，至于为何性能上相差这么大还不得知，有空了先找找开源代码看看，探寻一下究竟为什么会相差这么大。