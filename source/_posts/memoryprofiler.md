---
layout: post
title: "LuaMemoryProfiler的实现"
date: 2017-02-24 14:02
comments: true
tags: 
	- Lua
	
	- Unity  
---
## 编写C代码  
### lua_newtable  
创建一个 table 并压入栈顶。  
### lua_rawgeti(L, LUA_REGISTRYINDEX, LUA_RIDX_GLOBALS)  
**LUA_REGISTRYINDEX** 是lua注册表的伪索引, **LUA_RIDX_GLOBALS** 是lua中全局环境即 **_G** 在注册表中的索引。这行代码获取得了 **_G** 并压入栈顶。
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
lua_pop(L, 1);

/*lua栈当前情况
--------------------
-1 test_table
--------------------
*/
```  
需要注意的是 **lua_next 先把 table 的下一索引弹出** , 下一索引是相对于上一个索引的。所以我们在循环开始的时候入栈了nil，表示我们是从头开始遍历table。而在循环中，每次都保留了上次的key。直到 lua_next 返回0, 表示 table 已经遍历完, 此时退出循环, 最后一个 key 才出栈。  
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

## getsize  
### print  
在编写统计代码的时候，我们市场会用到测试代码。但是需要注意的是，有些测试代码本身就会产生一些内存，以误导我们的测试。比如print，我猜测是因为每次print都产生了一个函数上值。所以我们在print后需要手动 **collectgarbage()** 两次。
### table
```c  
//单位为byte
size = sizeof(Table) + sizeof(TValue) * t->sizearray + sizeof(Node) * sizenode(t)
```   
在macmini中:  
- sizeof(Table)  = 56  
- sizeof(TValue) = 16  
- sizeof(Node)   = 32  
- sizenode(t)  
    table的hashtable部分长度，值总是2的幂，a power of 2。hashtable.len为0时，sizenode为1，hashtable.len为1时，sizenode为1。所以想要分辨出0和1需要判断一下t->lastfress，定义宏如下：
```c  
#define isdummy(t)		((t)->lastfree == NULL)
```  
### function  
在lua中function分为 **LColsure** 和 **CColsure**，即Lua函数和C函数。  
```c  
ttisCclosure(o) ? sizeCclosure(cl->c.nupvalues) : sizeLclosure(cl->l.nupvalues)

//sizeCclosure(n) = sizeof(CClosure) + sizeof(TValue)* (n-1)
//sizeLclosure(n) = sizeof(LClosure) + sizeof(TValue)* (n-1)
```  
- sizeof(CClosure) = 48  
- sizeof(LClosure) = 40  
### thread  
lua中的thread即为Coroutine  
```lua  
size = sizeof(lua_State) + sizeof(TValue) * th->stacksize + sizeof(CallInfo) * th->nci
```  
- sizeof(lua_State) = 208
- sizeof(CallInfo) = 72  
```c  
/*Current Lua Stack
--------------------
-1 TABLE
--------------------
*/

int n = 1;
lua_getfield(dL, -1, "infos");

/*Current Lua Stack
--------------------
-1 infos(table)
-2 TABLE
--------------------
*/

lua_pushnil(dL);

/*Current Lua Stack
--------------------
-1 nil
-2 infos(table)
-3 TABLE
--------------------
*/

while (lua_next(dL, -2) != 0)
{

/*Current Lua Stack
--------------------
-1 value
-2 key
-3 infos(table)
-4 TABLE
--------------------
*/

    str = lua_tostring(dL, -1);
    lua_pushstring(L, str);
    lua_rawseti(L, -2, n);
    ++n;
    lua_pop(dL, 1);

/*Current Lua Stack
--------------------
-1 key
-2 infos(table)
-3 TABLE
--------------------
*/

}

/*Current Lua Stack
--------------------
-1 infos(table)
-2 TABLE
--------------------
*/
```  