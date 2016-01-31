---
layout: post
title: "设计一个简单易用的异步任务机制"
date:   2016-01-31
categories: Program-Languages
---

任务机制一般适用于类似下载，数据请求，耗时操作处理等。在各个系统里，Task机制算是比较常见的，网路上也有很多关于Task机制的开源代码足够我们使用。但足够使用并不代表合适，现有的开源有关Task机制的库在功能上大而全，种类繁杂，使用细致，但这从另外一方面也制约了我们的使用，对于一些非实时或者使用量并不高的服务或系统而言，用这种大而全又复杂的开源库反而会让代码陷入“僵尸”阶段，有时候我们仅仅只需要一个简单/多线程/异步的任务机制，它并不需要很复杂的操作，接口尽量简化即可，出于这样的考量，我们可以尝试自己设计一个任务机制。

明白一下我们的设计目的：使用简单，运行要求稳定，线程数量需要严格控制，要可随时扩展为同步任务。分析一下以上需求：  

1. 使用简单  
    使用简单意味着我们不需要经过复杂的初始化或者参数工作，同时也需要接口清晰明了，理想情况下是无参数即可搞定，最多也应该一到两个参数。

2. 运行稳定  
    一个系统的稳定与工作量是成正比的，愈是复杂想要稳定运行付出的工作量就愈大。稳定的前提下是足够我们使用，不使用的功能不加，但正常使用的功能必须是可长久运行的，这也要求我们不能出现内存泄漏同时要求尽可能的排除第三方代码的影响。

3. 线程数量严格控制  
    线程数量影响着系统的性能。线程之间的切换也非常消耗性能。线程数量不能超过当前CPU核心数，现在普通PC基本上都是双核起步，这在一定程度上也限制了我们线程的数量。总之，线程不是越多越好，一个线程能否重复使用才是最佳方案。

4. 可扩展性  
    虽然我们以异步为设计目的，现实情况是有时候我们也需要同步执行任务，所以要求这个机制必须随时能扩展为同步任务。

明白需求了之后，我们就可以开始讨论实现方案了。

既然是Task机制，Task的运行实体是一段代码，那么如何方便的将一段代码绑定到一个可来回移动的变量上，答案是Lambda函数。可以甚至可以把这一段代码抽象成一个Task对象，然后管理器对这个Task对象做操作，我们可以了解下JavaScript中的闭包概念。这样，我们的代码看起来类似这样的:  

    
    class Task
    {
    public:
      typedef std::function<void ()>  Closure;
    	Task(Closure callback)
    	{
    	  excute_ = callback;
    	}
    private:
    	void Run()
    	{
    	  excute_();
    	}
    private:
    	Closure excute_;
    };

有了Task之后，我们需要一个管理器来管理很多Task，所以理论上这个管理器应该被设计成Singelton。这个管理器有一个Task列表，不断轮询这个列表，一旦发现列表不为空就立即执行任务。当这个管理器退出时，需要在当前任务执行完成后清空列表，最后停止线程。对于这个管理器，我们设计的接口类似这样：  

    class TaskMgr
    {
    private:
      void Run();
    public:
      TaskMgr();
      ~TaskMgr();
    
      void Post(Task *task);
      void ExitWait();
      void ExitWithIgnoreTasks();
      void Sleep();
      void Resume();
    private:
      std::thread *taskThread_;
      std::list<Task*> tasks_;
    };

该有的都有了，对于这个管理器，我们一共有5个对外接口，Task则没有对外接口，较好的控制了用户的使用规范了代码，公开的接口都是用户可执行的接口。下面是实现了：

    class TaskMgr
    {
    public:
      TaskMgr()
      {
    	  taskThread_ = new std::thread(std::bind(&TaskMgr::Run, this));
      }
    
      ~TaskMgr()
      {
    	  delete taskThread_;
    	  taskThread_ = nullptr;
      }
    
      void Post(Task *task)
      {
    	  tasks_.push_back(task);
    	  // 如果当前thread处于sleep状态应该Resume
      }
    
      void Run()
    	{
    	  while (true)
    	  {
    		  if (tasks_.size() > 0 && taskThread_)
    		  {
    			  auto task = tasks_.front();
    			  if (task)
    			  {
    				  task->Run();
    			  }
    		  }
    
    		// 无任务时sleep
    	  }
      }
        
      // 等待任务执行完毕时退出
    	void ExitWait()
      {
        if (taskThread_->joinable())
    	  {
    		  taskThread_->join();
    	  }
      }
    
      void ExitWithIgnoreTasks()
      {
    	  Sleep();
    	  // 清除front之后的未执行task
    	  Resume();
      }
    
      void Sleep()
      {
    	  // 睡眠
      }
    
      void Resume()
      {
    	  // 唤醒thread
      }
    private:
      std::thread *taskThread_;
      std::list<Task*> tasks_;
    };

当然，上面的代码并不完善，并没有考虑到Task需要参数/变量之间需要控制竞争等情况。上面的代码只是抛砖引玉，分享一下对于Task机制的经验。其实对于Task需要参数这个问题采用模块特化即可解决，而变量竞争这采用合适的锁即可。对于Task机制，可以看下Chromium的开源代码，其中对于Task机制有一个非常完善且漂亮的实现。