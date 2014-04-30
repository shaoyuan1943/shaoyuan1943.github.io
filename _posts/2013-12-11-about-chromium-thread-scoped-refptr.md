---
layout: post
title: "chromium里的scoped_refptr"
categories: c++
---

很多线程在潜移默化中倡导每次操作尽可能开线程去做，同时利用各种锁来对线程进行控制和同步。而有很多时候我们需要做连续性的工作，每一个操作都要开一个线程我自己接受不了，强迫症来了说不定每次都会想下怎么重构。理想下的线程框架应该只有几个线程，而这几个线程跟随主程序生死，将需要用线程做的操作包装成一个对象或者一个函数交给线程去执行，在线程完成工作只需要发送消息通知下主线程或者其他线程告知某一操作已完成。  

chromium的线程框架很完整，而且它鼓励我们尽可能使用已存在的线程，尽量规避锁的使用。chromium采用的是Task and MessageLoop机制，也就是说将需要做的操作封装成一个Task，然后将这个Task post到线程的MessageLoop里，在这个线程的消息循环里会自动检查当前任务列表，MessageLoop单独作为一个对象存在于线程内部，同时也满足跨线程操作。跨线程操作是线程中比较麻烦的一个地方就是资源同步，比如说task1被post到thread1中，task1用到object1，然后task2倍post到了thread1中，但是他妈的现在task1执行完了之后把object1干掉了，等到执行task2的时候，完了，崩溃了。这就是资源没有同步导致的崩溃，object1被task1干掉了，但是task2不知道，或者说这个时候线程thread1不知道object1被干掉了，这种情况下麻烦就大了，好在chromium给出了一套完整的解决办法，那就是通过弱引用。  

chromium中的弱引用没有并非使用的boost中的weak_ptr，chromium中弱引用的实现是基于scoped_refptr的，即区域指针，这个指针是有引用计数的，屁话少说，咱们还是先看代码吧，今天先把scoped_refptr撸完。chromuim中的线程都在src/base/threading目录下，咱们先从scoped_refptr入手，在src/base/memory/ref_counted.h文件中.

``` c++
RefCountedBase; //为子类增加引用计数,基类的基类,无线程安全 
RefCountedThreadSafeBase; //为子类增加引用计数,基类的基类,有线程安全  
RefCounted : public RefCountedBase; //为子类增加引用计数,基类，线程安全 
RefCountedThreadSafe : public RefCountedThreadSafeBase; //为子类增加引用计数,基类，线程安全 
DefaultRefCountedThreadSafeTraits; //为RefCountedThreadSafe提供析构支持 
RefCountedData : public base::RefCounted< base::RefCountedData<T> >; //为其它数据增加引用计数,此类用的略少 
scoped_refptr； //自身有作用域的、其对象带引用计数的 智能指针。出作用域就析构, 释放自己对对象的引用。
```

下面就开始看代码，一个一个来解析吧。  

``` c++
class BASE_EXPORT RefCountedBase   
{  
public:  
    bool HasOneRef() const   
    {   
        return ref_count_ == 1;   
    }  
  
protected:  
    RefCountedBase();  
    ~RefCountedBase();  
  
    void AddRef() const;  
  
    // Returns true if the object should self-delete.  
    bool Release() const;  
  
private:  
    mutable int ref_count_;  
#ifndef NDEBUG  
    mutable bool in_dtor_;  
#endif  
  DFAKE_MUTEX(add_release_);  
  DISALLOW_COPY_AND_ASSIGN(RefCountedBase);  
};  
```

暂且不用管DFAKE_MUTEX宏，DISALLOW_COPY_AND_ASSIGN宏实际上就是声明了宝贝构造函数和赋值函数，阻止编译器的默认版本，加上private权限阻止外部访问，也就是说该类不支持拷贝操作和=操作，DISALLOW_COPY_AND_ASSIGN宏的展开：  

``` c++
#define DISALLOW_COPY_AND_ASSIGN(TypeName) \  
        TypeName(const TypeName&);         \  
        TypeName& operator=(const TypeName&)  
```

RefCountedBase引用计数的最顶层类，有两个最主要的函数AddRef和Release，当所引用的对象被析构的时候，Release函数将返回true，从代码上看出此类并非线程安全的，google当然也给出了此类的一个线程安全版本：  

