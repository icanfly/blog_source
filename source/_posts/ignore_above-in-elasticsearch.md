---
title: Elasticsearch中ignore_above的作用
tags:
  - 搜索引擎
  - lucene
  - elasticsearch
categories:
  - 原创文章
originContent: ''
toc: false
date: 2019-02-13 16:57:50
author:
thumbnail:
blogexcerpt:
---

ignore_above一般配合keyword类型使用，指示该字段的最大索引长度（即超过该长度的内容将不会被索引），对于超过ignore_above长度的字符串，analyzer不会进行索引分析，所以超过该长度的内容将不会被搜索到。这个选项主要对not_analyzed字段有用，这些字段通常用来进行过滤、聚合和排序。而且这些字段都是整体存在的，不需要进行索引分析处理，所以一般不会允许在这些字段中索引过长的项。

当在设置索引的mapping设置后，如果keyword字段没有显式设置ignore_above的值，则ES会默认设置该长度为256，当然你可以在后续的操作中修改这个值，但是修改后需要重建索引才能让以前不满足的值重新变得满足而被索引。

1. 不满足该设置的文档会被保存，但是该字段值不会被索引
2. 通过查询该字段的值时该文档不会被索引到，并被输出
3. 通过其它字段的查询时，如果该文档满足条件会被索引到，并被输出
4. 该设置选项并不影响文档的保存，只影响文档的字段是否被索引和搜索

注：keyword类型的字段的最大长度限制为32766个UTF-8字符，text类型的字段对字符长度没有限制

所以在设置keyword类型的ignore_above值时应该先遵守keyword本身的值最大长度限制。

> ignore_above 值表示字符个数，但是 Lucene 计算的是字节数。如果你使用包含很多非 ASCII 字符的 UTF-8 文本，你应该将这个限制设置成 32766 / 3 = 10922 因为 UTF-8 字符可能最多占用 3 个字节。