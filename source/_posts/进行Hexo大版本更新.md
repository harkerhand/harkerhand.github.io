---
title: 进行Hexo大版本更新
date: 2024-03-04 01:15:23
tags: Blog
categories: 经验分享
cover:
index_img: ./进行Hexo大版本更新/hexo.png
banner_img:
---

# 前言

昨晚偶然打开了 package.json，发现 hexo 版本为 6.3.0，遂决定升至最新 v7.1.1。

# 遭遇错误

执行报错

```shell
npm update hexo-cli -g
```



> ---
>
> 重启解决 99% 的问题，剩下 1% 用重装
>
> ---

秉持这个思想，我选择重装

# 操作步骤

## 备份

备份旧版本的文章，格式，Hexo及Fluid的配置文件

## 卸载

执行

```shell
npm uninstall hexo-cli -g
```

并进入 C:\Users\UserName\AppData\Roaming\npm 路径检查卸载是否彻底

## 安装本体

执行

```shell
npm install hexo -g
```

并进入 C:\Users\HarkerHand\AppData\Roaming\npm\node_modules\hexo\package.json 查找 ```version``` 字段，版本为 v7.1.1 即为安装成功

Git Bash cd 进入博客文件夹，需要清空

执行

```shell
hexo init
```

## 模块及主题恢复

*以下在 init 的文件夹中操作*

- [hexo-asset-image](https://github.com/xcodebuild/hexo-asset-image) 

  执行

  ```shell
  npm install hexo-asset-image --save
  ```

  在 ```_config.yml``` 中设置 ```post_asset_folder: true```

- [hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)

  执行
  
  ```shell
  npm install hexo-deployer-git --save
  ```
  
- [hexo-server](https://github.com/hexojs/hexo-server)

  执行

  ```shell
  npm install hexo-server --save
  ```

- [hexo-theme-fluid](https://github.com/fluid-dev/hexo-theme-fluid)

  执行

  ```shell
  npm install hexo-theme-fluid --save
  ```

  

# 补充

在更新主题时原配置被完全覆盖，提醒大家做好备份

**数据无价！！！**
