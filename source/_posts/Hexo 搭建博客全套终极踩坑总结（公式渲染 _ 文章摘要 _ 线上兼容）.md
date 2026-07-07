---
title: Hexo 搭建博客全套终极踩坑总结（公式渲染 / 文章摘要 / 线上兼容）
date: 2026-07-07 14:00:00
tags: Hexo 博客搭建 技术踩坑
categories:
  - 技术笔记
mathjax: true
---

本文整理了 Hexo 建站高频踩坑解决方案，针对 LaTeX 公式线上失效、CDN 报错、下划线转义冲突、首页无摘要、线上线下渲染不一致等常见问题，汇总出一套可长期复用的稳定配置方案。
<!--more-->

## 一、前言

Hexo 多数异常并非代码错误，而是由缓存机制、线上线下环境差异、CDN 兼容问题导致。本文统一标准化配置，保证本地与线上渲染效果完全一致。

## 二、LaTeX 数学公式渲染终极方案（彻底解决 404 / 失效 / 乱码）

### 2\.1 传统 MathJax CDN 方案（彻底废弃）

网络上主流的 MathJax 前端 CDN 方案存在多处致命缺陷，完全不推荐使用：

- 境外 CDN 国内极易出现 404、超时、拦截问题

- Markdown 下划线语法与公式下标冲突，公式会被误解析为斜体

- 本地预览正常，GitHub Pages 线上公式直接失效

- 依赖前端 JS 异步加载，存在公式延迟、闪屏、卡顿问题

**最终结论：****彻底放弃**** MathJax 前端 CDN 方案。**

### 2\.2 最终稳定方案：hexo\-math 服务端预渲染 KaTeX

采用 hexo\-math \+ KaTeX 服务端预渲染，核心优势：编译阶段直接将公式渲染为静态 HTML，无需前端 JS 和外部 CDN，彻底消除线上线下渲染差异。

#### 1\. 安装插件

```Plain Text
npm install hexo-math --save
```

#### 2\. 清理冲突旧插件（关键）

```Plain Text
npm uninstall hexo-renderer-kramed markdown-it-katex hexo-renderer-markdown-it --save
npm install hexo-renderer-marked --save
```

#### 3\. 全局最终配置（线上可用版本）

`_config.yml`

```Plain Text
math:
  katex:
    css: true
    options:
      throwOnError: false
      strict: false
      trust: true
```

**关键配置：css: true 必须开启，否则线上环境缺失样式，公式直接失效。**

#### 4\. 全网统一公式书写规范（根治转义问题）

废弃传统 `$$`、`$ $` 公式写法，统一使用标签式渲染，根治语法转义问题：

行内公式：

```Plain Text
{% katex %}x^2+y^2=1{% endkatex %}
```

块级公式：

```Plain Text
{% katex %}
a = \frac{b}{c}
{% endkatex %}
```

#### 5\. 生效命令

```Plain Text
hexo clean && hexo g && hexo s
```

## 三、博客首页文章显示全文、没有摘要问题解决

### 3\.1 问题原因

Hexo 默认规则：文章无截断标记，首页将完整展示全文内容。

### 3\.2 标准正确写法

文章简介内容结束后，单独另起一行添加截断标签：

```Plain Text
<!--more-->
```

注意：带空格的 `<!-- more -->` 为无效写法

### 3\.3 兜底方案

可在文章头部配置项中直接自定义固定摘要，无需截断标签：

```Plain Text
---
title: xxx
excerpt: 这里写文章摘要
---
```

## 四、线上线下渲染不一致核心原因

- 本地预览不启用缓存，线上环境强缓存，修改配置后必须执行 `hexo clean`

- 本地路径容错性高，GitHub Pages 路径严格区分大小写，极易裂图失效

- MathJax 依赖前端动态渲染，线上易加载失败；KaTeX 服务端渲染无外部依赖，稳定性拉满

- `css: false` 仅本地调试可用，部署线上会直接导致公式样式丢失

## 五、日常固定使用命令（一套用到毕业）

日常开发仅需两套固定命令，适配预览与部署：

```Plain Text
hexo clean && hexo g && hexo s
```

线上部署：

```Plain Text
hexo clean && hexo g && hexo d
```

## 六、最终稳定环境总结

- 公式渲染：优先使用 hexo\-math \+ KaTeX 服务端渲染，彻底舍弃 MathJax

- 缓存机制：所有配置、文章修改后，必须 clean 重建，规避缓存异常

- 资源路径：图片统一使用绝对路径 `/img/xxx.png`，避免裂图问题

> （注：部分内容可能由 AI 生成）
