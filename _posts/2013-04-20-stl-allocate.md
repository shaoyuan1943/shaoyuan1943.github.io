---
layout: post
title: "STL中空间配置器探究(一)"
date:   2013-04-20 00:00:00
categories: Win32
---

* content
{:toc}

总所周知，vector是线性容器而且内部动态空间分配，可以随着元素的增加内部自动配置空间。对于其vector内部如何实现自行扩充空间和“配置新空间-数据移动-释放就空间”的了解还是比较重要，当然这样对于STL这样的高效工具是必须要了解的。  

STL有几个版本，本来想直接阅读VS2010中的源代码的，但是发现VS2010中STL的代码实现可阅读程度不是很高(当然，看个人感觉)，阅读起来也不舒服，而且源文件找起来也相对麻烦，和linux下的STL版本好多文件名都修改了。正好手上有SGI版本的STL，可阅读程度比较高，就选择SGI版本的STL阅读，其实几个版本的STL实现实际也都是大同小异，一通全通，大概也就是这个理儿。有些许地方会将VC的STL版本和SGI的STL版本比对，会在文中指出的。  

先来看看vector的定义(SGI)：  

    template <class _Tp, class _Alloc = __STL_DEFAULT_ALLOCATOR(_Tp) >  
    class vector : protected _Vector_base<_Tp, _Alloc>   
    {};
  
看看VC版本：  

    template<class _Ty, class _Ax = allocator<_Ty> >  
    class vector: public _Vector_val<_Ty, _Ax>  
    {};
 
两者都差不多，注意第二个模板参数，都有一个默认参数 allocator<_Ty>,实际这个类就是vector内部的内存分配器，几乎所有的STL组件都使用了这个默认的内存分配器。我在defalloc.h文件中找到了allocator：  

    template <class T>  
    class allocator   
    {  
    public:  
        typedef T value_type;  
        typedef T* pointer;  
        typedef const T* const_pointer;  
        typedef T& reference;  
        typedef const T& const_reference;  
        typedef size_t size_type;  
        typedef ptrdiff_t difference_type;  
      
        pointer allocate(size_type n)   
        {   
            return ::allocate((difference_type)n, (pointer)0);  
        }  
        void deallocate(pointer p)   
        {   
            ::deallocate(p);  
        }  
        pointer address(reference x)   
        {   
            return (pointer)&x;   
        }  
        const_pointer const_address(const_reference x)   
        {   
            return (const_pointer)&x;   
        }  
        size_type init_page_size()   
        {   
            return max(size_type(1), size_type(4096/sizeof(T)));   
        }  
        size_type max_size() const   
        {   
            return max(size_type(1), size_type(UINT_MAX/sizeof(T)));   
        }  
    };  

在目前没有资料的情况对这个模板类的具体情况只能靠具体分析了。allocate应该是分配内存的，转调用了全局的allocate函数，deallocate应该是处理内存（应该就是释放内存）函数，转调用了全局的deallocate函数，const_address返回了T的指针。两个全局的实现：  

    template <class T>  
    inline T* allocate(ptrdiff_t size, T*)   
    {  
        set_new_handler(0);  
        T* tmp = (T*)(::operator new((size_t)(size * sizeof(T))));  
        if (tmp == 0)   
        {  
            cerr << "out of memory" << endl;   
            exit(1);  
        }  
        return tmp;  
    }  
      
      
    template <class T>  
    inline void deallocate(T* buffer)   
    {  
        ::operator delete(buffer);  
    }  

也就是说这个默认的配置器中采用了new和delete分配内存。到这里也许应该说要完了吧，但是就这么点？貌似哪里有点不对。因为我们只看到了目前对new和delete的一层简单封装，还没有看到更加实质性的东西。  

在www.sgi.com网站上找到了STL的Guide，在Memory Allocation一章中，发现STL对象的操作在STL中已经被分开，也就是说当对象的构造和析构都是调用了另外的函数进行内存的分配和释放，对于构造函数：  

    template <class T1, class T2>   
    void construct(T1* p, const T2& value);  

对于析构函数：  

    template <class T>   
    void destroy(T* pointer);  
      
    template <class ForwardIterator>  
    void destroy(ForwardIterator first, ForwardIterator last);  

也就是说对象的构造由construct()函数负责，析构由destory()负责。那这样的话，是不是内存的分配由allocate负责，而释放由deallocate负责呢？貌似这样的分工很精密。在<memory>文件中定义了auto_ptr_ref和auto_ptr，貌似没有出现和memory相关的东西，但是却包含了stl_alloc.h和stl_construct.h这两个头文件，看起来像内存分配和构造，那就先去看看吧。在stl_construct.h文件中，就是construct函数和destory的几个不同版本，这个文件里也就大致说明了对象的创建，相对来说比较简单。但是阅读stl_alloc.h文件之后，才发现内存分配才真的是头大。  

