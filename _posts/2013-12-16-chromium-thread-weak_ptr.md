---
layout: post
title: "chromium里的weak_ptr"
date:   2013-12-16
categories: Program-Languages
---

weakptr的实现最底层的东西就是```scoped_refptr```目标文件```/src/base/memory/weak_ptr.h```。WeakPtr主要是有如下的几个类构成：  

    class WeakReferenc; // 弱引用，此类中含有Flag类，实现WeakPtr就是检查Flag类是否有效。
    class Flag : public RefCountedThreadSafe<Flag>; // Flag 标志类，继承自引用计数的线程安全版本。
    class WeakReferenceOwner; // 若引用拥有者，有一个Flag的scpoed_refptr的变量，提供一个返回WeakReference接口
    class WeakPtrBase; // WeakPtr的基类，含有一个WeakReference作为成员变量
    class WeakPtr; // 弱指针的实现类，检查了基类中的WeakReference的Flag是否有效
    class WeakPtrFactory; // 主要用来产生一个WeakPtr<T>,被使用类继承此类即可

废话少说，还是先上代码的例子分析好了。

    // 弱引用  
    class WeakReference   
    {  
     public:  
      // Although Flag is bound to a specific thread, it may be deleted from another  
      // via base::WeakPtr::~WeakPtr().  
      // Flag类，检测所用，内部实现实际上就是引用计数  
      class Flag : public RefCountedThreadSafe<Flag>   
      {  
          public:  
          Flag();  
      
          void Invalidate();    // 设置为无效，即is_valid_ = false  
          bool IsValid() const;   
      
         private:  
          friend class base::RefCountedThreadSafe<Flag>;  
      
          ~Flag();  
      
          SequenceChecker sequence_checker_;  
          bool is_valid_;  
      };  
      
      WeakReference();  
      explicit WeakReference(const Flag* flag); // 隐式初始化  
      ~WeakReference();  
      
      bool is_valid() const;  
      
     private:  
        scoped_refptr<const Flag> flag_;  
    };  

WeakPtr最重要的一个类就是Flag，在weakPtr中就是检测这个类是否有效，进而可以判断弱指针是否有效，无效则被销毁。Flag继承自```RefCountedThreadSafe<T>```,说明Flag是线程安全的，含有引用计数。WeakReference中有一个```scoped_refptr<const Flag>```的变量，基本上此类就是靠这个```falg_```变量判断弱引用是否有效的。  

    // WeakReference's ower  
    // WeakPtr检查point的有效性是通过检查flag_来实现的。而flag_是scoped_refptr  
    class WeakReferenceOwner   
    {  
    public:  
        WeakReferenceOwner();  
        ~WeakReferenceOwner();  
      
        WeakReference GetRef() const;   // 返回一个weakReference对象  
      
        bool HasRefs() const   
        {  
            return flag_.get() && !flag_->HasOneRef();  
        }  
      
        void Invalidate();  
      
    private:  
        mutable scoped_refptr<WeakReference::Flag> flag_;  
    };  

```WeakReference```的拥有者，此类是通过```scoped_refptr<WeakReference::Flag> flag_;```来进行工作的，```GetRef()```的实现如下:  

    WeakReference WeakReferenceOwner::GetRef() const   
    {  
      // If we hold the last reference to the Flag then create a new one.  
      if (!HasRefs())  
        flag_ = new WeakReference::Flag();  
      
      return WeakReference(flag_.get());  
    }  

负责new出Flag对象,保证所有从本类请求的Flag对象都是同一个，而同一个Flag对象代表了同一个object指针。  

    // This class simplifies the implementation of WeakPtr's type conversion  
    // constructor by avoiding the need for a public accessor for ref_.  A  
    // WeakPtr<T> cannot access the private members of WeakPtr<U>, so this  
    // base class gives us a way to access ref_ in a protected fashion.  
    class WeakPtrBase   
    {  
     public:  
      WeakPtrBase();  
      ~WeakPtrBase();  
      
     protected:  
      explicit WeakPtrBase(const WeakReference& ref);  
      
      WeakReference ref_;  
    };  

