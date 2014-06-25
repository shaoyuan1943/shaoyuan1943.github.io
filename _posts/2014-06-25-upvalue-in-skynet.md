---
layout: post
title: "skynet中upvalue的用法"
categories: lua
---

今天花了几十分钟看了一下skynet中的```lua-clientsocket.c```文件，其实在实际项目中，这一部分代码是需要重写的，所以也就大致看了一下流程。仔细看了这个文件中的```lreadline```，这个函数的实现比较巧妙，主要涉及到lua中的```upvalue```，之前还没有接触到```upvalue```，就当学习了一下```upvalue```的用法。

在客户端，是这么调用:

``` lua
local socket = require("clientsocket");
socket.readline();
```

先看下```luaopen_clientsocket```的源代码：

``` c++
int luaopen_clientsocket(lua_State *L) 
{
	 luaL_checkversion(L);
	 luaL_Reg l[] = 
	 {
		  { "connect", lconnect },
		  { "recv", lrecv },
		  { "send", lsend },
		  { "close", lclose },
		  { "usleep", lusleep },
		  { NULL, NULL },
	 };
	 luaL_newlib(L, l);

	 struct queue * q = lua_newuserdata(L, sizeof(*q));
	 memset(q, 0, sizeof(*q));
	 lua_pushcclosure(L, lreadline, 1); //说明lreadline函数内部只有一个upvalue值
	 lua_setfield(L, -2, "readline");  // socket.readline = lreadline

	 pthread_t pid ;
	 pthread_create(&pid, NULL, readline_stdin, q);

	return 1;
}
```

在上面的函数，首先将几个函数加入了lua虚拟机中。注意```lreadline```并没有添加进去，而是在后面单独做了处理。```lua_newuserdata```首先申请内存返回一个```struct``` ```queue```指针。然后将一个C函数作为```closure```压入栈中，接下来:  

``` lua
tb["readline"] = lreadline;
```

这样使得我们可以在lua中直接使用```socket.readline()```。

在上面的代码中```lua_newuserdata```会将已经创建的内存块压入堆栈，然后```lua_pushcclosure```，这里的巧妙就是将创建的```*q```作为一个```upvalue```设置到```lreadline```函数内部，```upvalue```针对的是值，也就是说```lua_pushcclosure```直接将内存块当成了```lreadline```的```upvalue```，而不是```q```，```q```只是作为一个指针引用```*q```存在的。

``` c++
static int lreadline(lua_State *L) 
{
	 // 找到了在luaopen_clientsocket中push到栈中的struct queue * q
	 struct queue *q = lua_touserdata(L, lua_upvalueindex(1));
	 LOCK(q);
	 if (q->head == q->tail) 
	 {
		  UNLOCK(q);
		  return 0;
	 }
	 char * str = q->queue[q->head];
	 if (++q->head >= QUEUE_SIZE) 
	 {
		  q->head = 0;
	 }
	 UNLOCK(q);
	 lua_pushstring(L, str);
	 free(str);
	 return 1;
}
```

上面```lua_upvalueindex```返回了函数环境中index为1的```upvalue```。然后对返回的```upvalue```进行操作。

而```luaopen_clientsocket```只是创建了一个没有内容的```struct queue，*q```中的内容从哪里来呢。在```readline_stdin```线程中，会去读取console上的内容放在```*q```中，上面说了```upvalue```针对的是值而不是“引用”，因为我们在lua中可以通过```socket.readline```读取到内容。

这里利用到了lua中的```upvalue```。在lua中，函数有自己的上下文环境，特别是```closure```，当执行一个```closure```的时候lua虚拟机会创建一个新的```data object```，它包含了相应函数原型的引用，环境的引用以及一个由所有```upvalue```组成的```table```，这些```upvalue```只对这个```closure```可见。

``` lua
function f1(n)
     local c1 = function ()
          print(n);
     end
     n = n + 10;
     return c1;
end
local c1 = f1(1943);
c1(); -- 1953
```

