---
layout: post
title: "C++桥接Lua"
date:   2014-04-28
categories: Game-Dev
---


* content
{:toc}

现在开始用lua写游戏相关的东西了，之前我只用过C++和JS，lua也算是现学现用了，具体时间算下来约莫使用lua两个月左右了吧。

我们的游戏是cocos2d-x和lua写的，其中lua用来写游戏逻辑方面的东西。接触了游戏相关的技能之后，我一直对C++和lua之间的桥接比较好奇，因为之前一直听说lua和C++交互有着天然的优势。以前我做过C++和JS的桥接，用的是谷歌的开源JS引擎V8，虽然很快，但是V8之大也是有目共睹的，而且V8的编译比较麻烦，静态库就快有180MB了，要上手V8还是有一定难度的，要对引擎有一定的熟悉程度，不然使用起来各种问题。而lua和C/C++的交互有很大的优势，lua解释器也不过百来k，即使采用第三方桥接库tolua++也不过几百k，相比V8而言优势很明显。

正因为这样，才对C++里和lua交互有这么好奇，加上lua的使用经验才两个月，所以就一直想了解一下C++如何和lua交互的。

两者交互：  
1、lua与C++函数交互调用  
2、C++导出类供lua调用，lua暴露table变量给C++  

lua里有全局变量的概念，不用local修饰的变量在lua中都被认为是全局变量。lua 5.1之后都是将变量放在寄存器中，而C可以直接对寄存器进行存取，这样交互起来很方便。两者的交互主要是靠栈来实现的，将各种东西都放在栈中。

交互首先需要初始化lua虚拟机，而lua虚拟机都是通过lua_State*来进行存储和操作的。这里有几个函数需要说明：  
1、```lua_State* Lua = luaL_newstate();```新建一个lua虚拟机，即初始化lua的运行环境。  
2、```luaL_openlibs(Lua);```加载lua标准库。  
3、```luaL_dofile(Lua, "Test.lua");```加载lua脚本文件。
上面两行就完成了lua环境的初始化，接下来就可以做各种交互操作了。  
####lua调用C++函数
1、首先得让lua知道C++的函数，向lua注册一个函数，内部实现是以函数指针的形式：  

    lua_register(Lua, "CPlusPlus_Add", CPlusPlus_Add);

这个函数的实现：

    static int CPlusPlus_Add(lua_State* lua)
    {
    	int nParam = lua_gettop(lua);
    	int sum = 0;
    	for (int i = 1; i <= nParam; i++)
    	{
    		if (!lua_isnumber(lua, i))
    		{
    			lua_pushstring(lua, "Incorrect argument to 'CPlusPlus_Add'");
    			lua_error(lua);
    		}
    		sum = sum + lua_tointeger(lua, i);
    	}
    	lua_pushinteger(lua, sum);
    	return 1;
    }

说明：  

* ```lua_gettop(lua);```返回当前栈内的元素个数，在lua中，每一次的调用都会有一个独立的栈，它独立于lua也独立于C。  
* ```lua_isnumber(lua, i);```检测栈内i位置的元素是否为number类型。  
* ```lua_tointeger(lua, i);```将栈内i位置的元素转换成int类型并且出栈i位置的元素。
* ```lua_pushinteger(lua, sum);```此for结束后，栈内为空，此句就是将sum入栈，此时栈内只有一个元素，因为```return 1;```就是告诉lua返回值的个数。

接下来是调用lua中的函数：

    lua_getglobal(Lua, "add");
    lua_pushinteger(Lua, 6);	//函数参数1  
    lua_pushinteger(Lua, 5);	//函数参数2
    lua_pcall(Lua, 2, 1, 0);

说明：  

* ```lua_getglobal(Lua, "add");```获取lua中全局变量add，add将放在栈顶
* ```lua_pushinteger(Lua, 6);```将6值入栈
* ```lua_pushinteger(Lua, 5);```将5值入栈  
* ```lua_pcall(Lua, 2, 1, 0);```调用栈顶的函数，说明有2个参数1个返回值，最后一个值表示出错时是否catch。

lua脚本中：

    function add(int x, int y)
    	local add = CPlusPlus_Add(x, y);
    	local sum = add + y;
    	return sum;
    end

然后：  

    int result = lua_tointeger(Lua, -1); //从栈中取回返回值  
    lua_pop(Lua, 1);					//清栈
    std::cout << "result = " << result << endl;