``` c++
class BASE_EXPORT RefCountedThreadSafeBase   
{  
public:  
    bool HasOneRef() const;  
  
protected:  
    RefCountedThreadSafeBase();  
    ~RefCountedThreadSafeBase();  
  
    void AddRef() const;  
  
    // Returns true if the object should self-delete.  
    bool Release() const;  
  
private:  
    mutable AtomicRefCount ref_count_;  
#ifndef NDEBUG  
    mutable bool in_dtor_;  
#endif  
    DISALLOW_COPY_AND_ASSIGN(RefCountedThreadSafeBase);  
};  
```

线程安全版本不同点在于ref_count_由AtomicRefCount定义，线程安全就是由AtomicRefCount这个类型来完成安全实现的。在src/base/atomicops.h文件中：  

``` c++
typedef int32 Atomic32;  
typedef int64_t Atomic64;  
typedef intptr_t Atomic64;  
```

在src/base/atomic_ref_count.h文件中：

``` c++
typedef subtle::Atomic32 AtomicRefCount;  
```

实际上ref_count_也就是一个int32类型，涉及到线程安全的是AddRef和Release函数：  

``` c++
void RefCountedThreadSafeBase::AddRef() const   
{  
    AtomicRefCountInc(&ref_count_);  
}  
  
bool RefCountedThreadSafeBase::Release() const   
{  
    if (!AtomicRefCountDec(&ref_count_))   
    {  
        return true;  
    }  
    return false;  
}  
```

实际上就是ref_count_实现线程安全的+1和-1，这个倒很好解决，我们在自己写的时候可以使用windows里的CRITICAL_SECTION关键段来实现线程安全。接下来是RefCounted类：  

``` c++
// You should always make your destructor private, to avoid any code deleting  
// the object accidently while there are references to it.  
template <class T>  
class RefCounted : public subtle::RefCountedBase   
{  
public:  
    RefCounted() {}  
  
    void AddRef() const   
    {  
        subtle::RefCountedBase::AddRef();  
    }  
  
    void Release() const   
    {  
      if (subtle::RefCountedBase::Release())   
      {  
          delete static_cast<const T*>(this);  
      }  
    }  
protected:  
    ~RefCounted() {}  
private:  
    DISALLOW_COPY_AND_ASSIGN(RefCounted<T>);  
};  
```

RefCountedBase只是在Release中实现了-1，在RefCounted里面Release则是delete掉了所引用的对象。接下来是RefCountedThreadSafe这个类：  

``` c++
// Forward declaration.  
template <class T, typename Traits> class RefCountedThreadSafe;  
// Default traits for RefCountedThreadSafe<T>.  Deletes the object when its ref  
// count reaches 0.  Overload to delete it on a different thread etc.  
template<typename T>  
struct DefaultRefCountedThreadSafeTraits   
{  
    static void Destruct(const T* x)   
    {  
      // Delete through RefCountedThreadSafe to make child classes only need to be  
      // friend with RefCountedThreadSafe instead of this struct, which is an  
      // implementation detail.  
      RefCountedThreadSafe<T,DefaultRefCountedThreadSafeTraits>::DeleteInternal(x);  
    }  
};  
//  
// A thread-safe variant of RefCounted<T>  
//  
//   class MyFoo : public base::RefCountedThreadSafe<MyFoo> {  
//    ...  
//   };  
//  
// If you're using the default trait, then you should add compile time  
// asserts that no one else is deleting your object.  i.e.  
//    private:  
//     friend class base::RefCountedThreadSafe<MyFoo>;  
//     ~MyFoo();  
template <class T, typename Traits = DefaultRefCountedThreadSafeTraits<T> >  
class RefCountedThreadSafe : public subtle::RefCountedThreadSafeBase   
{  
public:  
    RefCountedThreadSafe() {}  
    void AddRef() const   
    {  
      subtle::RefCountedThreadSafeBase::AddRef();  
    }  
    void Release() const   
    {  
        if (subtle::RefCountedThreadSafeBase::Release())   
        {  
            Traits::Destruct(static_cast<const T*>(this));  
        }  
    }  
protected:  
    ~RefCountedThreadSafe() {}  
private:  
    friend struct DefaultRefCountedThreadSafeTraits<T>;  
    static void DeleteInternal(const T* x) { delete x; }  
    DISALLOW_COPY_AND_ASSIGN(RefCountedThreadSafe);  
};  
```