STL在内存分配上采用的是两层配置，当所需要内存大于128字节时，就采用第一层配置，当所需的内存较小时，就采用第二层配置，从内存池中取出内存返回给所需要存的“客户”。这样的精细分工节约内存防止碎片产生。麻烦就在两层配置上，比较难以理解，不过阅读源码将这两层配置一层一层撕开就明白了。  

####第一层内存配置器：__malloc_alloc_template  

__malloc_alloc_template比较简单，主要是分配了相当数量大小的内存，源码：  

    template <int __inst>  
    class __malloc_alloc_template   
    {  
      
    private:  
      //out of memory  
      static void* _S_oom_malloc(size_t);  
      static void* _S_oom_realloc(void*, size_t);  
      
    #ifndef __STL_STATIC_TEMPLATE_MEMBER_BUG  
      static void (* __malloc_alloc_oom_handler)();  
    #endif  
      
    public:  
      static void* allocate(size_t __n)  
      {  
        void* __result = malloc(__n);  
        if (0 == __result)  //if malloc not success,than excute _S_oom_malloc()  
            __result = _S_oom_malloc(__n);  
        return __result;  
      }  
      
      static void deallocate(void* __p, size_t /* __n */)  
      {  
        free(__p);  
      }  
      
      static void* reallocate(void* __p, size_t /* old_sz */, size_t __new_sz)  
      {  
        void* __result = realloc(__p, __new_sz);  
        if (0 == __result)  // is realloc is not success,than excute _S_oom_realloc()  
            __result = _S_oom_realloc(__p, __new_sz);  
        return __result;  
      }  
      
      static void (* __set_malloc_handler(void (*__f)()))()  
      {  
        void (* __old)() = __malloc_alloc_oom_handler;  
        __malloc_alloc_oom_handler = __f;  
        return(__old);  
      }  
    };  

可以看到，内存分配与释放时在allocate和deallocate中进行的，在这两个函数里使用的C中的malloc和free，reallocate也是使用的C中的realloc函数。只有当malloc或者realloc失败了，然后就会转换身去调用补救函数，也就是_S_oom_malloc或者_S_oom_realloc。  

