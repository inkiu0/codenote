---
layout: post
title: "Lua性能分析器的实现"
date: 2017-02-21 23:44
comments: true
tags: 
	- Lua
	
	- Unity  
---
这个Profiler主要由3部分组成：
1. C类重写luaC库钩子函数  
2. lua类在钩子的回调中采集信息，并生成报表供C#使用。  
3. C#类每帧取得数据存下来，并每0.5s取最新一帧显示在编辑器中。编辑器窗口完成一些自有功能。  
<!-- more -->
## 编写lua第三方C库  
想要编写一个第三方的luaC库，主要需要以下几点知识：
1. lua栈的概念  
2. lua栈的常用操作  
3. C代码示例  

### lua栈的概念  
lua可以作为一个独立运行的程序，也可以作为一个嵌入其他应用的程序库。但是在游戏开发中，lua基本都是作为一个嵌入式的语言存在的。同时把C作为lua的第三方库调用，C和lua通信的API称为C API。
>C API 是一个 C 代码与 Lua 进行交互的函数集。他有以下部分组成：读写 Lua 全局 变量的函数,调用 Lua 函数的函数，运行 Lua 代码片断的函数，注册 C 函数然后可以在 Lua 中被调用的函数，等等。  

而 C 和 lua 交互是基于一个虚拟的栈，所有的API调用都是对这个栈上的值的操作。  
lua的虚拟栈是为了解决lua数据和其他语言数据交互而设计的，否则我们就要面对这两个问题：
1. 动态类型和静态类型不匹配  
2. 手动管理内存和自动管理内存  
  
### lua栈的常用操作  
#### lua_open()  
创建一个luastate，即lua的运行环境。但这个环境不包括任何预定义的函数，比如 print 。lualib.h定义了我们打开其他标准库的方法，比如 **luaopen_io(L)** 和 **luaopen_table(L)** 可以创建io table和table table，并把I/O函数、table操作函数注册到lua环境中（比如io.write、table.insert）。  
#### lua_push*  
压入元素，把数据压入栈中。例如lua_pushstring、lua_pushnil等等。
#### lua_is*  
查询函数，判断数据类型。例如lua_isnumber、lua_istable、lua_isstring、lua_isboolean。  
#### lua_to*  
取值函数，从栈中取得值。例如lua_tonumber、lua_tostring等。  
#### 其他lua栈操作  
- lua_gettop  
- lua_settop  
- lua_pushvalue  
- lua_remove  
- lua_insert  
- lua_replace  
- stack_dump  
dump整个堆栈的内容  

### C代码示例  
```c  
static const char *const hooknames[] = {"call", "return", "line", "count", "tail call"};
static void hook(lua_State *L, lua_Debug *db)
{
    int event;

    //入栈light userdata 索引为1
    lua_pushlightuserdata(L, (void *)p);
    //取值LUA_REGISTRYINDEX，并绕过__index。取得注册的函数后，将函数移到栈顶，即索引为2。
    lua_rawget(L, LUA_REGISTRYINDEX);

    event = db->event;
    //入栈string，索引为3。
    lua_pushstring(L, hooknames[event]);
    //得到debug信息
    lua_getinfo(L, "nS", db);
    if(*(db->what) == 'C')
    {
        //如果是C函数，入栈此string，索引为4。
        lua_pushfstring(L, "[C?%s]", db->name);
    }
    else
    {
        //如果不是C函数，入栈此string，索引为4。
        lua_pushfstring(L, "%s-%d", db->short_src, db->linedefined > 0 ? db->linedefined : 0);
    }
    //调用索引为2的函数，即lua注册的hook回调。
    lua_call(L, 2, 0);
    //C函数退出，并清空当前虚拟栈。
}
```  
```lua  
-- 这时候回调给lua的函数如下
function OnHook(event, id)
	-- event	即索引为3的string
	-- id		即索引为4的string
end
```  

