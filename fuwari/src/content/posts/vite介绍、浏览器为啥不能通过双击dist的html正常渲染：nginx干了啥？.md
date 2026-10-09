---
title: vite介绍、浏览器为啥不能通过双击dist的html正常渲染：nginx干了啥？
published: 2026-10-09
description: ''
image: ''
tags: [前端]
category: '前端'
draft: false 
lang: ''
---
- [x] 是否借助ai


# Vite、构建、dist、Nginx 与浏览器运行原理笔记

## 1. Vite 是什么？

Vite 是前端开发阶段使用的工具，可以理解为：

> 帮助前端代码快速运行、转换、热更新、构建发布文件的工具。

主要作用：

1.  让现代前端代码可以运行
2.  修改代码后快速看到效果
3.  将项目构建成可部署文件

整体流程：

开发代码（Vue / React / TypeScript） → Vite → dist → Nginx → 浏览器

## 2. 为什么 Vite 启动快？

传统方式：

项目启动 → 扫描全部文件 → 转换全部代码 → 启动

项目大时会很慢。

Vite：

启动服务器 → 浏览器请求哪个文件 → 处理哪个文件

它不是渐进式披露，而是按需处理。

渐进式披露针对用户界面展示； Vite 的优化针对机器处理任务。

## 3. 什么是打包？

打包：

> 把方便开发的代码转换成适合浏览器运行的代码。

例如：

开发：

src/ - main.ts - user.ts - login.ts

构建后：

dist/ - index.html - assets/app.js - assets/style.css

## 4. 为什么需要转换代码？

浏览器主要认识：

-   HTML
-   CSS
-   JavaScript

但是开发中经常使用：

-   TypeScript
-   Vue
-   React JSX

例如：

TypeScript：

const age:number = 20;

转换：

const age = 20;

因为浏览器不认识 TypeScript 类型。

## 5. 打包会做什么？

### 合并文件

多个 CSS、JS 文件合并，减少浏览器请求数量。

### 删除无用代码

没有被使用的代码会被删除。

### 压缩代码

减少文件大小，提高加载速度。

### 兼容转换

把新的 JavaScript 语法转换成浏览器支持的形式。

## 6. 什么是请求数？

浏览器加载网页时，需要请求：

-   HTML
-   CSS
-   JS
-   图片

每请求一个资源，就是一次请求。

打包可以减少请求数量。

## 7. CDN 拉取请求是什么？

CDN 是内容分发网络。

作用：

把资源放到距离用户更近的服务器。

例如：

```{=html}
<script src="https://cdn.xxx.com/anime.js">
```

表示浏览器会额外向 CDN 请求 anime.js。

## 8. crossorigin 和 CORS

跨域：

一个网站请求另一个网站资源。

例如：

a.com 请求 b.com。

CORS：

服务器告诉浏览器：

哪些网站允许访问自己的资源。

crossorigin：

告诉浏览器按照 CORS 规则处理这个资源。

## 9. dist 为什么双击打不开？

双击：

file:///xxx/dist/index.html

这是本地文件模式。

现代前端通常依赖 HTTP 环境。

例如：

```{=html}
<script src="/assets/index.js">
```

这里的 / 表示网站根目录。

file:// 没有网站根目录，所以可能找不到资源。

## 10. 为什么 Nginx 可以运行 dist？

Nginx 是 HTTP 文件服务器。

流程：

浏览器 → HTTP 请求 → Nginx → 返回 dist 中的文件 → 浏览器执行 JS

Nginx 不执行 JavaScript。

它只负责：

-   找文件
-   返回文件

浏览器负责：

-   执行 JS
-   渲染页面

## 11. 开发和生产区别

开发：

Vue源码 → Vite开发服务器 → 浏览器

生产：

Vue源码 → vite build → dist → Nginx → 浏览器

Vite 只负责生成 dist，线上运行不需要 Vite。

## 12. 和 Java 类比

Java：

.java → Maven编译 → jar → JVM运行

前端：

.vue/.ts → Vite build → dist → 浏览器运行

最终理解：

Vite 是构建工具，不是运行环境。

dist 是构建产物。

Nginx 提供浏览器需要的 HTTP 环境。

浏览器最终执行 JavaScript 并渲染页面。