---
layout: post
title: "Lua迭代器和遍历"
date: 2017-01-16 00:00
comments: true
tags: 
	- Lua 
---
# 1. Lua遍历
在用Lua开发过程中经常会用到一些库, 最近看到了`Map`类中的`FindIf`方法。 感觉用的很巧妙, 遂分析学习一下。
`FindIf`方法我用原生的Lua实现了一遍, 去除了自定义的一些方法, 便于理解。  
```   
Map = {}
Map.data = {}
local hank = { name = "hank", weight = 75 }
local marry = { name = "marry", weight = 50 }
table.insert(Map.data, hank)
table.insert(Map.data, marry)

Map.Comparator = function(classNumber, equalValue)
	if classNumber == nil or equalValue == nil then
		print("参数错误! classNumber = " .. tostring(classNumber) .. "equalValue = " .. tostring(equalValue))
	end
	local func = function(equalValue)
		local comparator = function(data)
			return equalValue == data[classNumber]
		end
		return comparator;
	end
	return func(equalValue)
end

Map.Find = function(classNumber, equalValue)
	local comparator = Map.Comparator(classNumber, equalValue)
	local find = nil
	local index = -1;
	if Map.data ~= nil and #Map.data > 0 then
		for i = 1, #Map.data do
			if comparator(Map.data[i]) then
				find = Map.data[i]
				index = i;
				break;
			end
		end
	end
	return find, index;
end

-- 使用示例
print(Map.Find("name", "hank"))
print(Map.Find("name", "marry"))
print(Map.Find("weight", 50))
```  

<!-- more -->
使用[ideone](http://ideone.com/)运行以后输出:  
>Success	time: 0 memory: 2844 signal:0
  
table: 0x8573b10	1
  
table: 0x8573b68	2
  
table: 0x8573b68	2  

在这里推荐一下这个在线的ide。
## 1.1 疑惑  
### `Map.Comparator`返回的是什么?  
调用`Map.Comparator`后做了2件事:  
1. 声明一个局部变量`func`指向一个匿名函数`function(equalValue)`  
2. 调用`func`, 返回`func的返回值`。  
所以返回值为`local comparator`, 即`function(data)`。  

### `Map.Comparator`中的data是什么?  
data为调用`Map.Comparator的返回值`时传入的参数。比如Map.Find中`comparator(Map.data[i])`, Map.data[i]即为data参数。

# 2. Lua迭代器  
- ### next的使用
```   
local Map = {}
Map.data = {1, 2, 3}
Map.it = function()
    return next, Map.data, nil
end

-- 使用示例
for k,v in Map.it() do
  print(k..v)
end  
```  
使用[ideone](http://ideone.com/)运行以后输出:  
>Success	time: 0 memory: 2844 signal:0
11
22
33  

- ### 自定义迭代器 
```  
local Map = {}
Map.data = {1, 2, 3}
Map.iter = function(data, i)
    print("i: " .. i)
    i = i + 1
    local v = data[i]
    if v then
        return i, v  
    end
end
Map.v = 0
Map.it = function()
    return Map.iter, Map.data, Map.v
end

-- 使用示例
for k,v in Map.it() do
    print("Map.v: " .. Map.v)
    print(k..v)
end
```  
使用[ideone](http://ideone.com/)运行以后输出: 
>Success	time: 0 memory: 2844 signal:0
i: 0
Map.v: 0
11
i: 1
Map.v: 0
22
i: 2
Map.v: 0
33
i: 3 

- ### 疑惑  

1. `next`是什么  
next是Lua函数库中默认函数, 返回一个table的下一个值。
>next (table [, index])
  
运行程序来遍历表中的所有域。 第一个参数是要遍历的表，第二个参数是表中的某个键。 next 返回该键的下一个键及其关联的值。 如果用 nil 作为第二个参数调用 next 将返回初始键及其关联值。 当以最后一个键去调用，或是以 nil 调用一张空表时， next 返回 nil。 如果不提供第二个参数，将认为它就是 nil。 特别指出，你可以用 next(t) 来判断一张表是否是空的。
  
索引在遍历过程中的次序无定义， 即使是数字索引也是这样。 （如果想按数字次序遍历表，可以使用数字形式的 for 。）
  
当在遍历过程中你给表中并不存在的域赋值， next 的行为是未定义的。 然而你可以去修改那些已存在的域。 特别指出，你可以清除一些已存在的域  
引用自云风翻译的Lua5.3手册  
2. for in是怎么使用这三个返回值的  
`for in`的完整形式应该是`for k,v in f, s, v do`, 其中f是迭代器工厂函数, s为状态/数据, v为迭代器初始值。  
第一次调用工厂函数生成一个迭代器, iter = f(s, v)  
后面每次调用迭代器iter  
其中data, i, v作为闭包数据保存了下来, 每次调用时i+1。  
3. i三次迭代分别是什么值  
第一次i为Map.v  
第二次i为第一次的值保存下来后 + 1 = 2  
第三次i为第二次的值保存下来后 + 1 = 3