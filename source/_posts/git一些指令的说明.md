---
title: git一些指令的说明
date: 2016-10-28 17:40:02
categories: Git
tags: Git
---

本文记录本人在使用git的过程中，学习到的一些指令的说明。  
**本文不定期更新...**
<!-- more -->

## 1.本地库关联远程库，在本地仓库目录运行命令：

{% codeblock lang:git %}
$ git remote add origin git@github.com:username/repository.git
{% endcodeblock%}
origin: remote name

---
## 2.git fetch和git pull的区别  
Git从远程的分支获取最新的版本到本地可以用fetch和pull  
* `git fetch`  
相当于是从远程获取最新版本到本地，不会自动`merge`
{% codeblock lang:git %}
git fetch origin master
git diff HEAD FETCH_HEAD 
git merge origin/master
{% endcodeblock %}
以上命令的含义：  
首先从远程origin的master分支上下载最新的版本到本地的origin/master版本记录中  
然后比较本地的master和刚刚fetch的origin/master的区别  
最后进行合并  
`git diff HEAD FETCH_HEAD `  
 HEAD是本地仓库，不是本地工作区。查看`git fetch`后本地仓库和远程仓库的内容有哪些区别。  
* `git pull`
{% codeblock lang:git %}
git pull origin master
{% endcodeblock %}  
上述命令其实相当于`git fetch`和`git merge`  
在实际使用中，`git fetch`更安全一些，以为在`merge`之前，我们可以查看更新的情况，然后再决定是否合并  

* 关于`git fetch`和`git pull`有更详细地原理分析，请见  
http://blog.csdn.net/a19881029/article/details/42245955
---

## 3. git clone  
`git clone`自动创建了一个名为`origin`的远程连接，指向原有仓库。

---
## 4. ~字符  
~字符用户表示提交的父节点的相对引用。  
比如：3157e~1指向3157e前一个提交，HEAD~3是当前提交的回溯3个节点的提交。 

---
## 5. git reset 
* `git reset <file>` 
从缓存区移除特定文件，但不改变工作目录。它会取消这个文件的缓存，而不覆盖任何更改。  
`git reset`  
重设当前的整个缓冲区，匹配最近的一次提交，但工作目录不变。它会取消**所有**文件的缓存，而不会覆盖任何本地目录的修改，给你了一个重设缓存快照的机会。  
`git reset --hard`
重设缓冲区和工作目录，匹配最近一次的提交。除了取消缓存之外，`--hard`标记告诉Git还要重写当前工作目录中的更改。   

**注意：**  
上面所有的调用都是用来移除仓库中的修改。没有`--hard`标记时`git reset`通过取消缓存或取消一系列的提交，然后重新构建提交来清理仓库。而加上`--hard`标记对于作了大死之后想要重头再来尤其方便。

---
## 6. git rebase 
```git rebase <base>```  
将当前分支rebase到<base>，这里可以是任何类型的提交引用（ID、分支名、标签，或者是`HEAD`的相对引用）。 

rebase的主要目的是：**保持一个线性的项目历史**。比如说，当你在feature分支工作时master分支取得了一些进展：
{%asset_img 1.svg %}
要将你的feature分支整合进`master`分支，你有两个选择：直接merge，或者先rebase后merge。  
前者会产生一个三路合并(3-way merge)和一个合并提交，而后者产生的是一个快速向前的合并以及完美的线性历史。下图展示了为什么rebase到master分支会促成一个快速向前的合并。  
{%asset_img 2.svg %}
rebase是将上游更改合并进本地仓库的通常方法。你每次想查看上游进展时，用git merge拉取上游更新会导致一个多余的合并提交。在另一方面，rebase就好像是说：“我想将我的更改建立在其他人的进展之上。”  
**注意:**  
和`git commit --amend`,`git reset`一样，永远不应该rebase那些已经推送到公共仓库的提交。rebase会用新的提交替换旧的提交，你的项目历史会像突然消失了一样。	

---
## 7.git pull
{% codeblock %}
git pull <remote>
{% endcodeblock %}
和下面的代码起着相同的效果
{% codeblock %}
git fetch origin
git log --oneline master..origin/master //查看本地分支和远程分支的区别
git checkout master
git merge origin/master
{% endcodeblock %}
拉取当前分支对应的远程副本中的更改，并立即并入本地副本。效果和`git fetch`后接`git merge origin/.`一致。  
{% codeblock %}
git pull --rebase <remote>
{% endcodeblock %}和上一个命令相同，但是使用`git rebase`合并远程分支和本地分支，而不是`git merge`.

**基于rebase的Pull**  
`--rebase`标记可以用来保证线性的项目历史，防止合并提交（merge commits）的产生。很多开发者倾向于使用rebase而不是merge，因为“我想要把我的更改放在其他人完成的工作之后”。这种情况下，使用带有--rebase标记的git pull甚至更像svn update，与普通的git pull相比而言。

事实上，使用--rebase的pull的工作流是如此普遍，以致于你可以直接在配置项中设置它：

{% codeblock%}
git config --global branch.autosetuprebase always
{% endcodeblock %}
在运行这个命令之后，所有的git pull命令将使用git rebase,而不是git merge。
