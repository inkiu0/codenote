---
layout: post
title: "LuaMemory的实现(二)"
date: 2017-03-17 20:46
comments: true
tags: 
	- Lua
	
	- Unity  
---
# LeakMemoryEditor的编写  
LuaMemory的核心是C库的编写，编写好C库以后，其他的周边的Editor的编写会比较简单，但是也最见策略。  
LuaMemory的编辑器主要有三大功能：
1. Snapshot的逻辑操作
>可以对两个或多个采样(Snapshot)之间使用逻辑操作，求交集和补集。这样可以很容易地缩小范围，更容易地定位泄露。  

2. 内存的树状结构
>把所有地内存按树状结构展现出来，根节点是_G或者Registry。如果某一项被多次引用，则把他挂在层级最浅地那个节点上， _G层级1为最浅。  

3. 数据存在另一个lua_state  
>如果把数据储存在当前地lua_state中，则要很小心地避免其对采样地影响。而在C中另起一个state来储存则不存在这个问题。  

<!-- more -->
## Snapshot地逻辑操作  
1. 交集(与操作)  
对2个或2个以上的Snapshot求交集，得到在这些Snapshot中都存在的内存项。并储存在一个新的Snapshot中，这个新的Snapshot也能进行逻辑操作和输出。  

2. 补集(求增量)  
对2个Snapshot A B求补集，得到在B中但是不在A中的内存项，并储存在一个新的Snapshot中。 

## 内存的树状结构  
1. 获取C库采集的数据  
C库采集完成的数据，按先进后出的顺序储存在另外一个lua_state中，我们通过getlog接口来获取储存在pL中的某次采样。假设我们采集了5次，那么pL的栈结构为：  

```c  
/*pL栈当前情况
--------------------
5 第一次采样
4 第二次采样
3 第三次采样
2 第四次采样
1 第五次采样
--------------------
*/
```  

获取到的Snapshot是一个table结构，key为该内存项的指针，value为一个table，储存了lua和c#需要的信息。  

```lua  
-- Snapshot中每一项的结构如下
Snapshot[pointer] = 
{
    ["count"] = number,
    ["type"] = string,
    ["level"] = number,
    ["parent"] = pointer,
    ["size"] = number/byte,
    ["infos"] = 
    {
        string,
        ...
    },
    ["child_list"] = 
    {
        pointer,
        ...
    }
}
```  

2. 构造树状结构
因为需要实现某个子集的树状结构，而且子集中的信息不全。那么我们就需要从这个子集的全集中查找信息，以填充子集的树状结构。  
- ABC...交集的父集为A  
- AB补集的父集为B  
>实现细节：每一项都用while一直查询到parent为nil，取得该项的完整调用。然后再从根节点开始，在Map中查找，没找到就新建一个节点，找到了就进入改节点，然后继续找自身的下一级。  

3. 传输调整后的数据给C#
把得到的数据做了处理以后生成一串jsonstring传给c#，然后c#解析出一个树状的Dictonary。c#中有一个递归的结构LeakItemNode，用来储存数据。LuaMemoryEditor持有根节点m_Root    
```c#  
//...
//一些从json中解析出来的数据
//...
List<LeakItemNode> childList = list<LeakItemNode>();
```  
## 另一个lua_state
在C中用一个静态变量储存起来的lua_state *pL，所有的采样数据都储存在其中。另外提供3个重要的接口用以操作该栈：
1. insert(idx)  
把栈顶的Snapshot插入idx处  

2. pop(num)  
出栈num个Snapshot  

3. clear  
清空栈