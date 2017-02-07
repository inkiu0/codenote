---
layout: post
title: "AssetBundle冗余检测器"
date: 2016-11-11 00:00
comments: true
tags: 
	- Unity 
---
最近我们项目快上线了，把项目提交到了一个专业做Unity项目优化的网站。——[uwa](www.uwa4d.com)

他们号称没有不存在资源冗余的项目，我们提交以后确实发现了一些冗余资源。但是他们网站有2个缺陷：

- 免费用户一个月只能检测2次

 - 不自由

 - 付费用户6600/季度

- 需要上传自己项目的所有AB文件

 - 不安全

而且，我仔细想了下，这里面的技术其实不是很复杂。就衍生了一个自己写一个小插件的想法，然后**ABRedundancyChecker**就诞生了。

<!-- more -->
## 一、插件介绍

1. 我把AB包所有的资源分为两类

 - 本包资源

 - 依赖包资源

2. 该插件把每个AB包的本包资源都列举出来，然后统计这些资源是否有重复，重复则为冗余。

3. 插件github仓库地址：https://github.com/inkiu0/ABRedundancyChecker

4. 喜欢的赏颗星星

## 二、ABRedundancyChecker使用方法

### 1. 修改脚本参数

1. 把以下参数改成自己想要的:

```
/// <summary>

/// AB文件名匹配规则

/// </summary>

public string searchPattern = "*.ab";

/// <summary>

/// 冗余资源类型白名单

/// </summary>

public List<Type> assetTypeList = new List<Type> { typeof(Material), typeof(Texture2D), typeof(AnimationClip),

typeof(AudioClip), typeof(Sprite), typeof(Shader), typeof(Font), typeof(Mesh) };

/// <summary>

/// 输出路径

/// </summary>

public string outPath = Environment.GetFolderPath(Environment.SpecialFolder.DesktopDirectory);

/// <summary>

/// AB文件存放路径，会从这个文件夹下递归查找符合查找规则searchPattern的文件。

/// </summary>

public string abPath = "Assets/StreamingAssets";

[MenuItem("AB冗余检测/AB检测")]
```

### 2. 开始使用

1. 将`ABRedundancyChecker.cs`放在Unity项目的Editor目录下

2. 将所有打包好的AssetBundle文件放在`abPath`目录下

3. 点击菜单栏`AB冗余检测`->`AB检测`

4. 喝一杯茶

 - 250MB的AB文件(1600个文件)检测时间为2分钟

5. 打开输出到目标目录的MarkDown文件

### 3. 输出的MarkDown形如
![](/assets/blogImg/UnityShader/QQ截图20161111103735.png)