####获取lua中table的元素
这个比较简单的，看代码：  

    // 获取lua中表的元素
    lua_getglobal(Lua, "tb");
    if (lua_istable(Lua, -1))
    {
    	lua_getfield(Lua, 1, "sex");
    	lua_pushstring(Lua, "name");
    	// 参数是tb表在栈中的索引，以栈顶的元素作为key，将key对应的value压入栈顶
    	lua_gettable(Lua, -2);
    	cout << "tb.name = " << lua_tostring(Lua, -1) << endl;
    	lua_pop(Lua, -1);
    }

上面两句可以修改成：

    // 获取lua中表的元素
    lua_getglobal(Lua, "tb");
    if (lua_istable(Lua, -1))
    {
    	lua_getfield(Lua, 1, "sex"); // 第二个值为"sex"在table中的索引
    	cout << "tb.name = " << lua_tostring(Lua, -1) << endl;
    	lua_pop(Lua, -1);
    }

####lua调用C++类
1、往lua中注册函数指针，并告诉lua该函数有几个参数，几个返回值。  
2、在lua中调用C/C++注册的函数。  
3、在C/C++的函数中，从栈中依次弹出参数，然后进行数据处理，最后将返回值压入栈。  
lua中不存在class的概念，但是很多时候我们可以利用table来模拟OO，而lua终的metatable就是最重要的东西了。lua里最table的操作其实也是通过操作metatable实现的。lua中访问table时，会首先判断变量是否为table，如果是table则按照index访问目的变量或者函数，如果在index中没有要访问的目的变量或者函数的话，那么lua就会根据\_\_index去查找元方法，我们在利用table模拟OO的时候：

    Base = {};
    Base.new = function (self)
    	local  oo = {};
    	setmetatable(oo,self);
    	self.__index = self;
    	return oo;
    end

将Base设置为oo，将\_\_index设置为自己又返回oo，这样就将\_\_index设置到table表中了，所以我们访问Base的时候table会先通过index去找目的变量或者函数，找不到再通过\_\_index去找已经设置到Base上的元方法。

C++类：

    class CPlusPlus_Test
    {
    public:
    	CPlusPlus_Test(int val)
    	{
    		std::cout << "CPlusPlus_Test::CPlusPlus_Test()->" << val << endl;
    		value = val;
    	}
    	~CPlusPlus_Test()
    	{
    		std::cout << "CPlusPlus_Test::~CPlusPlus_Test()->" << value << endl;
    	}
    public:
    	int Add(int x, int y)
    	{
    		std::cout << "CPlusPlus_Test::Add()->" << x + y << endl;
    		return x + y;
    	}
    
    	int Sum(int x, int y)
    	{
    		std::cout << "CPlusPlus_Test::Sum()->" << x - y << endl;
    		return x - y;
    	}
    
    	void Print()
    	{
    		std::cout << "CPlusPlus_Test::Print()->" << value << endl;
    	}
    private:
    	int value;
    };
    
    static int Create_CPlusplus_Test(lua_State* lua)
    {
    	// 出栈构造参数
    	int value = lua_tointeger(lua, -1);
    	// 然后将new出来的CPlusPlus_Test**放在栈顶
    	*(CPlusPlus_Test**)lua_newuserdata(lua, sizeof(CPlusPlus_Test*)) = new CPlusPlus_Test(value);
    	luaL_getmetatable(lua, "CPlusPlus_Test");
    	lua_setmetatable(lua, -2);
    	return 1;
    }
    
    static int Destory_CPlusplus_Test(lua_State* lua)
    {
    	//释放对象  
    	delete *(CPlusPlus_Test**)lua_topointer(lua, 1);
    	return 0;
    }
    
    static int LuaCall_CPlusPlusAdd(lua_State* lua)
    {	
    	CPlusPlus_Test* pT = *(CPlusPlus_Test**)lua_topointer(lua, 1);
    	lua_pushnumber(lua, pT->Add(lua_tonumber(lua, 2), lua_tonumber(lua, 3)) );
    	return 1;
    }

上面向lua暴露了三个接口:  

    static int Create_CPlusplus_Test(lua_State* lua);  
    static int Destory_CPlusplus_Test(lua_State* lua);  
    static int LuaCall_CPlusPlusAdd(lua_State* lua);