在这个线程安全的引用计数版本里，AddRef是调用的基类的AddRef，在Release中也调用了基类的Release，同时做了一个判断如果基类Release返回true则说明没有再引用T了，需要将T干掉，具体的干掉动作是有DefaultRefCountedThreadSafeTraits完成，也就是说T的new和destruct是分开进行的，将destruct分开的好处能够提供一定的灵活性，比如说在析构了T之后还需要做一些其余的动作，而不必将业务死锁在引用计数里。到这里为止，google提供了两个版本的引用计数，一个是非线程安全的RefCounted，一个是线程安全的RefCountedThreadSafe，可以自由选择版本。好了，引用计数的部分解决完了，接下来就是正菜了，scoped_refptr，直接先上代码再说： 
 
``` c++
template <class T>  
class scoped_refptr   
{  
 public:  
  typedef T element_type;  
  
  scoped_refptr() : ptr_(NULL)   
  {  
  }  
  
  scoped_refptr(T* p) : ptr_(p)   
  {  
    if (ptr_)  
      ptr_->AddRef();  
  }  
  
  scoped_refptr(const scoped_refptr<T>& r) : ptr_(r.ptr_)   
  {  
    if (ptr_)  
      ptr_->AddRef();  
  }  
  
  template <typename U>  
  scoped_refptr(const scoped_refptr<U>& r) : ptr_(r.get())   
  {  
    if (ptr_)  
      ptr_->AddRef();  
  }  
  
  ~scoped_refptr()   
  {  
    if (ptr_)  
      ptr_->Release();  
  }  
  
  T* get() const { return ptr_; }  
  
  // Allow scoped_refptr<C> to be used in boolean expression  
  // and comparison operations.  
  operator T*() const { return ptr_; }  
  
  T* operator->() const   
  {  
    assert(ptr_ != NULL);  
    return ptr_;  
  }  
  
  scoped_refptr<T>& operator=(T* p)   
  {  
    // AddRef first so that self assignment should work  
    if (p)  
      p->AddRef();  
    T* old_ptr = ptr_;  
    ptr_ = p;  
    if (old_ptr)  
      old_ptr->Release();  
    return *this;  
  }  
  
  scoped_refptr<T>& operator=(const scoped_refptr<T>& r)   
  {  
    return *this = r.ptr_;  
  }  
  
  template <typename U>  
  scoped_refptr<T>& operator=(const scoped_refptr<U>& r)   
  {  
    return *this = r.get();  
  }  
  
  void swap(T** pp)  
  {  
    T* p = ptr_;  
    ptr_ = *pp;   // invoke scoped_refptr<T>& operator=(T* p)   
    *pp = p;  
  }  
  
  void swap(scoped_refptr<T>& r)   
  {  
    swap(&r.ptr_);  
  }  
  
 protected:  
  T* ptr_;  
};  
```

scoped_refptr的代码稍微比较简单，具体就不怎么讲了，先来个栗子吧。  

``` c++
class MyFoo : public RefCounted<MyFoo> // class MyFoo : public RefCountedThread<MyFoo>  
{  
    // TODO...  
public:  
    void DoYourSister()  
    {  
        std::cout << "Fuck!" << endl;  
    }  
};  
  
// 在DoSomething完成之后，foo被析构同时MyFoo被new出来的内存也被析构  
void DoSomething()  
{  
     scoped_refptr<MyFoo> foo = new MyFoo();  
     foo->DoYourSister();  
}  
  
void DoSomething2()  
{  
    scoped_refptr<MyFoo> a = new MyFoo();  
    scoped_refptr<MyFoo> b;  
    // swap完成之后a的引用为空，具体可以参看swap函数  
    b.swap(a);  
}  
  
void DoSomething3()  
{  
    scoped_refptr<MyFoo> a = new MyFoo();  
    scoped_refptr<MyFoo> b;  
    // 操作完成后a和b均有相同的引用就是new MyFoo(),具体可以参看scoped_refptr对 “=” 的重载   
    b = a;  
}  
```

上面的栗子选取的是非线程安全版本，当然你也可以选择线程安全版本，对于scoped_refptr的用法看下源代码即可明白，主要还是要明白RefCounted和RefCountedThreadSafe。