---
title: Hexo提交百度搜索引擎收录
date: 2023-03-14 15:01:30
tags: 
- Hexo
- 百度
categories: 经验分享
cover:
index_img: ./Hexo提交百度搜索引擎收录/hexo_baidu.jpg
banner_img:
---



# 百度资源平台添加网站

访问[百度搜索资源平台官网](https://ziyuan.baidu.com/), 注册或者登陆百度账号

依次选择**用户中心** **站点管理**, 添加你的网站

# 设置主动推送

## 安装插件[^1]

```bash
npm install hexo-baidu-url-submit --save
```

## 配置_config.yaml

```yaml
baidu_url_submit:
	count: 1 //每次提交多少个新链接
	host: //域名
	token: //在百度站长平台获取
	path: baidu_urls.txt
```

*其中token在 **站点管理** **资源提交** **普通收录** 中*

## 加入新的deployer

```yaml
deploy:
//这里是之前的配置
- type: baidu_url_submitter
```

## 最后部署

运行

```bash
hexo g -d
```

出现形如

```bash
{"remain":2951,"success":19}
```

表示成功

# 原理介绍

```bash
hexo g
```

产生一个文本文件, 包含新连接

```bash
hexo d
```

读取链接, 提交至百度引擎



[^1]:[huiwang/hexo-baidu-url-submit](https://github.com/huiwang/hexo-baidu-url-submit)
