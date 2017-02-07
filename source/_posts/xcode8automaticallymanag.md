---
layout: post
title: "Xcode8自动打包去掉AutoMaticallyManageSigning"
date: 2016-10-31 00:00
comments: true
tags: 
	- Xcode打包 
---
最近项目里的自动打包脚本不能用了，一直提示
```
Check dependencies
Signing for "Unity-iPhone" requires a development team. Select a development team in the project editor.
Code signing is required for product type 'Application' in SDK 'iOS 10.0'

** BUILD FAILED **

The following build commands failed:
 Check dependencies
(1 failure)
```
Google了一下发现是xcode新的自动管理签名机制的问题，你要不使用AutoMatic自动管理，要不使用Manual手动指定证书的模式。

无奈我们打包的时候只有证书，没有对应的AppleID，所以自动管理的就用不了。但是UnityBuild出来的Xcode项目是自动勾选`Auto MaticallyManageSigning`的，而且Xcode也没有支持用命令行设置这个值。那这样的话，我们每次打包出Xcode项目的时候需要手动点一下，这就失去打包工具的意义了。

无奈之下，找到一个办法解决这个问题。  
<!-- more -->
我先用UnityBuild出一个干净的Xcode项目，然后把项目传到Git。然后手动点一下`BuildSetting`里的`Auto MaticallyManageSigning`，去掉勾选。然后查看下`diff`，当然其中有很多修改。
主要修改在Unity-iPhone.xcodeproj\/project.pbxproj，在Finder里想打开该文件应选中**Unity-iPhone.xcodeproj**右键**显示包内容**。  

**project.pbxproj**内也有很多修改，重要的修改其实只有几行，主要是在这个地方加上ProvisioningStyle = Manual。

修改前：

```
TargetAttributes = {
    5623C57217FDCB0800090B9E /* Unity-iPhone Tests */ = {
        TestTargetID = 1D6058900D05DD3D006BFB54 /* Unity-iPhone */;
    };
};
```

修改后：

```
TargetAttributes = {
    5623C57217FDCB0800090B9E /* Unity-iPhone Tests */ = {
        TestTargetID = 1D6058900D05DD3D006BFB54 /* Unity-iPhone */;
    };
    1D6058900D05DD3D006BFB54 = {
        ProvisioningStyle = Manual;
    };
};
```
**需要注意的是**从来没有用Xcode打开并且操作过的project.pbxproj是不存在`ProvisioningStyle`字段的，所以应追加3行。但是打开并操作过的项目是存在`ProvisioningStyle`字段的，这个时候如果想用脚本修改该值应直接替换该值
```
sed -i "" s/'ProvisioningStyle = Automatic;'/'DevelopmentTeam = None;ProvisioningStyle = Manual;'/g project.pbxprojPath
```
因为我们是全自动的打包过程，正常流程是不用打开xcode项目的，所以我准备用`sed`在指定文本下追加3行，并且要获取上一次匹配到的`TestTargetID`，我写到3点还没写出来……实在不会。所以我用python实现了这个操作，附上python脚本`DelMatically.py`。
```
#!/usr/bin/python
import os
import re

print 'start python script! Delete AutoMatically Manage Signing'
filePath = "/Users/yons/Documents/work/bin/prj/Unity-iPhone.xcodeproj/project.pbxproj"
f = open(filePath, 'r+')
contents = f.read()
f.seek(0)
f.truncate()
pattern = re.compile(r'(TestTargetID = (\w*)) \/\* Unity-iPhone \*\/;')
f.write(pattern.sub(r'\1;\n\t\t\t\t\t};\n\t\t\t\t\t\2 = {\n\t\t\t\t\t\tProvisioningStyle = Manual;', contents))
f.close()
print 'end python script !'
```
然后用shell运行python，把`xcodeprojPath`和`Yours`换成你们自己对应的值。
```
python DelAutoMatically.py
xcodebuild -project xcodeprojPath -sdk iphoneos -scheme "Unity-iPhone" CONFIGURATION_BUILD_DIR='./' CODE_SIGN_IDENTITY="Yours" PROVISIONING_PROFILE="Yours"
```
**需要注意的是**`PROVISIONING_PROFILE`值应该是一串数字+字母，这个值可以用NodePad++打开对应的mobileprovision文件，其中有如下结构。其中string标签包裹的值即是`PROVISIONING_PROFILE`。

```
<key>UUID</key>
<string></string>
```