WealPtrBase是WeakPtr的基类，此类拥有一个```WeakReference```，从而保证了WeakPtr拥有一个Flag对象的引用。对象要跨线程，比如说线程1有一个对象A通过指针将对象传递给线程2,线程2怕的是在用的时候，那个对象A被释放了。而如果能有一个地方查到此对象是否还存在的话，跨线程就ok了。假设有对象指针T,Flag对象就是用来标志那个对象T是否还存在的标志。```WeakReferenceOwner```只能产生针对T的WeakReference(同一个Flag构成的)由于Flag是new出来的，它的析构不是由外力决定的，而是由它被引用的情况决定的，因为它是"可引用计数"(意即```scoped_refptr```)的。引用计数归0后自然就是释放内存。所以Flag寿命比object长一些,与最后一个```WeakPtr```一起结束生命。每个WeakReference是Flag的一份引用，而每个WeakPtr中有一个```WeakReference```,当WeakPtr析构时,WeakReference就析构了，最终引起Flag引用数量-1,当引用数量==0时，Flag析构，也就是说在所有WeakPtr析构前，Flag是不会析构的,这个对象可被任何关于T的WeakPtr用来查询，T是否有效构建过程是同一个```WeakReferenceOwner```用同一个Flag来构建WeakPtr的，所以所有WeakPtr都指向同一个对象。看下WeakPtr的实现：  

    template <typename T>  
    class WeakPtr : public internal::WeakPtrBase   
    {  
     public:  
      WeakPtr() : ptr_(NULL) {  
      }  
      
      // Allow conversion from U to T provided U "is a" T. Note that this  
      // is separate from the (implicit) copy constructor.  
      template <typename U>  
      // U 和 T 必须是is-a关系  
      WeakPtr(const WeakPtr<U>& other) : WeakPtrBase(other), ptr_(other.ptr_)   
      {  
      }  
      
      T* get() const   
      {   
          return ref_.is_valid() ? ptr_ : NULL;   
      }  
      
      T& operator*() const   
      {  
          return *get();  
      }  
      
      T* operator->() const   
      {  
          return get();  
      }  
      
      // Allow WeakPtr<element_type> to be used in boolean expressions, but not  
      // implicitly convertible to a real bool (which is dangerous).  
      //  
      // Note that this trick is only safe when the == and != operators  
      // are declared explicitly, as otherwise "weak_ptr1 == weak_ptr2"  
      // will compile but do the wrong thing (i.e., convert to Testable  
      // and then do the comparison).  
     private:  
      typedef T* WeakPtr::*Testable;  
      
     public:  
      operator Testable() const   
      {   
        return get() ? &WeakPtr::ptr_ : NULL;   
      }  
      // set nullptr  
      void reset()   
      {  
        ref_ = internal::WeakReference();  
        ptr_ = NULL;  
      }  
      
     private:  
      // Explicitly declare comparison operators as required by the bool  
      // trick, but keep them private.  
      template <class U>   
      bool operator==(WeakPtr<U> const&) const;  
        
      template <class U>   
      bool operator!=(WeakPtr<U> const&) const;  
      
      friend class internal::SupportsWeakPtrBase;  
      template <typename U> friend class WeakPtr;  
        
      friend class SupportsWeakPtr<T>;  
      friend class WeakPtrFactory<T>;  
      
      WeakPtr(const internal::WeakReference& ref, T* ptr)  
          : WeakPtrBase(ref),  
            ptr_(ptr)   
      {  
      }  
      
      // This pointer is only valid when ref_.is_valid() is true.  Otherwise, its  
      // value is undefined (as opposed to NULL).  
      T* ptr_;  
    };  

最主要的是先都在上面的几个类中完成了，```WeakPtr```只需要做好指针所要做的事情就可以了。```WeakPtr```支持is-a关系之间的转换。具体可以参见```SupportsWeakPtrBase```。目前来说```WeakPtr```指针已经搞定，那个如何将一个对象与一个```WeakPtr```进行绑定呢？答案是```WeakPtrFactory```.  

    template <class T>  
    class WeakPtrFactory   
    {  
     public:  
      explicit WeakPtrFactory(T* ptr) : ptr_(ptr)   
      {  
      }  
      
      ~WeakPtrFactory()   
      {  
        ptr_ = NULL;  
      }  
      
      WeakPtr<T> GetWeakPtr()   
      {  
        DCHECK(ptr_);  
        return WeakPtr<T>(weak_reference_owner_.GetRef(), ptr_);  
      }  
      
      // Call this method to invalidate all existing weak pointers.  
      void InvalidateWeakPtrs()   
      {  
        DCHECK(ptr_);  
        weak_reference_owner_.Invalidate();  
      }  
      
      // Call this method to determine if any weak pointers exist.  
      bool HasWeakPtrs() const   
      {  
        DCHECK(ptr_);  
        return weak_reference_owner_.HasRefs();  
      }  
      
     private:  
      internal::WeakReferenceOwner weak_reference_owner_;  
      T* ptr_;  
      /* 
        禁止隐式构造 
        #define DISALLOW_IMPLICIT_CONSTRUCTORS(TypeName) \ 
        private:                     \ 
        TypeName();                                    \ 
        DISALLOW_COPY_AND_ASSIGN(TypeName) 
      */  
      DISALLOW_IMPLICIT_CONSTRUCTORS(WeakPtrFactory);  
    };  

```WeakFactory```负责产生一个WeakPtr指针，通过```WeakReferenceOwer```产生一个```WeakPtr<T>```的指针。至于如何产生一个```WeakPtr<T>```指针请看```WeakReferenceOwer```中的```GetRef()```的实现。
对于如何使用的问题，请看下面：  

    int data;  
    WeakPtrFactory<int> factory(&data);  
    WeakPtr<int> ptr = factory.GetWeakPtr();  
    //这是针对普通内建内型的使用，在使用WeakPtr之前必须要先对ptr进行null-test:
    class A{};
    A a;
    WeakPtrFactory<A> factory(&a);
    WeakPtr<A> ptr = factory.GetWeakPtr(0;
    if (ptr)
        ptr->fuck();
    
    //还有栗子如下：
    class A : public WeakPtrFactory<A>{};
    A a;
    WeakPtr<A> ptr = a.GetWeakPtr();