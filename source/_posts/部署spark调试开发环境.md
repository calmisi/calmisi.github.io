---
title: 部署spark调试开发环境
categories: Spark
date: 2016-10-28 21:02:13
tags:
- Spark本地调试
---

**本文摘取自网络，非本人原创**

记录一下关于 spark 开发的相关环境配置, 以及与 intellij 的集成.

本文针对的是需要了解 spark 源码 的情况下的开发环境配置. 如果只是需要写 spark job, 而并不想 trace 到源码里面去看运行上下文, 那么有很多资料讲这个的了: 无非是下载 spark 的 jar, 新建一个 scala/python/R 项目, 把这个 jar 设置成依赖就可以开始写了. 具体怎么运行 spark job, 在官方文档中已经写得很清楚了. 本文记录的 不是 后一种情况
由于实验室工作需要于是开始学习 spark 的源码. 之前都是东一坨西一块地配置, 好了也不知道为什么, 没好也只会到处找相关 blog, 然后关闭项目再从头 import 试试. 大部分能找到的中文资料还是比较没用, 或者只能干掉一两件事, 于是坐下来好好看了看 maven 的入门文档 (因为 spark 开发组偏向于 maven 做项目管理, 感觉钦定的比较🐵), 从头到尾地把大部分入门 spark 开发的 intellij 配置给手动操作了一遍, 目前算是一篇milestone文档.
<!-- more -->
可能的前置要求

能够翻墙(最好能够翻墙)
Windows 没有充分测试, 建议 POSIX 环境
脑补能力
配置代码阅读环境 符号跳转
获取 spark 源码
从这里选择一个镜像, 下载源码 (我用的是spark-1.6.1.tgz).

## 编译spark源码
这里可以配置国内的Maven镜像，提高Mevan获取速度。
以下是OSChina的镜像
{% codeblock lang:xml %}
 <mirrors>
     <mirror>
        <id>nexus-osc</id>
        <mirrorOf>central</mirrorOf>
        <name>Nexus osc</name>
        <url>http://maven.oschina.net/content/groups/public/</url>
    </mirror>
    <mirror>
          <id>nexus-osc-thirdparty</id>
          <mirrorOf>thirdparty</mirrorOf>
          <name>Nexus osc thirdparty</name>
          <url>http://maven.oschina.net/content/repositories/thirdparty/</url>
    </mirror>
</mirrors>
{% endcodeblock %}

以下是阿里云的镜像
{% codeblock lang:xml %}
<mirrors>
	<mirror>
	<id>nexus-aliyun</id>
	<mirrorOf>*</mirrorOf>
	<name>Nexus aliyun</name>
	<url>http://maven.aliyun.com/nexus/content/groups/public</url>
	</mirror>
</mirrors>
{% endcodeblock %}

然后用mvn build
build/mvn -Pyarn -Phadoop-2.4 -Dhadoop.version=2.4.0 -DskipTests clean package 执行一次干净的从头编译, 包括所有 spark submodules, 不运行测试

在官方文档里面有更详细的介绍

---

## 配置Intellij IDEA
编译完Spark，在导入Intellij IDEA之前，先配置好IDEA.

** a. 配置SBT**
FILE-->other settings-->Build,Execution,Deployment-->Build Tools-->SBT

* JVM-->Custom
选择你自己的Java 安装目录

* Launcher(sbt-launch.jar)-->Custom
选择你自己的sbt-launch.jar

** b. 配置Maven**
File-->Build,Execution,Deployment-->Build Tools-->Maven

* 修改Maven home directory, 设置成你的安装目录
* 修改User settings file,设置成你的安装目录下面conf中settings.xml

File-->Build,Execution,Deployment-->Build Tools-->Maven-->Runner

* VM Options 加入如下参数
`-Xmx2g -XX:MaxPermSize=512M -XX:ReservedCodeCacheSize=512m`

* JRE
设置成自己想要的Java版本

** c. 配置Scala**
File-->Settings-->Build,Execution,Deployment-->Compiler-->Scala Compiler

* Incrementality type
设置成SBT.

File-->Settings-->Languages & Frameworks-->Scala Compile Server

* 选中 Use external compile server for scala
* JVM SDK 设置成自己需要的。

---

## 导入Intellij
导入 Intellij 项目

在 Intellij 什么项目也没有打开的小页面, 选择

Import Project
    -> /path/to/spark-1.6.1/
    -> 单选 Import project from external model[Maven]
    -> 勾选 Search for projects recursively
    -> Next -> Next -> Next -> Next -> Finish
配置依赖Maven

等待 Intellij 下方状态条显示 Index 和 Resolve Dependencies 等工作结束, 查看左边Project视图最下方的External Libraries, 展开后应该只有JDK

在Project视图中, 右键点击根目录 (spark-1.6.1) 下的pom.xml文件, 选择 Maven -> Reimport, 完成后在External Libraries内会找到大量 spark 项目的 maven 依赖

如果没有执行之前的编译操作, 这一步大概并不能找到 maven 依赖Scala

