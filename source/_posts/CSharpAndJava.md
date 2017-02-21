---
layout: post
title: "Unity和Java交互"
date: 2016-12-08 04:42
comments: true
tags: 
	- Unity  
    - Java  
    - C#  
---
#CSharpToJava  
##1. new Java类
```  
 static AndroidJavaClass _UnityPlayer;
 static AndroidJavaClass unityPlayer
 {
     get
     {
         if (_UnityPlayer == null)
             _UnityPlayer = new AndroidJavaClass("com.unity3d.player.UnityPlayer");
         return _UnityPlayer;
     }
 }
```
##2. 调用Java函数  
- 调用无参无返回函数
```
//调用静态函数为CallStatic
androidJavaClass.Call("methodName");
```
- 调用带参数有返回函数
```
//返回值为int则为Call<int>
//返回值为Java自定义的数据类型, 比如Java类。则为CallStatic<AndroidJavaObject>
//调用静态函数为CallStatic
String str = androidJavaClass.Call<String>("methodName", param1, param2);
```  

<!-- more -->
##3. 访问Java类的属性
- 访问普通属性
```  
//访问int属性则为Get<int>
String str = androidJavaClass.Get<String>("PropertyName")
```  
- 访问静态属性
```
//示例如下:
unityPlayer.GetStatic<AndroidJavaObject>("currentActivity")
```  

#JavaToCSharp  
## 1. Java调用CSharp
**需要在场景中创建一个名为AScript的GameObject**
```
//调用场景中的GameObject:AScript挂载的脚本AScript中的BMethod，传入参数param。
UnityPlayer.UnitySendMessage("AScript","BMethod", param); 
```

