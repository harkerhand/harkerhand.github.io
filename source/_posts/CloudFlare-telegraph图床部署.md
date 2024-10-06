---
title: CloudFlare+telegraph图床部署
date: 2024-03-03 18:59:43
tags:
- Blog
- 图床
categories: 经验分享
cover:
index_img: https://image.harkerhand.online/file/5d0e3cb00e7e03672e347.png
banner_img:
---



# 提醒

本文封面及所涉及的所有图片均来自云端图床，故随时存在无法访问的风险

# 部署

## Fork 项目

将此项目Fork进自己的仓库 [Telegraph-Image](https://github.com/cf-pages/Telegraph-Image)

## 部署CloudFlare

使用 Pages 服务部署该仓库

创建KV : img_url

命名空间 img_url = img_url

环境变量 BASIC_USER BASIC_PASS 后台管理用户名，密码

部署后得到前端网址

后缀 /admin 进入后台管理

# 效果

公开使用：

[Pic-Url](https://telegraph-image-bhq.pages.dev) *源链*

[Image](https://image.harkerhand.online) *转发链*

| <img src="https://image.harkerhand.online/file/d75a74b5acb3470accf7c.jpg"  />**Knife is short, I use python** | <img src="https://image.harkerhand.online/file/1e6d8b6f0f776fb6d77c2.jpg" style="zoom:90%;" /> |
| :----------------------------------------------------------: | :----------------------------------------------------------: |







