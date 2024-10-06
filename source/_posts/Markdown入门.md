---
title: Markdown入门
date: 2023-03-10 19:10:48
tags: Markdown
categories: 经验分享
cover: /img/md.jpg
index_img: /img/md.jpg
banner_img: 
---

# 前言

## 什么是Markdown?

> Markdown 是一种轻量级标记语言，它允许人们使用易读易写的纯文本格式编写文档，Markdown文件的后缀名便是“.md”。

## 工欲善其事，必先利其器！
作者使用Visual Studio Code进行编辑
安装插件[Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one)和[Markdown Preview Enhanced](https://marketplace.visualstudio.com/items?itemName=shd101wyy.markdown-preview-enhanced)

## 注意

本文并非[Markdown官方教程](https://markdown.com.cn/basic-syntax/)，仅介绍个人常用语法


---

# 基本语法

## 标题

|  Markdown语法  |  预览效果  |
|---------------|-----------|
|\# Heading level 1	|<h1>Heading level 1</h1>|
|\## Heading level 2|<h2>Heading level 2</h2>|
|**···**|**···**|

_Tips:总共有六级，在\#后加上空格保证兼容性_

## 强调

#### 粗体

|  Markdown语法  |  预览效果  |
|----------------|-----------|
|Thin \*\*Bold**|Thin <strong>Bold</strong>|
|Thin \_\_Bold__|Thin <strong>Bold</strong>|

#### 斜体

|  Markdown语法  |  预览效果  |
|----------------|-----------|
|Straight \*Italic*|Straight <em>Italic</em>|
|Straight \_Italic_|Straight <em>Italic</em>|

_Tips:表示又粗又斜时连续使用三个\*\*\*_

## 引用

<strong>

\> 引用1
\>\> 引用2

</strong>

> 引用1
>> 引用2

_Tips:空格保证兼容性，多级之间需要分隔_

## 代码块

<strong>

\```cpp
\#include<iostream>
int main()
{
    cout << "Hello World!";
    return 0;
}
\```

</strong>

```cpp
#include<iostream>
using namespace std;
int main()
{
    cout << "Hello World!";
    return 0;
}
```

## 链接

<strong>

\[Where is Mountain](`https://harker299.gitee.io`)

</strong>

[Where is Mountain](`https://harker299.gitee.io`)


## 图片

<strong>

!\[这是图片](路径或url "图片描述")

</strong>



---