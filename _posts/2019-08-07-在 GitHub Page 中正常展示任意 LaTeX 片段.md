---
layout: post
title: "在 GitHub Page 中正常展示任意 LaTeX 片段"
description: 在 GitHub Page 中正常展示任意 LaTeX 片段
modified: 2019-08-07
category: Jekyll
tags: [GitPage, Jekyll]
imagefeature:
mathjax: false
chart:
comments: false
featured: false
---

> 参考文件：
> [https://zhuanlan.zhihu.com/p/50361221](https://zhuanlan.zhihu.com/p/50361221)  
> [https://jekyllrb.com/docs/liquid/tags/#code-snippet-highlighting](https://jekyllrb.com/docs/liquid/tags/#code-snippet-highlighting)  

- GitHub Page 的（主要）输入是 markdown 文件，输出是 HTML/CSS/JS 文件。
- 如果 markdown 文件包含代码块，且代码块中包含花括号 { 或 }，尤其是包含 {% raw %}{% 或 {{{% endraw %} 符号组合时，GitHub Page 会报错。
- 分别在代码块前后添加 {% raw %}{%{% endraw %} raw {% raw %}%}{% endraw %} 和 {% raw %}{%{% endraw %} endraw {% raw %}%}{% endraw %} 即可解决该问题。
