---
title: 重新开始使用博客
date: 2022-12-10 14:00:00
categories:
	- 生活
tags:
	- 生活
---

# 重新开始使用博客

工作一年多，荒废学习许久，决定重新开始使用博客，愿能学有所得。



```bash
npm install --global hexo
6.3.0

hexo init blog

git clone https://github.com/next-theme/hexo-theme-next themes/next
8.14.0

npm --registry https://registry.npm.taobao.org install hexo-excerpt --save
减小 depth
excerpt:
  depth: 3
  excerpt_excludes: []
  more_excludes: []
  hideWholePostExcerpts: true
这个 depth 就是 md 文件中的代码块，也就是说当从文章（不包括标题）开始算起 10 个代码块后就会开始显示阅读全文按钮（在主页上）

图片
/images/
typora-root-url: ../
```



## 图片

/images/
typora-root-url: ../



网站缩略图

favicon.png 放入 source/images

修改 next 配置

favicon:

  small: /images/favicon.png

  medium: /images/favicon.png



## 显示更新时间

updated

如果文章头部 yaml 定义有 updated 字段、或 md 文件修改时间，前者优先级更高。

- 如果开启了 `post_meta.updated_at.another_day` ，当 date 和 updated 日期相同时，只会显示发布时间。
- https://www.voidking.com/dev-hexo-next-update-time/



## 布局

scheme: Gemini

