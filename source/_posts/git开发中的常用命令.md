---
title: git开发中的常用命令
categories: Git
date: 2016-11-03 09:29:52
tags: Git
---

# 分支  
## git checkout
`git checek`命令允许你切换用`git branch`创建的分支。查看一个分支会更新工作目录中的文件，以符合分支中的版本，它还会告诉Git记录那个分支上的新提交。

---
## git merge
{% codeblock %}
git merge --no-ff <branch>
{% endcodeblock %}
<!-- more -->
将指定分支并入当前分支，但总是生成一个合并提交（即使是快速向前合并）。**这可以用来记录仓库中发生的所有合并**

# 撤销，查看历史等。
## git revert
{% codeblock %}
git revert <commit>
{% endcodeblock %}
**生成**一个撤销了指定的`<commit>`引入的修改的**新提交**，然后应用到当前分支。

撤销(revert)应该用在你想要项目历史中移除一整个提交的时候。比如，你在追踪一个bug,然后你发现它是由一个提交造成的，这时候撤销就很有用。与其说自己去修复它，然后提交一个新的快照，不如用`git revert`，它帮你做了所有的事情。

* ### **撤销(revert)和重设(reset)对比**  
理解这一点很重要--`git revert`回滚了『单独一个提交』——它没有移除后面的提交，然后回到项目之前的状态。
{% asset_img the_comparison_between_revert_and_reset.svg %}  
撤销和重设相比有两个重要的优点。首先，它不会改变项目历史，对那些已经发布到共享仓库的提交来说这是一个安全的操作。
其次，`git revert`可以针对历史中任何一个提交，而`git reset`只能从当前提交向前回溯。比如，你想用`git reset`重设一个旧的提交，你不得不移除那个提交后的所有提交，再移除那个提交，然后重新提交后面的所有提交。不用说，这并不是一个优雅的回滚方案。

* ### 例子
下面这个例子是`git revert`一个简单的演示。它提交了一个快照，然后立即撤销这个操作。
{% codeblock %}
# 编辑一些跟踪的文件

# 提交一份快照
git commit -m "Make some changes that will be undone"

# 撤销刚刚的提交
git revert HEAD
{% endcodeblock %}  
这个操作可以用下图可视化：  
{% asset_img the_example_of_revert.svg %}  

---
## git reset  
和`git checkout`一样，`git reset`有很多种用法。它可以被用来移除提交快照，尽管它通常被用来撤销缓存区和工作目录的修改。不管是哪种情况，它应该只被用于 本地 修改——你永远不应该重设和其他开发者共享的快照。

* ### 用法  
{% codeblock %}git reset <file>{% endcodeblock %} 
从缓存区移除特定文件，但不改变工作目录。它会取消这个文件的缓存，而不覆盖任何更改。
{% codeblock %}git reset{% endcodeblock %} 
重设缓冲区，匹配最近的一次提交，但工作目录不变。它会取消 所有 文件的缓存，而不会覆盖任何修改，给你了一个重设缓存快照的机会。
{% codeblock %}git reset --hard{% endcodeblock %} 
重设缓冲区和工作目录，匹配最近的一次提交。除了取消缓存之外，--hard 标记告诉Git还要重写所有工作目录中的更改。换句话说：它清除了所有未提交的更改，所以在使用前确定你想扔掉你所有本地的开发。
{% codeblock %}git reset <commit>{% endcodeblock %} 
将当前分支的末端移到<commit>，将缓存区重设到这个提交，但不改变工作目录。所有<commit>之后的更改会保留在工作目录中，这允许你用更干净、原子性的快照重新提交项目历史。
{% codeblock %}git reset --hard <commit>{% endcodeblock %} 
将当前分支的末端移到<commit>，将缓存区和工作目录都重设到这个提交。它不仅清除了未提交的更改，同时还清除了<commit>之后的所有提交。

---
## git clean
git clean命令将未跟踪的文件从你的工作目录中移除。它只是提供了一条捷径，因为用git status查看哪些文件还未跟踪然后手动移除它们也很方便。和一般的rm命令一样，git clean是无法撤消的，所以在删除未跟踪的文件之前想清楚，你是否真的要这么做。

git clean命令经常和git reset --hard一起使用。记住，reset只影响被跟踪的文件，所以还需要一个单独的命令来清理未被跟踪的文件。这个两个命令相结合，你就可以将工作目录回到之前特定提交时的状态。

* ### 用法
{% codeblock %}git clean -n{% endcodeblock %} 
执行一次git clean的『演习』。它会告诉你那些文件在命令执行后会被移除，而不是真的删除它。
{% codeblock %}git clean -f{% endcodeblock %} 
移除当前目录下未被跟踪的文件。-f(强制)标记是必需的，除非clean.requireForce配置项被设为了false (默认为true)。它 不会 删除 .gitignore中指定的未跟踪的文件。
{% codeblock %}git clean -f <path>{% endcodeblock %} 
移除未跟踪的文件，但限制在某个路径下。
{% codeblock %}git clean -df{% endcodeblock %} 
移除未跟踪的文件，以及目录。
{% codeblock %}git clean -xf{% endcodeblock %} 
移除当前目录下未跟踪的文件，以及Git一般忽略的文件。

