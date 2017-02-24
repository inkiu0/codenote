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
取得栈上索引处的值，并压入栈顶。lua_pushvalue(L, -2), 把 -2 位置的值移到栈顶。
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
--------------------
*/
```  