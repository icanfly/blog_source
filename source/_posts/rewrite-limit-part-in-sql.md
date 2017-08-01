title: SQL小记-改写SQL中Limit限制
date: 2017-08-01
tags:
 - SQL
categories:
 - 技术

---

最近在做一款大数据查询系统，该系统提供给数据分析人员一个用户交互界面，让用户提供SQL语言进行交互查询，前台应用提供一些辅助功能，这样的数据查询系统要优于通过纯console的方式进行查询，查询平台提供了用户管理，权限管理，交互易用性扩展，多引擎框架支持等。

最近出现的一个问题是用户提交的SQL很大部分时间上并没有携带Limit限制返回条数，导致产生了大量的数据查询浪费，因为在界面上是不会展示所有的结果数据，一般我们在前台页面上只展现少量数据，比如只展现前面100条，而这100条在很大部分情况下已经满足了分析师同学的数据分析需求，然而不太规范的SQL编写可能导致无limit关键字而导致大量的查询，所以当前的一个需求是对用户提交的SQL进行改写，将SQL中存在limit限制的地方进行判断，如果没有设置limit数或者limit大于最大限制，设置成最大限制，限制数据的查询量和返回量。

<!-- more -->

# 思路

因为SQL是动态变化的，想要解析出limit的话，可能需要解析整个SQL语句成SQL抽象语法树，然后遍历得到limit计算其值得到，但是对于目前这个需求，有个比较取巧的地方在于对于标准的SQL语句，Limit关键字始终位于语句最末尾的地方。形如：`select * from a limit 100 `这样的SQL，可以抽象出一个正则表达式，用这个表达式去匹配用户提交的SQL，并解析出对应的limit数，并对解析的SQL进行limit替换就可以了

# 正则表达式

我们要编写的其实是对SQL中limit的匹配关系解析，而SQL中limit形式其实是有两种的，一种是形如`select * from a limit 100`，另一种是形如`select * from a limit 0,200`。我在做的时候设计成了两组正则，因分析师在分析的时候大部分会使用第一种形式，如果在匹配的时候优先匹配第一种情况，这也是短路优化的一种手段吧。

我们的正则表达式为：
```java

//Limit关键字匹配: limit 10
public static final String LIMIT_REPR_1 = "^(.|\\s)+\\s+(limit|LIMIT)\\s+(\\d+)$";

//Limit关键字匹配: limit 10,100
public static final String LIMIT_REPR_2 = "^(.|\\s)+\\s+(limit|LIMIT)\\s+(\\d+)\\s*,\\s*(\\d+)$";
```
这时有一个问题，LIMIT_REPR_2表达式出现了严重的性能问题，匹配会一直卡在这个表达式的match上，非常糟糕，慢得无法忍受。

需要对上面我随手写的正则表达式进行优化。 网上有一篇文章讲Java正则表达式优化的，链接在这里：[Java正则表达式优化](http://nspace.iteye.com/blog/1929568)，总体的思路是减少不确定的表达和累赘表达。

优化点：

- 表达式(.|\\s)+\\s+最开始我想表达的是允许limit关键字前面可以有任意的字符和空白，它其实可以简化成.+\\s+ (这里应该算是主要优化点，对于NFA的正则匹配方式优化前的表达式存在太多的折返导致性能极其低下)
- (limit|LIMIT)这部分去掉LIMIT，可以在匹配的时候将匹配串转成小写再匹配

优化后：

```java
//Limit关键字匹配: limit 10
public static final String LIMIT_REPR_1 = "^.+\\s+limit\\s+(\\d+)$";

//Limit关键字匹配: limit 10,100
public static final String LIMIT_REPR_2 = "^.+\\s+limit\\s+\\d+\\s*,\\s*(\\d+)$";

```



# 实现

```java
/**
 * SQL查询Limit改写器
 * Created by luopeng on 2017/8/1.
 */
public class QuerySqlLimiter {

	//Limit关键字匹配: limit 10
	public static final String LIMIT_REPR_1 = "^.+\\s+limit\\s+(\\d+)$";

	//Limit关键字匹配: limit 10,100
	public static final String LIMIT_REPR_2 = "^.+\\s+limit\\s+\\d+\\s*,\\s*(\\d+)$";

	private static Pattern p1 = Pattern.compile(LIMIT_REPR_1);
	private static Pattern p2 = Pattern.compile(LIMIT_REPR_2);

	public static String rewriteForPrestoSql(String querySql, int queryLimit) {
		String result = matchPatter_1(querySql, queryLimit);
		if (result != null)
			return result;
		return querySql + " limit " + queryLimit;
	}

	public static String rewriteForHiveSql(String querySql, int queryLimit) {
		String result = matchPatter_1(querySql, queryLimit);
		if (result != null)
			return result;

		//匹配第二种情况
		result = matchPatter_2(querySql, queryLimit);
		if (result != null)
			return result;

		return querySql + " limit " + queryLimit;
	}

	private static String matchPatter_1(String querySql, int queryLimit) {
		String result = resolveMatcher(querySql, queryLimit, p1.matcher(querySql.toLowerCase()), 1);
		if (result != null)
			return result;
		return null;
	}

	private static String matchPatter_2(String querySql, int queryLimit) {
		String result = resolveMatcher(querySql, queryLimit, p2.matcher(querySql.toLowerCase()), 1);
		if (result != null)
			return result;
		return null;
	}

	private static String resolveMatcher(String querySql, int queryLimit, Matcher m, int group) {
		if (m.matches()) {
			int idx = m.start(group);
			int limit = Integer.parseInt(m.group(group));
			if (limit > queryLimit) {
				return querySql.substring(0, idx) + " " + queryLimit;
			}
			return querySql;
		}
		return null;
	}
}
```

# 验证

```
/**
 * Created by luopeng on 2017/7/26.
 */
public class MyTest {

	public static void main(String[] args) {
		String sql = "select * from (select a.id as userid,a.name as username from a left join b on a.id = b.userid order by a.id limit "
					 + "10000) limit 300";
		//未超出限制，返回原语句
		System.out.println(QuerySqlLimiter.rewriteForPrestoSql(sql,500));
		//超出限制，修改成最大限制
		System.out.println(QuerySqlLimiter.rewriteForPrestoSql(sql,50));

		sql = "select * from (select a.id as userid,a.name as username from a left join b on a.id = b.userid order by a.id limit "
					 + "10000) limit 0,300";
		//未超出限制，返回原语句
		System.out.println(QuerySqlLimiter.rewriteForPrestoSql(sql,500));
		//超出限制，修改成最大限制
		System.out.println(QuerySqlLimiter.rewriteForPrestoSql(sql,50));
	}
}
```

输出：

> select * from (select a.id as userid,a.name as username from a left join b on a.id = b.userid order by a.id limit 10000) limit 300
select * from (select a.id as userid,a.name as username from a left join b on a.id = b.userid order by a.id limit 10000) limit  50
select * from (select a.id as userid,a.name as username from a left join b on a.id = b.userid order by a.id limit 10000) limit 0,300
select * from (select a.id as userid,a.name as username from a left join b on a.id = b.userid order by a.id limit 10000) limit 0, 50


