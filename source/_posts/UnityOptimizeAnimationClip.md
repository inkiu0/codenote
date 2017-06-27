---
layout: post
title: "Unity动画文件优化探究"
date: 2017-06-14 20:20
comments: true
tags: 
	- Unity 
---

## 主要思路  

1. 压缩浮点数精度  
2. 去除scale曲线  

<!-- more -->  
## 各Size含义  
在参考了王亮的代码后，我也尝试优化了动画文件，发现一个很奇怪的现象。下图中Inspector中显示的大小并没有任何变化，但是文件大小和Profiler中的内存大小确实是减小了。那么Inspector中显示的Size是什么含义呢？  
下图中的动画文件不包含Scale曲线，所以这次优化中只压缩了浮点数精度。  
![](/assets/blogImg/Unity/OptimizeAnimationClip/noscalecurve.png)  
我分别对比了动画文件优化前和优化后的大小  

- FileSize
    - FileInfo.Length取得的文件大小  
    - 可以在操作系统的文件系统中看到  
- MemorySize
    - Profiler.GetRuntimeMemorySize取得的内存大小  
    - 可以在Profiler中通过采样看到  
- BlobSize
    - 反射取得的AnimationClipStats.size二进制大小  
    - 显示在AnimationClip的Inspector的面板上  

|前后|FileSize|Editor MemorySize|真机MemorySize|BlobSize|  
|:-:|:------:|:---------------:|:-----------:|:------:|  
|优化前|11.75kb|8.75kb|3.1kb|2.8kb|  
|优化后|5.87kb|3.63kb|3.1kb|2.8kb|  
|对比|-50%|-58%|-0%|-0%|  

![](/assets/blogImg/Unity/OptimizeAnimationClip/animationclipinspector.png)  
红色框内即是BlobSize，在我的理解，FileSize是指文件在硬盘中占的大小，BlobSize是从文件反序列化出来的对象的二进制大小。Editor下的MemorySize不仅有序列化后的内存大小，还维护了一份原文件的内存。就像我们在Editor下加载一张Texture内存是双份一样，而真机下就约等于BlobSize。真机下的MemorySize和Inspector里的BlobSize非常接近，BlobSize可以认为是真机上的内存大小，这个大小更有参考意义。  

下面这次测试证实了我的想法，下图这个动画文件原来Inspector中Scale的值为4，即有Scale曲线。所以BlobSize减小了27%。  
![](/assets/blogImg/Unity/OptimizeAnimationClip/hasscalecurve.png)  

|前后|FileSize|Editor MemorySize|真机MemorySize|BlobSize|  
|:-:|:------:|:---------------:|:-----------:|:------:|  
|优化前|85.43kb|46.29kb|10.2kb|  
|优化后|23.34kb|18.69kb|7.4kb|  
|对比|-73%|-60|-27%|  

## Curve减少导致内存减小  
从上面的实验可以看出来，只动画文件的压缩精度，没有引起Curve减少。BlobSize是不会有任何变化的，因为每个浮点数固定占32bit。而文件大小、AB大小、Editor下的内存大小，压缩精度后不管有没有引起Curve的变化，都会变小。  
裁剪动画文件的精度，意味着点的位置发生了变化，所以Constant Curve和Dense Curve的数量也有可能发生变化。由于是裁剪精度所以动画的点更稀疏了，而连续相同的点更多了。所以Dense Curve是减少了，Constant Curve是增多了，总的内存是减小了。  

Constant Curve只需要最左边的点就可以描述一个曲线段   
![](/assets/blogImg/Unity/OptimizeAnimationClip/ConstantCurve.png)

## 总结  
隔壁项目组对他们项目中所有的动画文件都进行了优化。其中文件大小从820m->225, ab大小从72m->64m, 内存大小从50m->40m。总的来说动画文件的scale越多优化越明显。  

## 取BlobSize代码  
```csharp  
AnimationClip aniClip = AssetDatabase.LoadAssetAtPath<AnimationClip> (path);
var fileInfo = new System.IO.FileInfo(path);
Debug.Log(fileInfo.Length);//FileSize
Debug.Log(Profiler.GetRuntimeMemorySize (aniClip));//MemorySize

Assembly asm = Assembly.GetAssembly(typeof(Editor));
MethodInfo getAnimationClipStats = typeof(AnimationUtility).GetMethod("GetAnimationClipStats", BindingFlags.Static | BindingFlags.NonPublic);
Type aniclipstats = asm.GetType("UnityEditor.AnimationClipStats");
FieldInfo sizeInfo = aniclipstats.GetField ("size", BindingFlags.Public | BindingFlags.Instance);

var stats = getAnimationClipStats.Invoke(null, new object[]{aniClip});
Debug.Log(EditorUtility.FormatBytes((int)sizeInfo.GetValue(stats)));//BlobSize
```  

