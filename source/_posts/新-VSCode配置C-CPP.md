---
title: '[新]VSCode配置C/CPP'
date: 2024-09-19 15:47:24
tags:
- C++
categories:
- 经验分享
cover:
index_img: ./新-VSCode配置C-CPP/cpp.png
banner_img:
---

# 前言

近日在阅读他人题解时，发现了未曾见过的语法

```cpp
auto a = vector(n, INT_MIN);
```

即初始化一个长度为n的 `vector` 并命名为 `a` ，初始值为 `INT_MIN`，这比先前的

```cpp
vector<int> a(n);
for(auto &i: a) i = INT_MIN;
```

方便太多了！

然而，我在自己的编辑器里这么写，编译器会报错，查看发现我所使用的一直是c++11的标准，为了支持新标准，我们需要新的编译器，之前的文章 [VS Code中C++环境配置 - Where Is Mountain](https://www.harkerhand.online/VS-Code中CPP环境配置/) 所使用的gcc版本是8.1.0，所以写此文，更新一下配置相关。

# 安装 mingw-w64

这是官网的下载页 [Downloads - MinGW-w64](https://www.mingw-w64.org/downloads/)，然后进入 MinGW-W64-builds 的 GitHub Release 页面，按照你的需要选择对应的版本下载，下面给出各个选项的含义。

- i686、x86_64 也就是俗称的 32/64 位系统，现在基本都是 x86_64 
- 线程模型：win32没有C++ 11多线程特性；posix支持C ++ 11多线程特性
- 异常处理模型：32位系统推荐dwarf，64位系统推荐seh
- msvcrt 是老古董了，没有继承远古项目的需求用 ucrt 就行

按照上面的解释，我选择了 [x86_64-14.2.0-release-win32-seh-ucrt-rt_v12-rev0.7z](https://github.com/niXman/mingw-builds-binaries/releases/download/14.2.0-rt_v12-rev0/x86_64-14.2.0-release-win32-seh-ucrt-rt_v12-rev0.7z) 。

解压缩到你喜欢的位置，就算是安装完成一半了。

# 配置环境变量

就算不进行这一步骤，在部分场景下也是可以正常编译的，但你需要手动指定编译器的路径，比较麻烦。

为了适配一些自动化脚本，比如很常用的 VS Code 插件 Code Runner，你需要帮助它们找到你的编译器。

- win + S 搜索**编辑系统环境变量**，进入后点击**环境变量**，在系统变量的**Path**值中添加你的安装路径下的bin文件夹路径，比如 D:\mingw64\bin 。完成后保存退出。
- win + R 输入cmd进入命令行，输入 `gcc -v` 回车，返回 `gcc version 14.2.0 (x86_64-win32-seh-rev0, Built by MinGW-Builds project)` 即配置完成。

关于vscode的设置可以参考前面提到的 [VS Code中C++环境配置 - Where Is Mountain](https://www.harkerhand.online/VS-Code中CPP环境配置/) 这篇文章。

