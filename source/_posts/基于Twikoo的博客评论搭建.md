---
title: 基于Twikoo的博客评论搭建
date: 2024-03-03 15:06:11
tags: Blog
categories: 经验分享
cover:
index_img: ./基于Twikoo的博客评论搭建/twikoo.png
banner_img:
comment: 'twikoo'
---

# 写在前面

寒假期间，好友章隐基于[WordPress](https://cn.wordpress.org/)搭建了自己的博客[彰隐的个人博客](http://117.72.35.48/)，包含有评论系统，十分有趣，故决定给自己的站点也增加这一服务。

本站基于[Hexo](https://hexo.io/zh-cn/index.html)搭建，在GitHub Pages和Gitee Pages双端部署，使用[Fluid](https://fluid-dev.github.io/hexo-fluid-docs/)主题进行美化，功能丰富，内置了许多主题插件使用。

打开 node_modules/hexo-theme-fluid/_config.yml 并查找 comment 字段，可以看到

```yaml
# 评论插件
# Comment plugin
comments:
  enable: true
  # 指定的插件，需要同时设置对应插件的必要参数
  # Options: utterances | disqus | gitalk | valine | waline | changyan | livere | remark42 | twikoo | cusdis | giscus | discuss
  type: twikoo
```

将```enable```字段改为```true```并将```type```字段改为对应的评论服务即可

# 评论系统的挑选

## Valine

[Valine](https://valine.js.org/)诞生于2017年8月7日，是一款基于[LeanCloud](https://leancloud.cn/)的快速、简洁且高效的无后端评论系统。

其无后端的性质导致了评论难以管理，故放弃

## Waline

显然，[Waline](https://waline.js.org/)是对Valine的一个继承，它具有后端。

推荐的后端部署渠道有Vercel，CloudeBase等，在我的账号与设备上出现了不可预知的错误，另外vecel.app域名在国内无法正常访问，会出现Failed to fetch错误。

较为繁琐但免费高效的Netlify部署，以我无法注册账号宣告失败。

很可惜，Waline也无法使用。

## Twikoo

这便是本文的主角，[Twikoo | 一个简洁、安全、免费的静态网站评论系统](https://twikoo.js.org/)。

我选择了 [Hugging Face](https://www.hugging-face.org/) 部署，与Netlify类似，完全免费并且在国内拥有不错的访问速度，其数据库位于 [MongoDB](https://www.mongodb.com/zh-cn) ，可以使用 Google 账号快速登陆，不存在注册账号问题，并且同样免费。

获取环境ID后填入_config.yml中的

```yaml
twikoo:
  envID: Here!!!
```

# 效果展示

![comment](./基于Twikoo的博客评论搭建/comment.png)

本文下方的评论区同理，如果你使用的是移动设备，布局会略有不同
