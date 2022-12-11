---
title: Hexo 使用记录
date: 2022-12-10 17:03:18
categories:
	- others
tags:
	- hexo
typora-root-url: ../
---

# 简介
工作一年多，很久没有系统学习了，重新收拾一下博客。
重新使用 Hexo 和 NexT 主题，发现很多功能都忘记了，所以再次记录一下。

# 运行
```bash
# 在更改 hexo 的配置、CSS、JS 文件后，需要清理缓存，重新运行才能生效
hexo clean

# 在本地启动临时服务
hexo server

hexo generate
hexo deploy
```

# 语法技巧
## 公式
> 见[官方文档](https://theme-next.js.org/docs/third-party-services/math-equations.html)

块公式需要以`$$\begin{equation} \end{equation}$$` 包裹。
行内公式以 `$$` 包裹。

可以使用 Latex 的所有公式语法，包括公式编号 `\label{eq1}` 和对其的引用 `\eqref{eq1}`。举个例子

```latex
$$\begin{equation} \label{eq1}
e=mc^2
\end{equation}$$
```
上述代码会生成如下的公式，并且可以使用 `$\eqref{eq1}$` 对其进行引用 $\eqref{eq1}$。
$$\begin{equation} \label{eq1}
e=mc^2
\end{equation}$$

## 链接站内文章
> 参考 [[GitHub] next 或者 hexo 链接站内文章的方法](https://github.com/iissnan/hexo-theme-next/issues/978)

对于链接站内文章有两种方式：

- 不链接页内锚点    
  Hexo 语法支持 `{% post_link &lt;file-name&gt; &lt;text&gt; %}`。例如 `{% post_link hexo-use Hexo 使用记录 %}` ⇨ {% post_link hexo-use Hexo 使用记录 %}。
- 链接页内锚点    
  Hexo 语法不支持这种方式，只能自己手写链接了。例如 `[2022/12/hexo-use/#运行](/2022/12/hexo-use/#运行)` ⇨ [2022/12/hexo-use/#运行](/2022/12/hexo-use/#运行)。

## 大于号和小于号

Hexo 渲染大于号和小于号时，如果二者成对出现，会被识别为一个 HTML 标签，所以需要使用转义字符：

- `&lt;` ⇨ &lt; 
- `&gt;` ⇨ &gt;
