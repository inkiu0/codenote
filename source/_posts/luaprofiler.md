---
layout: post
title: "Lua性能分析器的实现"
date: 2017-02-21 23:44
comments: true
tags: 
	- Lua  
	- Unity  
---
![](/assets/blogImg/Unity/luaprofiler.gif)  
这个Profiler主要由3部分组成：
1. C类重写luaC库钩子函数  
2. lua类在钩子的回调中采集信息，并生成报表供C#使用。  
3. C#类每帧取得数据存下来，并每0.5s取最新一帧显示在编辑器中。编辑器窗口完成一些自有功能。  
<!-- more -->
## lua第三方C库的编写  
想要编写一个第三方的luaC库，主要需要以下几点知识：
1. lua栈的概念  
2. lua栈的常用操作  
3. 其他常用的lua C API  
### lua栈的概念  
lua可以作为一个独立运行的程序，也可以作为一个嵌入其他应用的程序库。但是在游戏开发中，lua基本都是作为一个嵌入式的语言存在的。并且把C作为lua的第三方库调用，而C和lua通信的API称为C API。
>C API 是一个 C 代码与 Lua 进行交互的函数集。他有以下部分组成：读写 Lua 全局 变量的函数,调用 Lua 函数的函数，运行 Lua 代码片断的函数，注册 C 函数然后可以在 Lua 中被调用的函数，等等。  

lua的虚拟栈是为了解决lua数据和其他语言数据交互而设计的，否则我们就要面对：
1. 动态类型和静态类型不匹配  
2. 手动管理内存和自动管理内存  

这两个问题的。

而 C 和 lua 交互是基于一个虚拟的栈，所有的API调用都是对这个栈上的值的操作。  
### lua栈的常用操作  
#### lua_open()  
创建一个luastate，即lua的运行环境。但这个环境不包括任何预定义的函数，比如 print 。lualib.h定义了我们打开其他标准库的方法，比如 **luaopen_io(L)** 和 **luaopen_table(L)** 可以创建io table和table table，并把I/O函数、table操作函数注册到lua环境中（比如io.write、table.insert）。  