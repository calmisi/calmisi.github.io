---
title: 使用Hexo和Next的一些技巧
tags: Hexo
categories: Hexo&NEXT
date: 2016-10-29 15:19:36
---



本文描述一些关于Hexo和NEXT的技巧，大多数为自己在网上搜索所得，整理于此，方便自己查看。
<!-- more -->
## 1. Hexo设置阅读全文  
Hexo的首页中，文章是会显示全文的，这样看起来太冗长了。首页应该显示文章的部分截取，然后提供“阅读全文”功能，点击跳转到具体的文章显示页。我在网上搜索到了3种方法。

1. 在文章中使用`<!-- more -->`手动进行截取    
这种方法可以根据文章的内容，自己在合适的位置添加`<!-- more -->`标签，则首页显示的文章截取到该标签之前。  
**注意：**Next主题中，需要将主题配置文件_config.yml中的 `scorll_to_more`设置为`true`;

2. 在文章的`front-matter`中添加description，并提供具体的description描述。  
这种方法只会在首页列表中显示文章的摘要内容，进入文章详情后不会再显示。  
{% codeblock lang:md %}
---
titile: firstArticle
date: 
tag:
categories:
description: it is the desciption of this article, and it will display at homepage. When you clcik 'read more', it won't showed at the whole article.
---
{% endcodeblock %}  

3. 自动形成摘要，在主题配置文件中设置  
默认截取的长度为150字符，可以根据需要自行设置  
{% codeblock lang:yml %}
auto_excerpt:
enable: true
length: 150
{% endcodeblock %}

**建议使用第1种方法，这样不仅可以精确的控制不同的文章需要显示的摘要的内容，还可以让Hexo中的插件更好地识别。并且第一种方法的优先级比第三种方法的高。**

---
## 2.Next主题解决首页菜单(导航栏)中`tags`和`categories`需要手动创建页面  
在NEXT主题中，可以设置菜单。需要编辑主题配置文件_config.yml，对应的字段为`menu`。  
{% codeblock lang:yml %}
menu:
  home: /
  archives: /archives
  #about: /about
  #categories: /categories
  tags: /tags
  #commonweal: /404.html
{% endcodeblock %}  
其中:前面代表menu中选项的字段，后面表示该字段对应的链接为网站的那个目录。  
其中`tags`和`categories`取消注释后，Hexo generate后会提示没有页面的。这是因为我们只是创建了这个链接，而没有创建页面，默认情况下`home`和`archieves`页面是不用自己创建的，所以`tags`和`categories`需要我们自己创建。  
* ### 创建分类  
在Hexo站点的根目录下执行
{% codeblock %}
hexo new page "tags"
{% endcodeblock %}
* ### 修改type字段  
执行完，在Hexo根目录下的source文件夹中会多出一个tags文件夹，里面默认有一个`index.md`的文件，这就是我们创建的导航栏的tags页面  
默认这个页面是空的，我们需要修改其`front-matter`以让Hexo识别。  
编辑`index.md`添加`type`和`comments`字段  
{% codeblock %}
type: tags
comments: false
{% endcodeblock %}  
Hexo会根据type字段来生成对应的页面的。
**同理，`categoris`页面需要设置对应的`type`为categories**





