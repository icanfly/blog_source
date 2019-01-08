---
title: jackson-ctrl-char-problem-resovle
tags:
  - java
  - json
  - jackson
categories:
  - 原创文章
originContent: ''
toc: false
author:
thumbnail:
blogexcerpt:
---

在使用swagger传递json数据的时候，突然报错：
```
org.codehaus.jackson.JsonParseException: Illegal unquoted character ((CTRL-CHAR, code 10))
```
意思是说使用了在json内容中使用了控制字符。而这个code 10是说使用了换行字符。

解决方法:

方式1. 使用显式转义方式

使用\n代替控制性转行（不可打印）字符

方式2. 配置Jackson
```java
ObjectMapper mapp = new ObjectMapper();
mapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_CONTROL_CHARS, true);

```