####第二层配置器：__default_alloc_template  

    template <bool threads, int inst>  
    class __default_alloc_template 
    {  
      
    private:  
      // Really we should use static const int x = N  
      // instead of enum { x = N }, but few compilers accept the former.  
    #if ! (defined(__SUNPRO_CC) || defined(__GNUC__))  
        enum {_ALIGN = 8};  
        enum {_MAX_BYTES = 128};  
        enum {_NFREELISTS = 16}; // _MAX_BYTES/_ALIGN  
    # endif  
      static size_t  
      _S_round_up(size_t __bytes)   //将字节数调整至8的倍数  
        { return (((__bytes) + (size_t) _ALIGN-1) & ~((size_t) _ALIGN - 1)); }  
      
    __PRIVATE:  
      union _Obj {  
            union _Obj* _M_free_list_link;  //理解成指向自由链表节点的指针，一物二用  
            char _M_client_data[1];    /* The client sees this.        */  
      };  
    private:  
    # if defined(__SUNPRO_CC) || defined(__GNUC__) || defined(__HP_aCC)  
        static _Obj* __STL_VOLATILE _S_free_list[];   
            // Specifying a size results in duplicate def for 4.1  
    # else  
        static _Obj* __STL_VOLATILE _S_free_list[_NFREELISTS]; //定义了自由链表个数是16  
    # endif  
      static  size_t _S_freelist_index(size_t __bytes)   
      {     //根据内存区大小决定使用第几号节点，n从1算起  
            return (((__bytes) + (size_t)_ALIGN-1)/(size_t)_ALIGN - 1);  
      }  
      
      // Returns an object of size __n, and optionally adds to size __n free list.  
      static void* _S_refill(size_t __n);   //填充自由链表  
      // Allocates a chunk for nobjs of size size.  nobjs may be reduced  
      // if it is inconvenient to allocate the requested number.  
      static char* _S_chunk_alloc(size_t __size, int& __nobjs);  
      
      // Chunk allocation state.  
      static char* _S_start_free;  
      static char* _S_end_free;  
      static size_t _S_heap_size;  
      
    # ifdef __STL_THREADS  
        static _STL_mutex_lock _S_node_allocator_lock;  
    # endif  
      
        // It would be nice to use _STL_auto_lock here.  But we  
        // don't need the NULL check.  And we do need a test whether  
        // threads have actually been started.  
        class _Lock;  
        friend class _Lock;  
        class _Lock {  
            public:  
                _Lock() { __NODE_ALLOCATOR_LOCK; }  
                ~_Lock() { __NODE_ALLOCATOR_UNLOCK; }  
        };  
      
    public:  
      
      /* __n must be > 0      */  
      static void* allocate(size_t __n)  
      {  
        void* __ret = 0;  
      
        if (__n > (size_t) _MAX_BYTES) //如果大于128字节，就调用第一级配置  
        {  
          __ret = malloc_alloc::allocate(__n);  
        }  
        else   
        {  
          _Obj* __STL_VOLATILE* __my_free_list  
              = _S_free_list + _S_freelist_index(__n);  
          // Acquire the lock here with a constructor call.  
          // This ensures that it is released in exit or during stack  
          // unwinding.  
    #     ifndef _NOTHREADS  
          /*REFERENCED*/  
          _Lock __lock_instance;  
    #     endif  
          _Obj* __RESTRICT __result = *__my_free_list;  
          if (__result == 0)  
            __ret = _S_refill(_S_round_up(__n));  
          else   
          {  
            *__my_free_list = __result -> _M_free_list_link; //调整__my_free_list的指向  
            __ret = __result;  
          }  
        }  
      
        return __ret;  
      };  

第二层配置器里有一个union，这个union实际上就是就是第二层配置器中自由链表的节点结构。第二层配置器的做法是：如果需要的内存块大于128字节，就交给第一层配置器处理，这个是由一个宏来决定（稍后给出这个宏）。当小于128字节时，就以内存池统一管理。内存池的好处是集中管理小内存块，避免产生碎片。当区块小于128字节时，以memory pool来管理，每次配置一大块内存，由相对应的自由链表来维护，下次再有相同大小的内存，直接可以从自由链表中分配。当分配内存时，配置器回主动将所要分配的内存提升至8的倍数，比如说我需要30字节内存，但是在分配的时候会分配32字节，30字节返回给我，剩下的交给自由链表来维护。16个自由链表节点分别维护8到128字节大小的区块。_S_round_up将所要分配的字节数提升至8的倍数。
```static _Obj* __STL_VOLATILE _S_free_list[_NFREELISTS];``` 定义了一个16个节点的链表。每一个节点维护一块内存，以8的倍数。```_S_freelist_index()```根据大小决定使用第几号空间，如果你是24字节，那就是使用第3号空间，index以1开始。```_S_chunk_alloc()```配置空间，配置容纳__nobjs个大小为__size的内存块。  

好了，基本的逻辑清楚了，那到底是谁来负责分配呢？答案就是allocate函数。当__n大于128字节时，让第一层配置器去分配。如果小于就在自由链表中造一个合适的返回。如果没有找到，那么就先提升到8的倍数，然后_S_refill。如果找到了就调整自由链表的指向位置，然后将所找到的那个内存块返回给客户。allocate是调用```malloc_alloc::allocate```来完成的，而```malloc_alloc::allocate```的来源：  

    typedef __malloc_alloc_template<0> malloc_alloc;  
    //如果定义了__USE_MALLOC就将__malloc_alloc_template<0>作为默认配置器  
    //否则使用__default_alloc_template作为默认配置器  
    # ifdef __USE_MALLOC<span style="white-space:pre">  </span>  
    typedef malloc_alloc alloc;  
    typedef malloc_alloc single_client_alloc;  
    # else  

这样整个allocate结束。当需要释放空间时，deallocate的处理与allocate类似。内存从哪个自由链表节点出来的就返回到哪里去，然后更改自由链表指针指向。  

回到allocate来，当在自由链表中没有找到适合的内存块时，调用了_S_refill函数，_S_refill函数：  

    template <bool __threads, int __inst>  
    void* __default_alloc_template<__threads, __inst>::_S_refill(size_t __n)  
    {  
        int __nobjs = 20;  
        char* __chunk = _S_chunk_alloc(__n, __nobjs);   //取到了一块内存  
        _Obj* __STL_VOLATILE* __my_free_list;  
        _Obj* __result;  
        _Obj* __current_obj;  
        _Obj* __next_obj;  
        int __i;  
      
        if (1 == __nobjs)   
            return(__chunk);  
        __my_free_list = _S_free_list + _S_freelist_index(__n);  
      
        /* Build free list in chunk */  
          __result = (_Obj*)__chunk;      
          *__my_free_list = __next_obj = (_Obj*)(__chunk + __n);    //指向所分配内存下一个偏移地址  
          for (__i = 1; ; __i++)   
          {  
            __current_obj = __next_obj;  
            __next_obj = (_Obj*)((char*)__next_obj + __n);  
            if (__nobjs - 1 == __i)   
            {  
                __current_obj -> _M_free_list_link = 0;  
                break;  
            }   
            else   
            {  
                __current_obj -> _M_free_list_link = __next_obj; //链接自由链表  
            }  
          }  
        return(__result);   //然后返回所取到的那块内存给客户  
    }  

先调用_S_chunk_alloc从内存池中取得__nobjs个大小为n的内存块。然后寻找适合的节点，将所取到的内存块添加进去进行维护。解析来的for循环就是添加节点操作，将整个fre-list串起来了。