将构造函数和析构函数暴露给lua的主要作用就是让lua来托管类的构造与析构，不然当C++类析构了，而lua不知道，如果lua继续引用C++类就会出问题。
然后：  

    lua_pushcfunction(Lua, Create_CPlusplus_Test);
    lua_setglobal(Lua, "CPlusplus_Test");  // 设置 CPlusplus_Test = Create_CPlusplus_Test;栈为空
    
    luaL_newmetatable(Lua, "CPlusplus_Test"); // 构造一个元表
    
    lua_pushstring(Lua, "__gc");
    lua_pushcfunction(Lua, Destory_CPlusplus_Test);
    lua_settable(Lua, -3);// 设置CPlusplus_Test[__gc] = Destory_CPlusplus_Test; 栈顶为 CPlusplus_Test，将析构暴露给lua
    
    lua_pushstring(Lua, "__index"); 
    lua_pushvalue(Lua, -2); // 将__index复制一份放在栈中，CPlusplus_Test，__index，__index 
    lua_settable(Lua, -3);  // 设置CPlusplus_Test[__index] = __index; 栈顶为 CPlusplus_Test,两个__index都弹出
    
    //放元素中增加函数,这样所有基于该元素的Table就有Add方法了  
    lua_pushstring(Lua, "Add");
    lua_pushcfunction(Lua, LuaCall_CPlusPlusAdd);
    lua_settable(Lua, -3); // 设置 CPlusplus_Test[Add] = LuaCall_CPlusPlusAdd; 栈顶为 CPlusplus_Test
    lua_pop(Lua, 1); // 清栈

lua中可以这样使用：

    function add()
    	local tb = CPlusplus_Test(10);
    	local sum = tb:Add(10, 20);
    	return sum;
    end

####使用tolua++
其实使用上面的办法暴露C++类给lua还是略显麻烦，一旦项目中大量使用lua写逻辑的话，这种方式不仅麻烦而且容易出错，幸好我们有tolua++第三方库。获取tolua++的源代码，编译出dll和lib，加入自己的项目中，然后通过pkg文件生成与lua的绑定文件。我们有这样一个C++类：

    #ifndef _HELLO_H_
    #define _HELLO_H_
    
    class Hello
    {
    public:
    	Hello(int val);
    	int Add(int x, int y);
    	int Sum(int x, int y);
    	void Print();
    	int value;
    };
    
    #endif

然后我们通过tolua++.exe生成绑定文件：
```tolua++ -o hello_toLua.cpp hello.pkg```
然后就会生成hello\_toLua.cpp绑定文件，直接引入项目即可。下面是lua脚本：  

    function TestCPlusPlus()
    	local tb = Hello(20);
    	tb:Print();
    
    	local add = tb:Add(10, 30);
    	local sum = tb:Sum(add, 10);
    	return sum;
    end

调用：  

    lua_State* Lua;
    TOLUA_API int  luaopen_hello(lua_State* tolua_S);
    
    int _tmain(int argc, _TCHAR* argv[])
    {
    	Lua = luaL_newstate();
    	if (Lua == nullptr)
    	{
    		std::cout << "not enough memory!" << endl;
    	}
    
    	luaL_openlibs(Lua);
    	luaopen_hello(Lua);
    	luaL_dofile(Lua, "Test.lua");
    	
    	lua_getglobal(Lua, "TestCPlusPlus");
    	// 如果有参数则需要push参数进栈
    	//lua_pushinteger(Lua, 6);//函数参数1  
    	//lua_pushinteger(Lua, 5);//函数参数2
    	lua_pcall(Lua, 0, 1, 0);	//0个参数1个返回值不catch
    
    	int result = lua_tointeger(Lua, -1);//从栈中取回返回值  
    	lua_pop(Lua, 1);					//清栈
    
    	std::cout << "result = " << result << endl;
    	lua_close(Lua);
    	return 0;
    }

####环境配置
如果不使用tolua++的话，环境就比较好配置了，加载lua的lib，然后包含头文件就可以了。  

    extern "C"
    {
    	#include "lua.h"  
    	#include "lualib.h"  
    	#include "lauxlib.h"
    	#pragma comment(lib, "lua5.1.lib")
    }

如果使用tolua++的话，稍微麻烦一点。tolua++主页上告诉我用Scons编译，我嫌麻烦，直接用VS2013建工程，然后把tolua++的源代码加进来，在tolua++的头文件上改了一下：

    #ifdef _WINDLL   
    #define TOLUA_API __declspec(dllexport)  
    #else  
    #define TOLUA_API extern  
    #endif 

成功导出了，然后加入项目中来：  

    extern "C"
    {
    	#include "lua.h"  
    	#include "lualib.h"  
    	#include "lauxlib.h"
    	#include "tolua++.h"
    	#pragma comment(lib, "lua5.1.lib")
    	#pragma comment(lib, "tolua++.lib")
    }

编译搞定！终于对这个过程熟悉了一点，心里也有底了，成功消除一个好奇点。