### lua5.3第三方库的注册  
```c  
//这是lua5.3新版的写法
LUAMOD_API int luaopen_profiler(lua_State *L)
{
	luaL_checkversion(L);
	luaL_Reg l[] = {
        { "sethook", profiler_set_hook },
		{ NULL, NULL }
	};
    luaL_newlib(L, l);
	lua_setglobal(L, "profiler");
	return 1;
}

//两者兼容的写法
LUALIB_API int luaopen_bitLib(lua_State *L)
{
    luaL_Reg l[] = 
    {
        { "sethook", profiler_set_hook },
        { NULL, NULL }
    };
#if LUA_VERSION_NUM < 502
  luaL_register(L, "profiler", l);
  lua_pop(L, 1);
#else
  luaL_newlib(L, l);
  lua_setglobal(L, "profiler");
#endif
  return 1;
}
```  

### 编译动态链接库
mac下用gcc生成动态链接库和linux有点不同  
```  
gcc -c profiler.c;gcc -O2 -bundle -undefined dynamic_lookup -o profiler.so profiler.o
```  

## 编写lua函数  
lua代码的编写比较简单，主要有以下一些不常用的用法需要注意。  
### 引入C库  
```lua  
-- 如果我们设置了global名字就可以用全局的名字调用C库里的函数
require "profiler"
local sethook = profiler.sethook

-- 如果我们没有设置global
local p = require "profiler" 
local sethook = p.sethook
```  

### sethook  
debug.sethook(onhook, 'crl')的第一个参数是回调函数，第二个参数是事件的mask字符串。  
- LUA_MASKCALL 0  
在解释器调用一个函数时被调用。 钩子将于 Lua 进入一个新函数后，函数获取参数前被调用。  
- LUA_MASKRET 1  
在解释器从一个函数中返回时调用。 钩子将于 Lua 离开函数之前的那一刻被调用。执行到returnhook的时候已经函数已经退出了。  
- LUA_MASKLINE 2  
在解释器准备开始执行新一行代码，或者跳转到这行代码（即使是同一行跳转）时被调用。  
- LUA_MASKCOUNT 3  
在解释器执行一个lua函数的cout指令时被调用。  
- LUA_HOOKTAILCALL 4  
在解释器执行一个尾调用的时候触发，尾调用对应的return同时也是它父级的return。  
回调函数onhook(event, line)第一个参数是事件类型，值为call, tail call, return。line参数是可选的。  
当我们调用了sethook(onhook, 'cr')的时候，第一个触发的钩子是return sethook。同样当我们调用debug.sethook()来停止的时候，最后一个触发的钩子是call sethook。
### getinfo  
debug.getinfo(2, 'Sn')的第一个参数是要查询的调用层级，第二个参数是要查询的数据的掩码，具体可以查询lua手册去了解。如果得到的信息不准确，比如得到一个C#的函数，往往信息都不对。这个时候我们可以往上回溯，即getinfo(level, 'Sn')，加大level。  
### 报表  
为了方便C#中取得我们采集的信息，lua被设计成每次report时：
- 暂停hook  
- 整理数据  
- 返回所有累计记录的数据  
- 重启hook，并清空数据。  
## 编写C#类  
C#主要由2部分组成，处理数据和更新UI。  
### 处理数据  
每帧的Update会从lua中取得最新的数据，然后缓存下来。每0.5s取最新的数据解析到树状结构中，由于在lua返回的已经是一个树状结构的json字符串了，所以解析的过程比较简单，一个递归就可以解决了。  
*已存在的节点保存了flodout状态，所以当我们解析到一个已存在节点的时候需要的是更新*
### OnGUI  
每帧编辑器都会尝试绘制树状结构，而当我们更新树状结构数据后，OnGUI才真正显示出新的数据。  
*OnGUI每帧都要绘制，要不然点击事件等等都不会响应了，所有一切视图上的操作反馈都以来OnGUI的更新*  
OnGUI不同的是在于各种Editor下控件的使用，还可以利用反射去的Unity内置的各种图标，使得编辑器的表现形式更多样。  
## 后记  
实践出真知，只有做了以后才理解的更深刻。要不然只是侃侃而谈，流于表面罢了。这次写编辑器让我对lua的使用更深入了一些，虽然还谈不上什么造诣。但是至少比普通人了解的更多、更深了，了解的更深入对我平时写代码并没有什么帮助，但是当遇到问题的时候，那么我能探究的深度就比别人更深了。总之来说，就是能查的bug更多了，范围更广了。深切感觉到，技术上想要牛逼，全靠同行衬托……