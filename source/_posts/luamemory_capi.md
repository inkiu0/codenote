---
layout: post
title: "LuaMemory的实现(一)"
date: 2017-02-24 14:02
comments: true
tags: 
	- Lua
	
	- Unity  
---
# LuaMemory的实现(一) LuaC库的编写  
### lua_next(lua_State *L, int idx)  
lua_next 先把 table ( lua 栈中 idx 所指的 table )的下一索引弹出，再把 table 当前索引的值弹出。没有数据了返回0，有数据返回非0。  
```lua 
local test_table = 
{
    ["a"] = 1,
    ["b"] = 2,
    ["c"] = "ccc"
} 
```  
```c  
lua_pushnil(L);

/*lua栈当前情况
--------------------
-1 nil
-2 test_table
--------------------
*/

while(lua_next(L, -2) != 0)
{

/*lua栈当前情况
--------------------
-1 value
-2 key
-3 test_table
--------------------
*/

    if(lua_isnumber(L,-2))
        cout<<"key:"<<lua_tonumber(L,-2)<<'\t';
    else if(lua_isstring(L,-2))
        cout<<"key:"<<lua_tostring(L,-2)<<'\t';
    if(lua_isnumber(L,-1))
        cout<<"value:"<<lua_tonumber(L,-1)<<endl;
    else if(lua_isstring(L,-1))
        cout<<"value:"<<lua_tostring(L,-1)<<endl;

/*lua栈当前情况
--------------------
-1 value
-2 key
-3 test_table
--------------------
*/

    lua_pop(L, 1);

/*lua栈当前情况
--------------------
-1 key
-2 test_table
--------------------
*/

}

/*lua栈当前情况
--------------------
-1 test_table
--------------------
*/
```  
<!-- more -->
需要注意的是 **lua_next 先把 table 的下一索引弹出** , 下一索引是相对于上一个索引的。所以我们在循环开始的时候入栈了nil，表示我们是从头开始遍历table。而在循环中，每次都保留了上次的key。直到 lua_next 返回0, 表示 table 已经遍历完, 此时退出循环, 最后一个 key 才出栈。  
### lua_newtable  
创建一个 table 并压入栈顶。  
### lua_rawgeti(L, LUA_REGISTRYINDEX, LUA_RIDX_GLOBALS)  
**LUA_REGISTRYINDEX** 是lua注册表的伪索引, **LUA_RIDX_GLOBALS** 是lua中全局环境即 **_G** 在注册表中的索引。这行代码获取得了 **_G** 并压入栈顶。  
### lua_pushvalue  
复制栈上索引处的值，并压入栈顶。lua_pushvalue(L, -2), 把 -2 位置的值复制一份压入到栈顶。
```c  
/*lua栈当前情况
--------------------
-1 level_111
-2 level_222
--------------------
*/

lua_pushvalue(L, -2);

/*lua栈当前情况
--------------------
-1 level_222
-2 level_111
-3 level_222
--------------------
*/
```  
### lua_insert  
将栈顶的元素压入索引处，索引处到栈顶的元素依次往上移，栈的大小不变。  
```c  
for(int i = 1; i < 6; i++)
{
    lua_pushnumber(L, i);
}

/*lua栈当前情况
--------------------
-1 5
-2 4
-3 3
-4 2
-5 1
--------------------
*/

lua_insert(L, 2);

/*lua栈当前情况
--------------------
-1 4
-2 3
-3 2
-4 5
-5 1
--------------------
*/
```  

## Lua _G内存结构
### Lua内存结构  
Lua内以下类型都是属于GCObject，是有gc方法的。
- table
    - [table]self
    - [table]metatable
    - [object]kv
- function
    - [object]upvalue
    - [lcl/ccl]self
- thread
    - [object]stack
- userdata
    - [table]metatable
    - [table]uservalue
- string
    - [TString]self
### print  
在编写统计代码的时候，我们时常会用到测试代码。但是需要注意的是，有些测试代码本身就会产生一些内存，以误导我们的测试。比如print，我猜测是因为每次print都产生了一个函数上值。所以我们在print后需要手动 **collectgarbage()** 。  

### table
#### table可能引用的内容
1. metatable
>metatable即该table的原表，一定是一个table。

2. 非weak的key
>key可以是任意类型  

3. 非weak的value
>value可以是任意类型

#### table大小计算
```c  
//单位为byte
size = sizeof(Table) + sizeof(TValue) * t->sizearray + sizeof(Node) * sizenode(t)
```     
- sizeof(Table)  = 56  
- sizeof(TValue) = 16  
- sizeof(Node)   = 32  
- sizenode(t)  
> table的hashtable部分长度，值总是2的幂，a power of 2。hashtable.len为0时，sizenode为1，hashtable.len为1时，sizenode为1。所以想要分辨出0和1需要判断一下t->lastfress，定义宏如下：  

```c  
#define isdummy(t)		((t)->lastfree == NULL)
```  
### function
#### function可能引用的内容  
1. upvalue  
>upvalue可以是任意类型  

2. 本身  
>function本身可能为Luafunction和Cfunction  


#### function大小计算  
在lua中function分为 **LColsure** 和 **CColsure**，即Lua函数和C函数。  
```c  
ttisCclosure(o) ? sizeCclosure(cl->c.nupvalues) : sizeLclosure(cl->l.nupvalues)
//sizeCclosure(n) = sizeof(CClosure) + sizeof(TValue)* (n-1)
//sizeLclosure(n) = sizeof(LClosure) + sizeof(TValue)* (n-1)
```  

- sizeof(CClosure) = 48  
- sizeof(LClosure) = 40  

### thread
#### thread可能引用的类型
1. stack  
>stack中的值，以及这些值的local变量  

#### thread大小计算  
lua中的thread即为Coroutine  
```c  
(sizeof(lua_State) + sizeof(TValue) * th->stacksize + sizeof(CallInfo) * th->nci);  
```  

### userdata
#### userdata可能引用的类型  
1. metatable
>userdata可能有自己的元表  

2. uservalue  
>userdata的内容，是一个table类型。

#### userdata大小计算  
```c  
sizeudata(uvalue(o));
```  

### string
#### string大小计算
```c  
//string分为LUA_TLNGSTR和LUA_TSHRSTR
case LUA_TSHRSTR:
    sizelstring(ts->shrlen);
case LUA_TLNGSTR:
    sizelstring(ts->u.lnglen);
```  