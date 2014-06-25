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

