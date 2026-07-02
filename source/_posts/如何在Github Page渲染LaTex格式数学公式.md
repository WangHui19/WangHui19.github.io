---
title: 如何在Github Page渲染LaTex格式数学公式
date: 2022-11-04 20:27:16
tags: 
    - blog
    - Latex
    - 数学公式
    - MathJax
categories:
    - blog
---
今天写博客的时候遇到了显示数学公式的问题，不想用图片显示，想到用Latex语法写。

在VSCode中写Markdown的时候，预览公式显示很正常，但是上传到github page不能显示。让人非常难受，于是上网搜索解决方法。

## 解决方案：
<!--more-->
### 第一种
在搜索结果中，很多的CSDN上，都说是chrome浏览器中没有对应插件导致。既然这样，那就下插件！！！
1. MathJax Plugin for Github
2. Math Anywhere

两个选一个就行，我都下载了，并且启用了。
不能说有用，只能说一点用也没有。

### 第二种
采用外挂Javascript的方案支持跨浏览器的内容渲染，可以用MathJax渲染。方法如下：
1. 设置markdown引擎为kramdown，方法为在_config.yml里添加：(中间有空格)

   markdown: kramdown
2. 在md文件开始输入以下代码：
~~~
<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>
~~~
然后正文就可以写公式：`$e = m c^2$` 这样就能正确显示了。

这次解决了我的问题。公式能够成功显示了。<br>

参考链接：
* https://zhuanlan.zhihu.com/p/36302775
* https://stackoverflow.com/questions/26275645/how-to-support-latex-in-github-pages
* https://zybuluo.com/anboqing/note/53250
  

但后面仍然需要注意的地方是：

`${a}_{1}$` 这种仍然没办法显示，因为只要 `_` 前面有 `{}` ，那就必须使用 `\` 进行转义，也就是变成 `${a}\_{1}$` 才能正常显示。

但出现了新问题，对于`${a}_{1}$`，没有办法将公式的Latex写法显示在markdown上，会直接被转换为公式形式展示。