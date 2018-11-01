---
layout: post
title:  "如何使用 VisualStudio2017 调试 Unreal4 Android APK？"
date:   2018-11-1 13:59:00 +0800
categories: language
---
如何使用 VisualStudio2017 调试 Unreal4 Android APK？
### VisualStudio 环境设置
1. 检查组件是否安装完全；<br>
开始菜单->所有程序->Vistual Studio Installer 打开组件配置界面，点击 更多>修改<br>
**组件**<br>
![](/images/vsDebugUEAndroid1.png)

2. 设置 Android SDK；<br>
**Android SDK 路径** 我是使用的默认路径，大家可以选择自己 AndroidSDK 的路径<br>
![](/images/vsDebugUEAndroid2.png)

### Unreal 符号表文件以及APK准备
新建一个文件夹，放入需要调试的 Unreal4 Android APK 以及 版本同时生产的 Debug符号表文件，请注意符号表名字一定要是 **libUE4.so**
### VisualStudio 新建调试工程
1. VisualStudio 点击 File>>>Add>>>Existing Project... 添加准备的 Unreal4 Android APK
2. 在新建的工程上点击 Debug>>>Start new instance

![](/images/vsDebugUEAndroid3.png)
### Android ADB 连接检查
1. 手机 开启开发者模式
2. cmd > adb devices，查看设备列表
3. cmd > adb version，查看ADB 版本（ADB 1.0.32 (or later)）

**CMD**<br>
![](/images/vsDebugUEAndroid4.png)

如果以上还不行，重启电脑试试。<br>

