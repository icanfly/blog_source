title: IDEA 快捷键
date: 2013-09-26

tags:
 - IDE
 - IDEA
categories:
 - 转载文章
---

1. Ctrl + Space  
完成类、方法、变量名称的自动输入,这个快捷键是我最经常使用的快捷键了，它可以完成类、方法、变量名称的自动录入，很方便  
2. Ctrl + N（Ctrl + Shift + N）  
跳转到指定的java文件（其它文件）这个功能很方便，至少我不用每回都在一长串的文件列表里找寻我想要编辑的类文件和jsp文件了  
3. Ctrl + B  
跳转到定义处这个就不用多说了，好象是个IDE就会提供的功能  
4. Ctrl + Alt + T  
用*来围绕选中的代码行（ * 包括if、while、try catch等）这个功能也很方便，把我以前要做的：①先写if-else，②然后调整代码的缩进格式，还要注意括号是否匹配了，现在用这个功能来做，省事多了（不过让我变得越来越懒了）  
5. Ctrl + Alt + B  
跳转到方法实现处这个也算是很普遍的功能了，就不多说了。  
6. Ctrl + W  
按一个word来进行选择操作在IDEA里的这个快捷键功能是先选择光标所在字符处的单词，然后是选择源  
代码的扩展区域。举例来说，对下边这个语句java.text.SimpleDateFormat formatter = new java.text.SimpleDateFormat("yyyy-MM-dd HH:mm");当光标的位置在双引号内的字符串中时，会先选中这个字符串，然后是等号右边的表达式，再是整个句子。我一般都是在对代码进行重新修改的时候使用  
它来选择出那些长长的复合表达式，很方便：）  
7. Shift + F1  
在浏览器中显示指定的java docs,这个也应该是几乎所有的java ide都提供的功能，就不多说了。  
8. Ctrl + Q  
在editor window中显示java docs这个功能很方便--因为有时仅仅是忘记了自己编写的方法中的某个参数的含义，此时又不想再起一个浏览器来查看java doc，此时这个功能的好处就体现出来了  
9. Ctrl + /  
注释/反注释指定的语句,这个功能很象PB中提供的一个功能，它可以注释和反注释你所选择的语句（使用单行注释符号"//"），你也可以用Ctrl + Shift + / 来进行多行语句的注释（即使用多行注释符号"/* ... */"）  
10. F2/Shift + F2  
跳转到下/上一个错误语句处IDEA提供了一个在错误语句之间方便的跳转的功能，你使用这个快捷键可以快捷在出错的语句之间进行跳转。
<!--more-->  
11. Shift + F6  
提供对方法、变量的重命名对IDEA提供的Refector功能我用得比较少，相比之下这个功能是我用得最多的了。对于这个功能没什么可说的了，确实很方便，赶快试一试吧。  
12. Ctrl + Alt + L  
根据模板格式化选择的代码,根据模板中设定的格式来format你的java代码，不过可惜的是只对java文件有效  
13. Ctrl + Alt + I  
将选中的代码进行自动缩进编排这个功能在编辑jsp文件的时候也可以工作，提供了一个对上边格式化代码功能的补充。  
14. Ctrl + Alt + O  
优化import自动去除无用的import语句，蛮不错的一个功能。  
15. Ctrl + ]/[  
跳转到代码块结束/开始处,这个功能vi也有，也是很常用的一个代码编辑功能了。
16. Ctrl+E  
可以显示最近编辑的文件列表  
17. Shift+Click  
可以关闭文件  
18. Ctrl+Shift+Backspace  
可以跳转到上次编辑的地方  
19. Ctrl+F12  
可以显示当前文件的结构  
20. Ctrl+F7  
可以查询当前元素在当前文件中的引用，然后按F3可以选择  
21. Ctrl+Shift+N  
可以快速打开文件  
22. Alt+Q  
可以看到当前方法的声明  
23. Ctrl+P  
可以显示参数信息  
25. Alt+Insert  
可以生成构造器/Getter/Setter等  
26. Ctrl+Alt+V  
可以引入变量。例如把括号内的SQL赋成一个变量  
27. Alt+Up and Alt+Down  
可在方法间快速移动  
28. Alt+Enter  
可以得到一些Intention Action，例如将”==”改为”equals()”  
29. Ctrl+Shift+Alt+N  
可以快速打开符号  
30. Ctrl+Shift+Space  
在很多时候都能够给出Smart提示  
31. Alt+F3  
可以快速寻找  
32. Ctrl+O  
可以选择父类的方法进行重写  
33. Ctrl+Alt+Space  
是类名自动完成  
34. Ctrl+J  
Live Templates!  
35. Ctrl+Shift+F7  
可以高亮当前元素在当前文件中的使用  
36. Ctrl+Alt+Up /Ctrl+Alt+Down  
可以快速跳转搜索结果  
37. Ctrl+Shift+J  
可以整合两行  
38. Alt+F8是计算变量值  

Ctrl+D 复制上一行或复制选定  
Ctrl+Alt+L 格式化代码  
Alt+Shift+Insert 列编辑  

装上UpperLowerCapitalize后  
Alt+P // to uppercase  
Alt+L // to lowercase  
Alt+C // 首字母大写 