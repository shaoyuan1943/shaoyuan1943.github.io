---
layout: post
title: "STL中空间配置器探究(二)"
categories: Win32
---

在allocate函数中，当```malloc_alloc::allocate```分配失败后，就转向调用_S_refill。_S_refill函数的源代码：  

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
        __my_free_list = _S_free_list + _S_freelist_index(__n); //准备开始调整free list  
      
        /* Build free list in chunk */  
          __result = (_Obj*)__chunk;    // 将此内块返回给客户      
          // 这下面的工作就是填充free list节点，将free list的节点指向内存块  
          *__my_free_list = __next_obj = (_Obj*)(__chunk + __n);    //指向所分配内存下一个偏移地址  
          for (__i = 1; ; __i++)   
          {  
            __current_obj = __next_obj;  
            __next_obj = (_Obj*)((char*)__next_obj + __n);  
            if (__nobjs - 1 == __i)   
        {  
                <span style="white-space:pre">  </span>__current_obj -> _M_free_list_link = 0;  
                <span style="white-space:pre">  </span>break;  
            }   
        else   
        {  
                __current_obj -> _M_free_list_link = __next_obj; //链接自由链表串  
            }  
          }  
        return(__result);   //然后返回所取到的那块内存给客户  
    }  

__n已经被调整至8的倍数，_S_free_list是自由链表首地址，for循环所做的操作时调整free list指向。
内存分配时在_S_chunk_alloc函数中，该函数内所做的操作较多。以当前内存池的容量判断如何分配内存。_S_chunk_alloc函数的源代码：  

    template <bool __threads, int __inst>  
    char* __default_alloc_template<__threads, __inst>::_S_chunk_alloc(size_t __size,   
                                                                int& __nobjs)  
    {  
        char* __result;  
        size_t __total_bytes = __size * __nobjs;    //所需要的内存大小  
        size_t __bytes_left = _S_end_free - _S_start_free;  //内存池剩余大小  
      
        if (__bytes_left >= __total_bytes)   //如果池中剩余大小大于所需要的空间，那就把在线性内存中的区块分配给客户并返回，完成取内存操作  
        {  
            __result = _S_start_free;  
            _S_start_free += __total_bytes;  
            return(__result);  
        }   
        else if (__bytes_left >= __size) //如果池中剩余大小小于所需要的空间，但是还剩下至少一个__size大小，那就先把  
        {                                   //这一个分配出来，防止空间碎片和节约内存使用  
            __nobjs = (int)(__bytes_left/__size);  
            __total_bytes = __size * __nobjs;  
            __result = _S_start_free;  
            _S_start_free += __total_bytes;  
            return(__result);  
        }   
       else //如果连一个__size大小都没有  
       {  
            size_t __bytes_to_get = 2 * __total_bytes + _S_round_up(_S_heap_size >> 4); //要分配的内存大小，大于客户所需的内存  
            // Try to make use of the left-over piece.  
            if (__bytes_left > 0)    //如果池中还有残留的小内存块，先将这个地址配置到free list中  
            {  
                _Obj* __STL_VOLATILE* __my_free_list =  
                            _S_free_list + _S_freelist_index(__bytes_left);  
      
                ((_Obj*)_S_start_free) -> _M_free_list_link = *__my_free_list;  
                *__my_free_list = (_Obj*)_S_start_free;  
            }  
            _S_start_free = (char*)malloc(__bytes_to_get);  //开辟内存空间  
            if (0 == _S_start_free) //如果malloc失败了  
        {  
                size_t __i;  
                _Obj* __STL_VOLATILE* __my_free_list;  
                _Obj* __p;  
                // Try to make do with what we have.  That can't  
                // hurt.  We do not try smaller requests, since that tends  
                // to result in disaster on multi-process machines.  
                for (__i = __size;  
                     __i <= (size_t) _MAX_BYTES;  
                     __i += (size_t) _ALIGN)   
          {  
                    __my_free_list = _S_free_list + _S_freelist_index(__i);  
                    __p = *__my_free_list;  
                    if (0 != __p) <span style="white-space:pre">    </span>//如果说free list中还有没有被使用的内存  
        <span style="white-space:pre">  </span>{  
                        *__my_free_list = __p -> _M_free_list_link;  
                        _S_start_free = (char*)__p;  
                        _S_end_free = _S_start_free + __i;  
                        return(_S_chunk_alloc(__size, __nobjs));    //反复递归调用，看被“挤”出的空间是否够客户所需大小  
                        // Any leftover piece will eventually make it to the  
                        // right free list.  
                    }  
                }  
            _S_end_free = 0;    // In case of exception.  
                _S_start_free = (char*)malloc_alloc::allocate(__bytes_to_get);  //作为补救措施，for有异常的时候就去调用一级配置器  
                // This should either throw an 如果还是没有的话，那没办法了只好抛出异常了  
                // exception or remedy the situation.  Thus we assume it  
                // succeeded.  
            }  
            _S_heap_size += __bytes_to_get;  
            _S_end_free = _S_start_free + __bytes_to_get;  
            return(_S_chunk_alloc(__size, __nobjs));    //很顺利，直接返回给客户  
        }  
    }  

_S_chunk_alloc函数以end_free 和 start_free指针来判定内存池是否还有足够的内存。假如程序开始运行的时候，客户调用_S_chunk_alloc(32,20)，于是分配了20*2个32bytes的内存块，其中前20给客户使用，返回第一个块的地址，其余就交给free-list[3]来维护，还有20个块就留在内存池维护。接下来客户又调用_S_chunk_alloc(62,20),内存池中还剩下20个32bytes大小的块，换算就是10个64bytes块，于是就将这10个给客户使用，返回第一个块的地址，其余就交给free-list[7]维护。接着又调用_S_chunk_alloc(96,20)，先去free-list中找找有没有内存，但是fee-list[11]没啥东西，那就找_S_chunk_alloc求救，但是池中也没有内存了，那就分配内存吧，于是alloc了40个96 bytes块，将20个块给客户使用，返回第一块的地址，其余的交给free-list[11]维护，剩下的20个96 bytes块继续留在内存池中，等待下一次的内存分配。
下图是第二层配置器的大致工作动向：  
![alt text](/img/2013-04-22-1.jpg)  

至此，第二层配置器的神秘面纱被揭开，大致清楚了内存的配置。