* ### 讨论
如果你在本地仓库中作死之后想要毁尸灭迹，git reset --hard和git clean -f是你最好的选择。运行这两个命令使工作目录和最近的提交相匹配，让你在干净的状态下继续工作。
git clean命令对于build后清理工作目录十分有用。比如，它可以轻易地删除C编译器生成的.o和.exe二进制文件。这通常是打包发布前需要的一步。-x命令在这种情况下特别方便。
请牢记，和git reset一样， git clean是仅有的几个可以永久删除提交的命令之一，所以要小心使用。事实上，它太容易丢掉重要的修改了，以至于Git厂商 强制 你用-f标志来进行最基本的操作。这可以避免你用一个git clean就不小心删除了所有东西。

* ### 例子
下面的例子清除了工作目录中的所有更改，包括新建还没加入缓存的文件。它假设你已经提交了一些快照，准备开始一些新的实验。
{% codeblock %}
# 编辑了一些文件
# 新增了一些文件
# 『糟糕』

# 将跟踪的文件回滚回去
git reset --hard

# 移除未跟踪的文件
git clean -df
{% endcodeblock %} 
在执行了reset/clean的流程之后，工作目录和缓存区和最近一次提交看上去一模一样，而git status会认为这是一个干净的工作目录。你可以重新来过了。
注意，不像git reset的第二个栗子，新的文件没有被加入到仓库中。因此，它们不会受到git reset --hard的影响，需要git clean来删除它们。

---
## git checkout
{% codeblock %}git checkout <commit> <file>{% endcodeblock %} 
查看文件之前的版本。它将工作目录中的<file>文件变成<commit>中那个文件的拷贝，并将它加入缓存区。
{% codeblock %}git checkout <commit>{% endcodeblock %} 
更新工作目录中的所有文件，使得和某个特定提交中的文件一致。你可以将提交的哈希字串，或是标签作为<commit>参数。这会使你处在分离HEAD的状态。

* ### 讨论

版本控制系统背后的思想就是『安全』地储存项目的拷贝，这样你永远不用担心什么时候不可复原地破坏了你的代码库。当你建立了项目历史之后，git checkout是一种便捷的方式，来将保存的快照『加载』到你的开发机器上去。
检出之前的提交是一个只读操作。在查看旧版本的时候绝不会损坏你的仓库。你项目『当前』的状态在 master上不会变化。在开发的正常阶段，HEAD一般指向master或是其他的本地分支，但当你检出之前提交的时候，HEAD就不再指向一个分支了——它直接指向一个提交。这被称为
『分离HEAD』状态。
在另一方面，检出旧文件不影响你仓库的当前状态。你可以在新的快照中像其他文件一样重新提交旧版本。所以，在效果上，git checkout的这个用法可以用来将单个文件回滚到旧版本 。

* ### 例子

这个栗子假定你开始了一个疯狂的实验，但你不确定你是否想要保留它。为了帮助你决定，你想看一看你开始实验之前的项目状态。首先，你需要找到你想要看的那个版本的ID。
{% codeblock %}git log --oneline{% endcodeblock %} 
假设你的项目历史看上去和下面一样：
{% codeblock %}
b7119f2 继续做些丧心病狂的事
872fa7e 做些丧心病狂的事
a1e8fb5 对hello.py做了一些修改
435b61d 创建hello.py
9773e52 初始导入{% endcodeblock %} 
你可以这样使用git checkout来查看『对hello.py做了一些修改』这个提交：
{% codeblock %}git checkout a1e8fb5{% endcodeblock %}
这让你的工作目录和a1e8fb5提交所处的状态完全一致。你可以查看文件，编译项目，运行测试，甚至编辑文件而不需要考虑是否会影响项目的当前状态。你所做的一切 都不会 被保存到仓库中。为了继续开发，你需要回到你项目的『当前』状态：
{% codeblock %}git checkout master{% endcodeblock %} 
这里假定了你默认在master分支上开发，我们会在以后的分支模型中详细讨论。
一旦你回到master分支之后，你可以使用 git revert或git reset来回滚任何不想要的更改。

* ### 检出文件

如果你只对某个文件感兴趣，你也可以用git checkout来获取它的一个旧版本。比如说，如果你只想从之前的提交中查看hello.py文件，你可以使用下面的命令：
{% codeblock %}git checkout a1e8fb5 hello.py{% endcodeblock %} 
记住，和检出提交不同，这里 确实 会影响你项目的当前状态。旧的文件版本会显示为『需要提交的更改』，允许你回滚到文件之前的版本。如果你不想保留旧的版本，你可以用下面的命令检出到最近的版本：
{% codeblock %}git checkout HEAD hello.py{% endcodeblock %} 