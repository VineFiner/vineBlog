---
title: "安装 Hugo"
date: 2023-04-16T18:33:53+08:00
description: "Quasimodo"
featured_image: ""
tags: []
draft: false
---

## 生成站点

```
hugo new site blog
```

## 创建文章

- 进入博客目录

```
cd blog
```
- 生成关于页面

```
hugo new about.md
```

- 生成第一篇文章

```
hugo new post/install.md
```

## 预览

```
hugo server
```

## 生成静态文件

```
hugo server -D
```