---
layout: post
title:  "如何使用 LuaStudio 断点调试 slua-unreal ？"
date:   2018-10-30 10:03:04 +0800
categories: language
---
### 如何使用 LuaStudio 断点调试 slua-unreal ？

## 前言
Lua作为脚本语言，不仅解决了C/C++ 无法解决的开发效率难题，还降低了开发的成本和风险。Lua开发的好处不用多说。<br>
LuaStudio是大家很熟悉的编写Lua的IDE，非常易用，我们当然也首选它来开发lua脚本。<br>
同时我们选择了之前开发Unity项目就已经使用的Slua方案来作为我们的UE lua的底层。<br>
我们要解决的是如何使用LuaStudio断点调试slua-unreal。<br>

## 开发环境
我们使用的是UE 4.20、LuaStudio v9.82 版本。

## 调试原理
LuaStudio 使用 hook C API 的方式来断点。
Lua5.x debug 库，可以把所有的 C API 中的 debug 相关 api 都导出了，包含 hook

`gethook(optional thread)`<br>
`Returns the current hook settings of the thread, as three values − the current hook function, the current hook mask, and the current hook count.`<br>

记录一张断点位置表，设置行模式的 hook ，每次进入 hook 都检查是否是断点处，若是就停下来等待交互调试。
但是 slua-unreal 设置思路是把 slua 作为项目的 sdk 嵌入，以避免和项目中的其他 lua 方案冲突，所以 slua-unreal 使用带命名空间的 c++ 方案，无法导出 c API，这就导致了 LuaStudio 无法断点调试 slua-unreal 了。（当然我们也可以使用的log来调试）

## 解决办法
slua-unreal 提供了 LuaInterface 入口，我们只要在这里把需要的Lua API 导出就可以了。<br>
如下<br>
LuaInterface.h<br>
![LuaInterface.h](/images/hook1.png)<br>

LuaInterface.cpp<br>
![LuaInterface.cpp](/images/hook2.png)<br>

在 Lua.h 文件中可以找到所有需要导出的API（所有带 LUA_API 标记），在 LuaInterface 中按上图格式导出即可。<br>
大家如果编译不过，尝试把namespace（slua::）加上去。<br>

## 如何调试
启动UE，选择工程，<br>
打开 LuaStudio 选择，调试>>>>>附加到进程，附加到UE工程进程，<br>
运行场景，打断点，就可以了。<br>

附加进程成功<br>
![附加进程成功](/images/debug1.png)<br>

断点成功<br>
![断点成功](/images/debug2.png)<br>

变量监视，输入变量名查看<br>
![变量监视](/images/debug3.png)<br>


最后感谢 LuaStudio 作者轩辕阿建、slua-unreal 作者 Siney 在这个过程中给予的帮助。<br>

搭车推荐下我写的一个 Unity 大地图遮挡剔除方案插件<br>
[PotentiallyVisibleSetPlugin](https://github.com/nashnie/PotentiallyVisibleSetPlugin)<br>

For more details see <br>
[DebuggingLuaCode](http://lua-users.org/wiki/DebuggingLuaCode)<br>
[lua_debugging](https://www.tutorialspoint.com/lua/lua_debugging.htm)<br>
[sluaunreal](https://github.com/Tencent/sluaunreal)<br>
