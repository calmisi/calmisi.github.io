---
title: scala函数相关知识点
categories: Spark
date: 2016-11-09 11:13:31
tags:
- scala
---

## 1.scala函数可以赋值给变量

{% asset_img 1.png %}

这里我们定义了一个函数fun1,然后将他赋值给了一个变量fun_v。
格式为：
`{% raw %} val 变量名 = 函数名 + 空格 _{% endraw %}`
这里函数名后面必须要有空格，表明是函数的原型。
`{% raw %}_{% endraw %}`代表函数的参数。