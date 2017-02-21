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
