title: linux 如何查找命令的路径
date: 2013-12-04

tags:
 - linux
categories:
 - 工具/类库
---
linux 下，我们常使用 cd ,grep,vi 等命令，有时候我们要查到这些命令所在的位置，如何做呢？

linux下有2个命令可完成该功能：which ,whereis

 which 用来查看当前要执行的命令所在的路径。
whereis 用来查看一个命令或者文件所在的路径。


which命令的原理：在PATH变量指定的路径中，搜索某个系统命令的位置，并且返回第一个搜索结果。也就是说，使用which命令，就可以看到某个系统命令是否存在，以及执行的到底是哪一个位置的命令。

which命令的使用实例：

　　$ which grep

whereis命令原理：只能用于程序名的搜索，而且只搜索二进制文件（参数-b）、man说明文件（参数-m）和源代码文件（参数-s）。如果省略参数，则返回所有信息。

whereis命令的使用实例：

　　$ whereis grep

下面举个例子来说明。加入你的linux系统上装了多个版本的java。如果你直接在命令行敲命令 "java -version" ，会得到一个结果。但是，你知道是哪一个路径下的java在执行吗？如果想知道，可以用 which 命令：

which java

返回的是 PATH路径中第一个JAVA的位置，也就是JAVA命令默认执行的位置

如果使用命令： whereis java

那么你会得到很多条结果，因为这个命令把所有包含java（不管是文件还是文件夹）的路径都列了出来。