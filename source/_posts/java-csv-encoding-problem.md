title: java输出csv文件中文乱码的问题
date: 2017-04-20

tags:
 - 基础问题
 - 最佳实践
categories:
 - 工具/类库
---



在开发程序的时候，通过java程序输出csv文件，采用utf-8编码，然后用excel打开，发现文件中的中文全部乱码了。

在网上搜索了各种手工解决办法，无外乎就是将文件打开另存为“ANSI”格式。

然而真正程序能解决的是如下：

```java
//在文件中增加BOM，详细说明可以Google,该处的byte[] 可以针对不同编码进行修改
out.write(new byte[] { (byte) 0xEF, (byte) 0xBB,(byte) 0xBF });
```

在文件的输出流中增加BOM信息，中文乱码得以解决。