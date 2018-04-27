---
title: Latex编译pdf后的字体嵌入问题
date: 2018-04-27 13:54:51
tags: Latex
categories: LuanQiBaZao
---


最近在提交论文的camera ready版本的时候需要在[IEEE PDF-eXpress]中检测。
结果检查结果提示`Font Arial, Arial, Italic is not embedded (18x on page 3)`问题。
就是一些字体没有embedded。

<!-- more -->

---

# 产生的原因
Latex源文件中有PDF格式的图片，
在这些PDF格式的图片中，没有嵌入字体（not embedded），
所以最终生成的PDF论文会提示字体没有嵌入。

- 有可能是用visio生成PDF文件时没有嵌入；
- 有可能是matlab生成的`.eps`格式的文件在Latex编译转化为PDF格式时，没有嵌入；

---

# 查看PDF格式的图片字体嵌入
可以打开PDF格式的文档，
`文件` ->`文档属性` -> `字体`，
其中就会显示该PDF文档中所用的所有字体了，
每一个字体后面，如果注明了`embedded(已嵌入)`或`embedded subset(已嵌入子集)`,
就说明该字体是嵌入的。

---

# 解决方案
## 如果我们用Latex编译论文时用了matlab生成的`.eps`的图
 由于eps文件是ascii文件，里边只是给出字体的名称。
 由于matlab图里面默认的是Helvetica字体，
 当我们用latex生成pdf时会发现Helvetica字体是没有嵌入的。
 
 1. 将eps文件用 写字板 打开
 2. 将字体设置部分：
 ```
    %% IncludeResource: font Helvetica
    /Helvetica /WindowsLatin1Encoding 120 FMSR
    第二行改为
    /ArialMT /WindowsLatin1Encoding 120 FMSR
 ```
 3. 保存，然后重新用 Latex 生成 pdf。 
 
## 如果是PDF图片本身没有嵌入字体
 可以用Acrobat修改为常用字体重新编译
 

[IEEE PDF-eXpress]: https://www.pdf-express.org/