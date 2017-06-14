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
在参考了王亮的代码后，我也尝试优化了动画文件，发现一个很奇怪的现象。下图中Inspector中显示的大小并没有任何变化，但是文件大小和Profiler中的内存大小确实是减小了。那么Inspector中显示的Size是什么含义呢？  
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

|前后|FileSize|MemorySize|BlobSize|  
|:-:|:------:|:--------:|:------:|  
|优化前|11.75kb|8.75kb|2.8kb|  
|优化后|5.87kb|3.63kb|2.8kb|  
|对比|-50%|-58%|-0%|  

![](/assets/blogImg/Unity/OptimizeAnimationClip/animationclipinspector.png)  
红色框内即是BlobSize，在我的理解，FileSize是指文件在硬盘中占的大小，MemorySize是指把文件读进内存后占的大小，而BlobSize是从内存中的文件序列化出来的对象的二进制大小。所以只压缩精度的动画文件，BlobSize是不会有任何变化的，因为每个浮点数固定占32bit。而文件大小，还是内存大小和AB大小，压缩精度后都会变小。  

下面这次测试证实了我的想法，下图这个动画文件原来Inspector中Scale的值为4，即有Scale曲线。所以BlobSize减小了27%。  
![](/assets/blogImg/Unity/OptimizeAnimationClip/hasscalecurve.png)  

|前后|FileSize|MemorySize|BlobSize|  
|:-:|:------:|:--------:|:------:|  
|优化前|85.43kb|46.29kb|10.2kb|  
|优化后|23.34kb|18.69kb|7.4kb|  
|对比|-73%|-60|-27%|  

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
using System;
using System.Collections.Generic;
using UnityEngine;
using System.Reflection;
using UnityEditor;
using System.IO;

public class OptimizeAnimationClipTool
{
	const string _AnimationClipExtension = ".anim";
	static List<string> _FilesList = new List<string> ();
	static List<string> _Errors = new List<string>();
	static int _FileIndex = 0;

	[MenuItem("Assets/Animation/裁剪浮点数去除Scale")]
	public static void Optimize()
	{
		CollectFileWithExtension(out _FilesList, _AnimationClipExtension);
		if (_FilesList.Count > 0)
		{
			_FileIndex = 0;
			_Errors.Clear ();
			EditorApplication.update = ScanAnimationClip;
		}
	}

	private static void ScanAnimationClip()
	{
		string file = _FilesList[_FileIndex];
		bool isCancel = EditorUtility.DisplayCancelableProgressBar("优化AnimationClip", file, (float)_FileIndex / (float)_FilesList.Count);
		OptimizeAnimationClip (_FilesList[_FileIndex]);
		_FileIndex++;
		if (isCancel || _FileIndex >= _FilesList.Count)
		{
			EditorUtility.ClearProgressBar();
			Debug.Log(string.Format("--优化完成--    错误数量: {0}    总数量: {1}/{2}    错误信息↓:\n{3}\n----------输出完毕----------", _Errors.Count, _FileIndex, _FilesList.Count, string.Join(string.Empty, _Errors.ToArray())));
			Resources.UnloadUnusedAssets();
			GC.Collect();
			AssetDatabase.SaveAssets();
			EditorApplication.update = null;
			_FilesList.Clear();
			_FileIndex = 0;
		}
	}

	static void OptimizeAnimationClip(string path)
	{
		AnimationClip aniClip = AssetDatabase.LoadAssetAtPath<AnimationClip> (path);
		if(aniClip != null)
		{
			try
			{
				//clear scale curve
				foreach(EditorCurveBinding curveBinding in AnimationUtility.GetCurveBindings(aniClip))
				{
					string name = curveBinding.propertyName.ToLower();
					if(name.Contains("scale"))
					{
						AnimationUtility.SetEditorCurve(aniClip, curveBinding, null);
					}
				}

				//float cut to f3
				AnimationClipCurveData[] curveDatas = null;
				curveDatas = AnimationUtility.GetAllCurves(aniClip);
				Keyframe key;
				Keyframe[] keyFrames;
				for(int i = 0; i < curveDatas.Length; ++i)
				{
					AnimationClipCurveData curveData = curveDatas[i];
					if(curveData.curve == null || curveData.curve.keys == null)
					{
						string error = string.Format("AnimationClipCurveData {0} don't have any curve;Animation name {1}", curveData, path);
						_Errors.Add(error + "\n");
						Debug.LogWarning(error);
					}
					keyFrames = curveData.curve.keys;
					for(int k = 0; k < keyFrames.Length; k++)
					{
						key = keyFrames[k];
						key.value = float.Parse(key.value.ToString("F3"));
						key.inTangent = float.Parse(key.inTangent.ToString("F3"));
						key.outTangent = float.Parse(key.outTangent.ToString("F3"));
						keyFrames[k] = key;
					}
					curveData.curve.keys = keyFrames;
					aniClip.SetCurve(curveData.path, curveData.type, curveData.propertyName, curveData.curve);
				}
			}
			catch(System.Exception e)
			{
				string error = string.Format ("CompressAnimationClip Failed ! animationPath = {0} error = {1}", path, e);
				_Errors.Add(error + "\n");
				Debug.LogError(error);
			}

		}
	}

	private static void CollectFileWithExtension(out List<string> files, string extension)
	{
		files = new List<string> ();
		List<string> _TopLvPath = GetTopLvSelected();
		string path = string.Empty;
		for(int i = 0; i < _TopLvPath.Count; i++)
		{
			path = _TopLvPath[i];
			if (File.Exists(path))
			{
				if (Path.GetExtension(path) == extension && !files.Contains(path))
					files.Add(path);
			}
			else
			{
				CollectFile (ref files, path, extension);
			}
		}
	}

	private static void CollectFile(ref List<string> files, string folder, string extension, bool recursive = true)
	{
		folder = AppendSlash (folder);
		DirectoryInfo dir = new DirectoryInfo(folder);
		foreach (var file in dir.GetFiles())
		{
			string fpath = folder + file.Name;
			if (file.Extension == extension && !files.Contains(fpath))
				files.Add(fpath);
		}
		if (recursive)
		{
			foreach (var sub in dir.GetDirectories())
			{
				CollectFile(ref files, folder + sub.Name, extension, recursive);
			}
		}
	}

	private static List<string> GetTopLvSelected()
	{
		UnityEngine.Object[] objs = Selection.GetFiltered(typeof(object), SelectionMode.Assets);
		List<string> _SelectedPath = new List<string>();
		List<string> _TopLvPath = new List<string>();
		bool _IsTopLv = true;
		string path = string.Empty;
		for (int i = 0; i < objs.Length; i++)
		{
			_SelectedPath.Add(AssetDatabase.GetAssetPath(objs[i]));
		}
		for (int i = 0; i < _SelectedPath.Count; i++)
		{
			path = _SelectedPath[i];
			_IsTopLv = true;
			for (int k = 0; k < _SelectedPath.Count; k++)
			{
				if (i != k && path.Contains(_SelectedPath[k]))
				{
					_IsTopLv = false;
					break;
				}
			}
			if (_IsTopLv)
				_TopLvPath.Add(path);
		}
		return _TopLvPath;
	}

	public static string AppendSlash(string path)
	{
		if (path == null || path == "")
			return "";
		int idx = path.LastIndexOf('/');
		if (idx == -1)
			return path + "/";
		if (idx == path.Length - 1)
			return path;
		return path + "/";
	}
}
```  