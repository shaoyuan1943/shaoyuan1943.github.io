---
layout: post
title: "STL中可能误用的 find\_first\_of 和 earse "
date:   2013-11-15
categories: Program-Languages
---

####string中find\_first\_of的误用
STL中提供的```string```可以说极大方便了对字符串的操作，但是很多函数由于样子上很相似，所以导致很容易理解错误，```find_first_of```和```find```就是一个很好的例子。我们先来看一下```string```提供的查找相关的函数列表  
```find_first_of()``` 查找第一个与value中的某值相等的字符  
```find_first_not_of()``` 查找第一个与value中的所有值都不相等的字符  
```find_last_of()``` 查找最后一个与value中的某值相等的字符  
```find_last_not_of()``` 查找最后一个与value中的所有值都不相等的字符  
```rfind()``` 查找最后一个与value相等的字符（逆向查找）  
如此简洁的说明，其实完全没有把他们最重要的区别描述出来，请务必记住:  
对于find和rfind，匹配的是整个被查找串。对于```find_first_of,find_first_not_of,find_last_of,find_last_not_of```，匹配的是被查找串中的任意字符。
我们来测试一下:  

    #include <iostream>  
    #include <memory>  
    #include <string>  
    #include <vector>  
    #include <set>  
    #include <map>  
    using namespace std;  
    int main(int argc, const char *argv[])  
    {  
        map<unsigned int,int> myMap;  
        int count = 10;  
        for (int i = 0; i < count; i++)  
        {  
            myMap[i]=i;  
        }  
        for(map<unsigned int, int>::iterator it = myMap.begin(); it != myMap.end(); )  
        {  
            if (it->first == 3 || it->first == 9)  
            {  
                myMap.erase(it++);  
            }  
            else  
            {  
                it++;  
            }  
        }  
        for(map<unsigned int, int>::iterator it = myMap.begin(); it != myMap.end(); ++it)  
        {  
            cout<<it->second<<endl;  
        }  
        return 0;  
    }  

运行结果如下:  

    2  
    4294967295  
    2  
    3  

结果中4294967295即```string::npos```，代表没有找到。而在```find_first_of(str2)```的时候返回的是3，即字符'e'，证明了我们之前的说法。

####erase函数的误用
STL的容器一般都会提供erase方法，而又有很多朋友需要在for循环中对容器进行erase，其实本来很简单的一个问题，现在却被搞得很复杂。为了不给大家造成混淆，这里只列出两种正确的方法，大家按照这种方法来写就绝对没有问题，也不用考虑容器上的区别。  
1.简短型  

    #include <iostream>  
    #include <memory>  
    #include <string>  
    #include <vector>  
    #include <set>  
    #include <map>  
    using namespace std;  
    int main(int argc, const char *argv[])  
    {  
        map<unsigned int,int> myMap;  
        int count = 10;  
        for (int i = 0; i < count; i++)  
        {  
            myMap[i]=i;  
        }  
        for(map<unsigned int, int>::iterator it = myMap.begin(); it != myMap.end(); )  
        {  
            if (it->first == 3 || it->first == 9)  
            {  
                myMap.erase(it++);  
            }  
            else  
            {  
                it++;  
            }  
        }  
        for(map<unsigned int, int>::iterator it = myMap.begin(); it != myMap.end(); ++it)  
        {  
            cout<<it->second<<endl;  
        }  
        return 0;  
    }  

2.易读型  

    #include <iostream>  
    #include <memory>  
    #include <string>  
    #include <vector>  
    #include <set>  
    #include <map>  
    using namespace std;  
    int main(int argc, const char *argv[])  
    {  
        map<unsigned int,int> myMap;  
        int count = 10;  
        for (int i = 0; i < count; i++)  
        {  
            myMap[i]=i;  
        }  
        for(map<unsigned int, int>::iterator it = myMap.begin(); it != myMap.end(); )  
        {  
            map<unsigned int, int>::iterator tempit = it;  
            it++;  
      
            if (tempit->first == 3 || tempit->first == 9)  
            {  
                myMap.erase(tempit);  
            }  
        }  
        for(map<unsigned int, int>::iterator it = myMap.begin(); it != myMap.end(); ++it)  
        {  
            cout<<it->second<<endl;  
        }  
        return 0;  
    }  

运行结果都为:  

    0  
    1  
    2  
    4  
    5  
    6  
    7  
    8  

对于第一种方法，千万不要理解等同于:  

    //这样是错误的，不要模仿！
    if (it->first == 3 || it->first == 9)
    {
        myMap.erase(it);
    }
    it++;

对于我本人来说，更倾向第二种方法，因为虽然这里中是直接调用 ```myMap.erase(tempit);```但实际情况可能是调用一个函数，而在这个函数里面会有一堆逻辑出来判断是否要删除这个元素，这种情况下只有用第二种方法能够满足。OK，STL的强大和危险性是成正比的，所以要熟练运用还是要深入理解才行。