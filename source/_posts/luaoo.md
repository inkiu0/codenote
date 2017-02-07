---
layout: post
title: "Lua OO 实现 "
date: 2016-12-09 21:49
comments: true
tags: 
	- Lua 
---
# Lua OO 实现
##1. 类的实现
```
testClass =
{
    id,
    num
};

testClass.__index = testClass

function testClass:new(id, num)
    local self = {}
    setmetatable(self, testClass)
    self.id = id
    self.num = num
    return self
end

function testClass:PrintNum()
    print("num is " .. self.num)
end
```
简单使用
```
local obj = testClass:new(10, 11)
obj:PrintNum()
```
得到输出`num is 11`
<!-- more -->
##2. 类的继承
Class A
```
A = {}

function A:new(o)
    o = o or {}
    //self指向o
    setmetatable(o, self)
    //__index设为A
    self.__index = self
    return o;
end

function A:PrintClassName()
    print("this is class A")
end

function A:AMethod()
    print("this is AMethod !")
end
```
Class B继承Class A
```
B = A:new()

function B:PrintClassName()
    print("this is class B !")
end
```
使用示例
```
local obj = B:new()
obj:PrintClassName()
obj:AMethod()
```
输出

`this is class B !`

`this is AMethod !`