随便打开一个 scala 文件, 比如examples/src/main/scala/org/apache/spark/examples/SparkPi, 编辑器右上角会提示设置 scala sdk, 建议设置为2.10.5版本

如果下拉框里面没有 scala sdk, 那么创建一个, 如果创建窗口里面仍然没有, 点击Download下载一个, 注意版本别太高, 行为未知
到这个地方, Intellij 已经解析了需要的依赖的symbol, 跳转功能正常, 已经可以阅读代码了. 接下来的配置是为了更好地编写调试代码

配置代码开发环境 编译->打包->运行
改动后快速编译

## maven快速编译
如果每次都使用之前的编译整个项目的mvn命令进行编译的话, 每天工作10小时大概能改20多次代码💩. 我想帮老板多干点活, 于是搬砖之前先去考察下有没有解决方案. 这里有一个能用的方案, 有更好的再更新.

Intellij 提供了比较好的 maven GUI 集成, 所以我们可以通过鼠标和快捷键来运行maven lifecycle/goal. 在 Intellij 里面打开View -> Tool Windows, 此时右侧会有Maven Projects按钮, 点击可以打开能够执行的 maven lifecycle/goal. 比如我更改了 examples/.../SparkPi.scala文件, 然后想重新编译打包运行, 那么我找到Maven Projects -> Spark Project Examples -> Plugins, 先双击运行scala -> scala:compile, 完成之后再双击运行jar -> jar:jar, 可以看到编译出来了对应的 jar (在console里面注意SUCCESS之前的信息, 这个 jar 是需要在运行spark-submit的时候指定的).

当然为了能够把这两件事一起做, 更重要的是跟下一步远程调试一起做掉, 最好的方式还是写进脚本在 CLI 运行. 我们在spark-1.6.1根目录下运行mvn -pl examples scala:compile jar:jar, 告诉 maven 只在 examples 这个 submodule 下运行scala:compile和jar:jar两个 maven goals, 能够看到类似输出

{% codeblock lang:shell%}
~/work/spark-1.6.1  mvn -pl examples scala:compile jar:jar
[INFO] Scanning for projects...
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] Building Spark Project Examples 1.6.1
[INFO] ------------------------------------------------------------------------
[INFO]
[INFO] --- scala-maven-plugin:3.2.2:compile (default-cli) @ spark-examples_2.10 ---
[INFO] Using zinc server for incremental compilation
[info] Compile success at May 11, 2016 11:42:13 PM [0.390s]
[INFO]
[INFO] --- maven-jar-plugin:2.6:jar (default-cli) @ spark-examples_2.10 ---
[INFO] Building jar: /Users/dragonly/work/spark-1.6.1/examples/target/spark-examples_2.10-1.6.1.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 10.931 s
[INFO] Finished at: 2016-05-11T23:42:15+08:00
[INFO] Final Memory: 54M/762M
[INFO] ------------------------------------------------------------------------
{% endcodeblock %}
可以看到我们的改动后的 examples 的代码被编译打包到了spark-1.6.1/examples/target/spark-examples_2.10-1.6.1.jar.

## 远程调试

废话

由于 spark job 的运行需要 spark 环境, 不同于以往的单机程序, 需要借助bin/spark-submit脚本提交给 spark 去运行 (查看脚本源码可以知道, 启动前要先做一些环境检查和初始化, 然后调用 launcher 和 deploy 相关 class 去加载和运行提交的 jar). 因此断点调试起来会比较麻烦, 因为这个 server 端跟我们写的代码是两个 process (广义), 所以需要类似 gdb 的远程调试一样的功能. 原理上讲是给出一堆运行参数, 让 jvm 运行 spark 的时候, 开一个调试端口, 然后在 Intellij 这边用 debugger 远程连过去进行调试. 虽然说是”远程”, 但是在这种情况下其实就是 localhost 上开的端口而已.

Intellij 创建远程调试的 Debug Configuration

进入Run -> Edit Configurations, 单击左上角+, 新建一个 Remote Configuration, 名字随便改一下, 其余留作默认. 点击OK之前, 复制Configurationtab下的Command line arguments for running remote JVM里面的内容, 之后会用到这个参数, 并做如下更改

-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005
改成
-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005
然后点OK创建这个 Configuration.

命令行

先运行mvn -pl examples scala:compile jar:jar进行编译和打包, 然后运行
{% codeblock lang:shell%}
bin/spark-submit \
--driver-java-options "-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005" \
--class org.apache.spark.examples.SparkPi \
examples/target/spark-examples_2.10-1.6.1.jar
{% endcodeblock %}
注意到第二行的参数是告诉 spark 运行的时候设置额外的 java 参数, 就是上面复制出来的参数, 改成suspend=y是告诉 spark 启动之后暂停运行, 等待 debugger 连接之后才开始运行, 方便我们加断点.

然后 CLI 会显示Listening for transport dt_socket at address: 5005, 表示 spark 正在等待 debugger 连接.

此时只需要在 Intellij 上打开SparkPi.scala文件, 加上断点, 再点Run -> Debug 'Remote', 就能开始单步追踪调试了.