## 工具代码  
最后附上工具的代码和简要使用说明，选中想要优化的文件夹或文件，右键Animation->裁剪浮点数去除Scale。  
![](/assets/blogImg/Unity/OptimizeAnimationClip/OptimizeAnimationClip.gif)  

```csharp   
//****************************************************************************
//
//  File:      OptimizeAnimationClipTool.cs
//
//  Copyright (c) SuiJiaBin
//
// THIS CODE AND INFORMATION IS PROVIDED "AS IS" WITHOUT WARRANTY OF
// ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO
// THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A
// PARTICULAR PURPOSE.
//
//****************************************************************************
using System;
using System.Collections.Generic;
using UnityEngine;
using System.Reflection;
using UnityEditor;
using System.IO;

namespace EditorTool
{
	class AnimationOpt
	{
		static Dictionary<uint,string> _FLOAT_FORMAT;
		static MethodInfo getAnimationClipStats;
		static FieldInfo sizeInfo;
		static object[] _param = new object[1];

		static AnimationOpt ()
		{
			_FLOAT_FORMAT = new Dictionary<uint, string> ();
			for (uint i = 1; i < 6; i++) {
				_FLOAT_FORMAT.Add (i, "f" + i.ToString ());
			}
			Assembly asm = Assembly.GetAssembly (typeof(Editor));
			getAnimationClipStats = typeof(AnimationUtility).GetMethod ("GetAnimationClipStats", BindingFlags.Static | BindingFlags.NonPublic);
			Type aniclipstats = asm.GetType ("UnityEditor.AnimationClipStats");
			sizeInfo = aniclipstats.GetField ("size", BindingFlags.Public | BindingFlags.Instance);
		}

		AnimationClip _clip;
		string _path;

		public string path { get{ return _path;} }

		public long originFileSize { get; private set; }

		public int originMemorySize { get; private set; }

		public int originInspectorSize { get; private set; }

		public long optFileSize { get; private set; }

		public int optMemorySize { get; private set; }

		public int optInspectorSize { get; private set; }

		public AnimationOpt (string path, AnimationClip clip)
		{
			_path = path;
			_clip = clip;
			_GetOriginSize ();
		}

		void _GetOriginSize ()
		{
			originFileSize = _GetFileZie ();
			originMemorySize = _GetMemSize ();
			originInspectorSize = _GetInspectorSize ();
		}

		void _GetOptSize ()
		{
			optFileSize = _GetFileZie ();
			optMemorySize = _GetMemSize ();
			optInspectorSize = _GetInspectorSize ();
		}

		long _GetFileZie ()
		{
			FileInfo fi = new FileInfo (_path);
			return fi.Length;
		}

		int _GetMemSize ()
		{
			return Profiler.GetRuntimeMemorySize (_clip);
		}

		int _GetInspectorSize ()
		{
			_param [0] = _clip;
			var stats = getAnimationClipStats.Invoke (null, _param);
			return (int)sizeInfo.GetValue (stats);
		}

		void _OptmizeAnimationScaleCurve ()
		{
			if (_clip != null) {
				//去除scale曲线
				foreach (EditorCurveBinding theCurveBinding in AnimationUtility.GetCurveBindings(_clip)) {
					string name = theCurveBinding.propertyName.ToLower ();
					if (name.Contains ("scale")) {
						AnimationUtility.SetEditorCurve (_clip, theCurveBinding, null);
						Debug.LogFormat ("关闭{0}的scale curve", _clip.name);
					}
				} 
			}
		}

		void _OptmizeAnimationFloat_X (uint x)
		{
			if (_clip != null && x > 0) {
				//浮点数精度压缩到f3
				AnimationClipCurveData[] curves = null;
				curves = AnimationUtility.GetAllCurves (_clip);
				Keyframe key;
				Keyframe[] keyFrames;
				string floatFormat;
				if (_FLOAT_FORMAT.TryGetValue (x, out floatFormat)) {
					if (curves != null && curves.Length > 0) {
						for (int ii = 0; ii < curves.Length; ++ii) {
							AnimationClipCurveData curveDate = curves [ii];
							if (curveDate.curve == null || curveDate.curve.keys == null) {
								//Debug.LogWarning(string.Format("AnimationClipCurveData {0} don't have curve; Animation name {1} ", curveDate, animationPath));
								continue;
							}
							keyFrames = curveDate.curve.keys;
							for (int i = 0; i < keyFrames.Length; i++) {
								key = keyFrames [i];
								key.value = float.Parse (key.value.ToString (floatFormat));
								key.inTangent = float.Parse (key.inTangent.ToString (floatFormat));
								key.outTangent = float.Parse (key.outTangent.ToString (floatFormat));
								keyFrames [i] = key;
							}
							curveDate.curve.keys = keyFrames;
							_clip.SetCurve (curveDate.path, curveDate.type, curveDate.propertyName, curveDate.curve);
						}
					}
				} else {
					Debug.LogErrorFormat ("目前不支持{0}位浮点", x);
				}
			}
		}

		public void Optimize (bool scaleOpt, uint floatSize)
		{
			if (scaleOpt) {
				_OptmizeAnimationScaleCurve ();
			}
			_OptmizeAnimationFloat_X (floatSize);
			_GetOptSize ();
		}

		public void Optimize_Scale_Float3 ()
		{
			Optimize (true, 3);
		}

		public void LogOrigin ()
		{
			_logSize (originFileSize, originMemorySize, originInspectorSize);
		}

		public void LogOpt ()
		{
			_logSize (optFileSize, optMemorySize, optInspectorSize);
		}

		public void LogDelta ()
		{

		}

		void _logSize (long fileSize, int memSize, int inspectorSize)
		{
			Debug.LogFormat ("{0} \nSize=[ {1} ]", _path, string.Format ("FSize={0} ; Mem->{1} ; inspector->{2}",
				EditorUtility.FormatBytes (fileSize), EditorUtility.FormatBytes (memSize), EditorUtility.FormatBytes (inspectorSize)));
		}
	}

	public class OptimizeAnimationClipTool
	{
		static List<AnimationOpt> _AnimOptList = new List<AnimationOpt> ();
		static List<string> _Errors = new List<string>();
		static int _Index = 0;

		[MenuItem("Assets/Animation/裁剪浮点数去除Scale")]
		public static void Optimize()
		{
			_AnimOptList = FindAnims ();
			if (_AnimOptList.Count > 0)
			{
				_Index = 0;
				_Errors.Clear ();
				EditorApplication.update = ScanAnimationClip;
			}
		}

		private static void ScanAnimationClip()
		{
			AnimationOpt _AnimOpt = _AnimOptList[_Index];
			bool isCancel = EditorUtility.DisplayCancelableProgressBar("优化AnimationClip", _AnimOpt.path, (float)_Index / (float)_AnimOptList.Count);
			_AnimOpt.Optimize_Scale_Float3();
			_Index++;
			if (isCancel || _Index >= _AnimOptList.Count)
			{
				EditorUtility.ClearProgressBar();
				Debug.Log(string.Format("--优化完成--    错误数量: {0}    总数量: {1}/{2}    错误信息↓:\n{3}\n----------输出完毕----------", _Errors.Count, _Index, _AnimOptList.Count, string.Join(string.Empty, _Errors.ToArray())));
				Resources.UnloadUnusedAssets();
				GC.Collect();
				AssetDatabase.SaveAssets();
				EditorApplication.update = null;
				_AnimOptList.Clear();
				_cachedOpts.Clear ();
				_Index = 0;
			}
		}

		static Dictionary<string,AnimationOpt> _cachedOpts = new Dictionary<string, AnimationOpt> ();

		static AnimationOpt _GetNewAOpt (string path)
		{
			AnimationOpt opt = null;
			if (!_cachedOpts.ContainsKey(path)) {
				AnimationClip clip = AssetDatabase.LoadAssetAtPath<AnimationClip> (path);
				if (clip != null) {
					opt = new AnimationOpt (path, clip);
					_cachedOpts [path] = opt;
				}
			}
			return opt;
		}

		static List<AnimationOpt> FindAnims()
		{
			string[] guids = null;
			List<string> path = new List<string>();
			List<AnimationOpt> assets = new List<AnimationOpt> ();
			UnityEngine.Object[] objs = Selection.GetFiltered(typeof(object), SelectionMode.Assets);
			if (objs.Length > 0)
			{
				for(int i = 0; i < objs.Length; i++)
				{
					if (objs [i].GetType () == typeof(AnimationClip))
					{
						string p = AssetDatabase.GetAssetPath (objs [i]);
						AnimationOpt animopt = _GetNewAOpt (p);
						if (animopt != null)
							assets.Add (animopt);
					}
					else
						path.Add(AssetDatabase.GetAssetPath (objs [i]));
				}
				guids = AssetDatabase.FindAssets (string.Format ("t:{0}", typeof(AnimationClip).ToString().Replace("UnityEngine.", "")), path.ToArray());
			}
			else
			{
				guids = AssetDatabase.FindAssets (string.Format ("t:{0}", typeof(AnimationClip).ToString().Replace("UnityEngine.", "")));
			}
			for(int i = 0; i < guids.Length; i++)
			{
				string assetPath = AssetDatabase.GUIDToAssetPath (guids [i]);
				AnimationOpt animopt = _GetNewAOpt (assetPath);
				if (animopt != null)
					assets.Add (animopt);
			}
			return assets;
		}
	}
}
```  