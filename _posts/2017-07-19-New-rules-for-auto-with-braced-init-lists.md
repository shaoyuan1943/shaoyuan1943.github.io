---
layout: post
title: "auto对于列表初始化的新规则"
date: 2017-07-19
categories: C++17/14/11
---

自从C++11引入```{}```这种通用列表初始化方式之后，截止到C++17，C++中共有七种对象[初始化方式](http://en.cppreference.com/w/cpp/language/initialization)。C++11中重新解释了auto关键字作为类型推导，于是在C++17版本中，对于这些初始化方式在auto关键字下如何运作的有了新的规则：[New rules for auto with braced-init-lists](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n3922.html)。

* 默认初始化
	
		std::string str;

* 值初始化

		std::string str{ "Hello World" };

* 直接初始化

		std::string str("Hello World");

* 拷贝初始化

		std::string str = "Hello World";

* 列表初始化

		std::string str{'H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd'};

* 聚合初始化

		char str[11] = {'H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd'};

* 引用初始化

		char& c = str[0];

在C++11引入```{}```列表初始化方式之后，所有的初始化方式可以被简化成两种：  

* 直接列表初始化

		T t{arg0, arg1, arg2, ...}

* 拷贝列表初始化

		T t = {arg0, arg1, arg2, ...}

当auto和```{}```配合使用时，auto会对初始化列表类型进行推导，依据规则，直接列表初始化将会把类型推导为原始类型T，而拷贝列表初始化则会把类型推导为为```std::initializer_list<T>```，但是在C++17以前，某些情况下```auto```会将对象全部推导为```std::initializer_list<T>```。

	auto a = { 1 };
	auto b{ 1 };
	auto c = { 1, 2 };
	auto d{ 1, 2 };

直接初始化的理想结果是类型为```int```，但是上述代码全部被推导为```std::initializer_list<int>```，这就是矛盾所在。所以在C++17中，对于```auto with braced-init-list```有了新的规则解释以强化```auto```的类型推导：

> * 对于直接列表初始化，如果列表具有单个元素，则会推导类型为T，当列表具有多个元素时不合法。  
> * 对于拷贝列表初始化，如果所有元素类型相同，则会推导类型为```std::initializer_list<int>```。

新的规则解决了上述代码问题：

* ```auto a = { 1 };``` 推导为 ```std::initializer_list<int>```。
* ```auto b{ 1 };``` 推导为 ```int```。
* ```auto c = { 1, 2 };``` 推导为 ```std::initializer_list<int>```。
* ```auto d{ 1, 2 };``` 不合法，程序编译不通过。
