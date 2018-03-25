title: 如何写好一个Git Commit Message
date: 2017-08-21
tags:
 - git
 - github
categories:
 - 翻译文章
toc: true
---

原文地址：[How to Write a Git Commit Message](https://chris.beams.io/posts/git-commit/)

# 介绍：为什么好的提交信息很重要

如果您随机浏览一个Git存储库的日志，您可能会发现其提交消息或多或少是一团糟。 例如，从早期的Spring提交日志来看这些问题点：

```shell
$ git log --oneline -5 --author cbeams --before "Fri Mar 26 2009"

e5f4b49 Re-adding ConfigurationPostProcessorTests after its brief removal in r814. @Ignore-ing the testCglibClassesAreLoadedJustInTimeForEnhancement() method as it turns out this was one of the culprits in the recent build breakage. The classloader hacking causes subtle downstream effects, breaking unrelated tests. The test method is still useful, but should only be run on a manual basis to ensure CGLIB is not prematurely classloaded, and should not be run as part of the automated build.
2db0f12 fixed two build-breaking issues: + reverted ClassMetadataReadingVisitor to revision 794 + eliminated ConfigurationPostProcessorTests until further investigation determines why it causes downstream tests to fail (such as the seemingly unrelated ClassPathXmlApplicationContextTests)
147709f Tweaks to package-info.java files
22b25e0 Consolidated Util and MutableAnnotationUtils classes into existing AsmUtils
7f96f57 polishing
```

与同一存储库中的这些最近的提交进行比较：

```shell
$ git log --oneline -5 --author pwebb --before "Sat Aug 30 2014"

5ba3db6 Fix failing CompositePropertySourceTests
84564a0 Rework @PropertySource early parsing logic
e142fd1 Add tests for ImportSelector meta-data
887815f Update docbook dependency and generate epub
ac8326d Polish mockito usage
```

哪一个你更愿意去阅读呢？

前者在长度和形式上变化很大; 后者简洁而一致。 前者是默认情况; 后者不会偶然发生。

虽然许多Git存储库的日志看起来像前者，但也有例外。 Linux内核和Git本身就是很好的例子。 看看Spring Boot，或者由Tim Pope管理的任何存储库。

<!-- more -->

对这些存储库的贡献者来说，精心设计的Git提交消息是向同事开发人员（以及其将来的自己）传达关于变更的上下文的最佳方式。 差异会告诉你什么改变了，但只有提交消息可以正确地告诉你为什么。 彼得·赫特雷尔（Peter Hutterer）表示：

> Re-establishing the context of a piece of code is wasteful. We can’t avoid it completely, so our efforts should go to reducing it [as much] as possible. Commit messages can do exactly that and as a result, a commit message shows whether a developer is a good collaborator.

如果你没有太多思考什么是一个很好的Git提交消息，可能是没有花费太多时间使用git日志和相关工具。 这里有一个恶性循环：因为提交历史是非结构化和易变的，所以不会花太多时间使用或在意它。 并且因为它没有得到使用或在意，所以它仍然是非结构化和易变的。

但是一个很好的提交日志是一个优美而有用的事情。 git blame, revert，rebase，log，shortlog等子命令。 审查他人的commits和pull requests成为值得做的事情。 了解几个月或几年前为什么会发生的事情不仅变得可能而且会很有效率。

一个项目的长期成功取决于其可维护性（除其他外），维护者的工具比其项目的日志更强大。 值得花时间学习如何妥善照顾。 最初的麻烦，很快就会成为习惯，最终成为所有参与者的骄傲和生产力的源泉。

在这篇文章中，我只讨论保持健康的提交历史的最基本的元素：如何编写一个单独的提交信息。 还有其他一些重要的做法，如我不在这里处理的强调。 也许我会在随后的一篇文章中这样做。

大多数编程语言对于什么构成习语风格，即命名，格式化等都有完整的约定。 当然，这些惯例有差异，但是大多数开发人员认为，选择一个并坚持下去，远远超过每个人自己独特个性化的风格所造成的混乱。

一个团队对其提交日志的方法应该没有什么不同。 为了创建一个有用的修订历史，团队应首先同意至少定义以下三件事情的提交消息约定：

- 样式

标记语法，包含边距，语法，大小写，标点符号。 拼出这些东西，消除猜测，尽可能简单。 最终结果将是一个非常一致的日志，不仅是阅读的乐趣，而且实际上可以定期阅读。

- 内容

提交消息的主体（如果有）包含什么样的信息？ 它不包含什么？

- 元数据

如何引用跟踪ID，提取请求号等？

幸运的是，实际上关于怎么让Git提交消息有一个很完善的约定没有什么是你需要重新规划或者发明创造的。 其中许多是以某些Git命令的功能为基础的，只要按照下面的七条规则，你就可以像个专业人士一样提交你的Git日志。

# 一个伟大的Git提交消息的七个规则

- 将摘要与详细内容用空行分开
- 将摘要行限制为50个字符
- 将摘要首字母大写
- 摘要结尾处不要使用标点符号
- 在摘要行中使用必要的语气
- 正文内容以72个字符为限进行换行
- 使用详细消息部分来解释what、why、how

例如：

```
Summarize changes in around 50 characters or less

More detailed explanatory text, if necessary. Wrap it to about 72
characters or so. In some contexts, the first line is treated as the
subject of the commit and the rest of the text as the body. The
blank line separating the summary from the body is critical (unless
you omit the body entirely); various tools like `log`, `shortlog`
and `rebase` can get confused if you run the two together.

Explain the problem that this commit is solving. Focus on why you
are making this change as opposed to how (the code explains that).
Are there side effects or other unintuitive consequences of this
change? Here's the place to explain them.

Further paragraphs come after blank lines.

 - Bullet points are okay, too

 - Typically a hyphen or asterisk is used for the bullet, preceded
   by a single space, with blank lines in between, but conventions
   vary here

If you use an issue tracker, put references to them at the bottom,
like this:

Resolves: #123
See also: #456, #789
```

## 将摘要与详细内容用空行分开

> Though not required, it’s a good idea to begin the commit message with a single short (less than 50 character) line summarizing the change, followed by a blank line and then a more thorough description. The text up to the first blank line in a commit message is treated as the commit title, and that title is used throughout Git. For example, Git-format-patch(1) turns a commit into email, and it uses the title on the Subject line and the rest of the commit in the body.

首先，并不是每一个提交日志都要求一个摘要和一个详细描述。 有时单行很好，特别是当变化如此简单，不需要进一步的上下文。 例如：
```
Fix typo in introduction to user guide
```
没有更多的需要说 如果读者想知道错字是什么，她可以简单地看一下变化本身，即使用git show或git diff或git log -p。
如果你想在命令行中提交了这样的内容，那么可以使用-m选项来简单地提交：
```
$ git commit -m"Fix typo in introduction to user guide"
```
但是，当一个提交有一点解释和上下文的时候，你需要写一个正文。 例如：
```
Derezz the master control program

MCP turned out to be evil and had become intent on world domination.
This commit throws Tron's disc into MCP (causing its deresolution)
and turns it back into a chess game.
```
使用-m选项提交带有主体的消息不是很容易编写。 你最好在适当的文本编辑器中编写消息。 如果您还没有在命令行中设置一个用于Git的编辑器，请阅读[Pro Git](https://git-scm.com/book/en/v2/Customizing-Git-Git-Configuration)的这一部分。

在任何情况下，提交日志摘要与详细描述的分离在浏览日志时都会付出代价。 这是完整的日志条目：
```
$ git log
commit 42e769bdf4894310333942ffc5a15151222a87be
Author: Kevin Flynn <kevin@flynnsarcade.com>
Date:   Fri Jan 01 00:00:00 1982 -0200

 Derezz the master control program

 MCP turned out to be evil and had become intent on world domination.
 This commit throws Tron's disc into MCP (causing its deresolution)
 and turns it back into a chess game.
```
现在使用git log --online，打印出主题行：
```
$ git log --oneline
42e769 Derezz the master control program
```
或者使用git shortlog，这个命令会按用户进行分组汇总提交，再次显示简洁的主题行：
```
$ git shortlog
Kevin Flynn (1):
      Derezz the master control program

Alan Bradley (1):
      Introduce security program "Tron"

Ed Dillinger (3):
      Rename chess program to "MCP"
      Modify chess program
      Upgrade chess program

Walter Gibbs (1):
      Introduce protoype chess program
```

在Git中还有其他一些上下文，其中摘要行和主体内容之间的区别是在两者之间没有空白行的情况下都是不能正常工作的。

## 将主题行限制为50个字符

50个字符不是一个极限，只是一个经验法则。 保持这个长度的主题确保了它们的可读性，并迫使作者想了解一下最简洁的方式来解释发生了什么。

> 提示：如果你很难总结，你可能是一次性提交了太多更改。 你需要争取将这次提交变成多个原子提交（一个提交对应一个主题域的修改）。

GitHub的UI完全了解这些约定。 如果你超过50个字符的限制，它会发出警告：

{% asset_img zyBU2l6.png %}

并且将使用省略号截断长度超过72个字符的任何主题行：

{% asset_img 27n9O8y.png %}

所以50个字符是合适的，最大不要超过72的硬限制。

## 摘要行首字母大写

这听起来很简单。 用大写字母开始所有主题行。

例如：

<span style="color:green">Accelerate to 88 miles per hour</span>

代替：

<span style="color:red">accelerate to 88 miles per hour</span>

## 摘要结尾处不要使用标点符号

主题行中不需要拖尾的标点符号。 此外，当你试图把它们保持在50个字符或更少时，空间是宝贵的。

例如:

<span style="color:green">Open the pod bay doors</span>

代替:

<span style="color:red">Open the pod bay doors.</span>

## 在摘要行中使用必要的语气

命令式的语气只是意味着“说出来或写出来，就像给出命令或指示”一样。 几个例子：

收拾你的房间
关门
把垃圾带出去
你正在阅读的七个规则中的每一个现在都写在命令中（“将消息内容以72个字符换行”等）。

命令可能听起来有点粗鲁; 这就是为什么我们不经常使用它。 但它却是比较适合的Git日志主题行的。 这样做的一个原因是，当Git自己代表你创建一个提交时，它就会使用这个命令。

例如，使用git merge时创建的默认消息如下：
```
Merge branch 'myfeature'
```
当使用git revert时：
```
Revert "Add the thing with the stuff"

This reverts commit cc87791524aedd593cff5a74532befe7ab69ce9d.
```
或者当点击GitHub PR请求上的“合并”按钮时：
```
Merge pull request #123 from someuser/somebranch
```
所以当你写下你的提交信息在命令行中时，你需要遵循Git自己的内置约定。 例如：
- Refactor subsystem X for readability
- Update getting started documentation
- Remove deprecated methods
- Release version 1.0.0

起初写这种方式可能有点尴尬。 我们更习惯于以指示性的心情来说话，这是关于报告事实。 这就是为什么提交信息通常最终会如下所示：

- Fixed bug with Y
- Changing behavior of X

有时候，提交消息将作为其内容的描述：

- More fixes for broken stuff
- Sweet new API methods

为了消除任何混乱，这里有一个简单的规则，每次都可以正确使用。

正确形成的Git提交主题行应始终能够完成以下句子：

- If applied, this commit will your subject line here

例如：

- If applied, this commit will <span style='color:green'>refactor subsystem X for readability</span>
- If applied, this commit will <span style='color:green'>update getting started documentation</span>
- If applied, this commit will <span style='color:green'>remove deprecated methods</span>
- If applied, this commit will <span style='color:green'>release version 1.0.0</span>
- If applied, this commit will <span style='color:green'>merge pull request #123 from user/branch</span>

请注意理解为什么这对于其他非必要形式不起作用：

- If applied, this commit will <span style='color:red'>fixed bug with Y</span>
- If applied, this commit will <span style='color:red'>changing behavior of X</span>
- If applied, this commit will <span style='color:red'>more fixes for broken stuff</span>
- If applied, this commit will <span style='color:red'>sweet new API methods</span>

> 请记住：只有在主题行中，以上的规约才是重要的。 当你在书写详细描述时，你可以放松这个限制。

## 正文内容以72个字符为限进行换行

Git不会自动包装文本。 当您写入提交消息的正文时，您必须记住其正确的边距，并手动换行。

建议是以72个字符做这个，所以Git有足够的空间来缩进文字，同时保持一切不超过80个字符。

一个好的文本编辑器可以提供帮助的。 例如，当您编写Git提交时，可以轻松地将Vim配置为包含72个字符的文本。 然而，传统上，IDE在提交消息中为文本包装提供智能支持非常可怕（尽管在最近的版本中，IntelliJ IDEA终于得到了更好的解决）。

## 使用详细消息部分来解释what、why、how

Bitcoin Core的这个提交是一个很好的例子，说明改变了什么，为什么要做这样的改变：
```
commit eb0b56b19017ab5c16c745e6da39c53126924ed6
Author: Pieter Wuille <pieter.wuille@gmail.com>
Date:   Fri Aug 1 22:57:55 2014 +0200

   Simplify serialize.h's exception handling

   Remove the 'state' and 'exceptmask' from serialize.h's stream
   implementations, as well as related methods.

   As exceptmask always included 'failbit', and setstate was always
   called with bits = failbit, all it did was immediately raise an
   exception. Get rid of those variables, and replace the setstate
   with direct exception throwing (which also removes some dead
   code).

   As a result, good() is never reached after a failure (there are
   only 2 calls, one of which is in tests), and can just be replaced
   by !eof().

   fail(), clear(n) and exceptions() are just never called. Delete
   them.
```

看看完整的差异，只是想想作者通过花时间在这里和现在提供这个上下文来节省他们和未来的提交者多少时间。 如果他没有，它可能会永远失去。

在大多数情况下，你可以省略有关如何进行更改的详细信息。 代码在这方面通常是不言自明的（如果代码如此复杂，需要在散文中解释，那就是源代码注释）。 只要重点明确你为什么首先做出改变的原因 - 变革之前工作的方式（以及这是什么问题），现在的工作方式，以及为什么决定以你所做的方式解决。

未来的维护者，谢谢你可能是你自己！

# 提示

<b>学会喜爱命令行。 离开IDE。</b>

由于多种原因，拥抱命令行是明智之举。 Git非常强大，IDE也是这样，但是每个IDE都有不同的方式。 我每天都使用一个IDE（IntelliJ IDEA），并广泛使用过其它的（Eclipse），但我从未看过Git的IDE集成可以像命令行一样强大。

某些与Git相关的IDE功能是非常宝贵的，例如在删除文件时调用git rm，并在重命名时使用git进行正确的操作。 当你开始尝试通过IDE提交，合并，重新生成或进行复杂的历史分析时，所有的事情都会分崩离析，导致你无法在IDE上很好的工作。

当谈到发挥Git的全部力量时，它就是命令行。

请记住，无论是使用Bash还是Zsh或Powershell，都有一些标辅助功能脚本，可以辅助进行一些命令提示。

# Read Pro Git

[Pro Git](https://git-scm.com/book/en/v2)可以在线免费获得，这太棒了。
