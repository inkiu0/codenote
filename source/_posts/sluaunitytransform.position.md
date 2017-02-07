---
layout: post
title: "Slua的优化初探"
date: 2017-02-05 23:27
comments: true
tags: 
	- Lua
	- Slua
	- Unity
---
看了[UWA的blog](http://blog.uwa4d.com/archives/USparkle_Lua.html)以后，了解到了tolua对c#对象取值赋值的过程。并且发现slua对比tolua在test1里性能是tolua的6倍，所以很好奇slua做了哪些优化。  
## tolua取值赋值的过程  
![](/assets/blogImg/UnityShader/setcsharpobjectvalue.png)  
## slua的取值赋值过程  
1. 取出object的缓存  

```  
ObjectCache oc = ObjectCache.get(l);
```  
2. 取得c#中的index然后再ObjectCache里面查找  
3. cache.get(index, out o)，类似tolua中的Translator，都是从缓存中取出object  

```  
int index = LuaDLL.luaS_rawnetobj(l, p);
object o;
if (index != -1 && cache.get(index, out o))
{
        return o;
}
```    
4. 从栈中取出3个float组成Vector3  

```  
UnityEngine.Vector3 v;
checkType(l,2,out v); 
static public bool checkType(IntPtr l, int p, out Vector3 v)
{
	float x, y, z;
	if(LuaDLL.luaS_checkVector3(l, p, out x, out y, out z)!=0)
		throw new Exception(string.Format("Invalid vector3 argument at {0}", p));
	v = new Vector3(x, y, z);
	return true;
}
```  
5. 赋值给position  

```  
self.position=v;
```  
<!-- more -->
## slua相比tolua做了哪些优化  
* 赋值是在c#中完成的而不是取得object后把index构造成userdata后在lua中赋值
* 返回的object缓存起来了，没有频繁地GC  
## 还有那些优化的余地  
* lua给c#传递3个float而不是一个内容为{x,y,z}的table
