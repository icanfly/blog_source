title: SQL小记-改写SQL中Limit限制
date: 2017-08-01
tags:
 - SQL
 - 参与开源
categories:
 - 原创文章
toc: true
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

> 最新更新：上述的正则表达式不能匹配带有格式（有空白符号和回车换行符的格式化语句）
需要进行一些修改：

```
//Limit关键字匹配: limit 10
public static final String LIMIT_REPR_1 = "^(.|\\s)+\\s+limit\\s+(\\d+)$";

//Limit关键字匹配: limit 10,100
public static final String LIMIT_REPR_2 = "^(.|\\s)+\\s+limit\\s+\\d+\\s*,\\s*(\\d+)$";
```
这组的正则表达式性能是极低的，复杂的sql基本上匹配会卡住非常非常长的时间

继续优化：
```
//Limit关键字匹配: limit 10
public static final String LIMIT_REPR_1 = "^(.|[ \\f\\n\\r\\t\\v])+\\s+limit\\s+(\\d+)$";

//Limit关键字匹配: limit 10,100
public static final String LIMIT_REPR_2 = "^(.|[ \\f\\n\\r\\t\\v])+\\s+limit\\s+\\d+\\s*,\\s*(\\d+)$";
```
实战中我也发现，正如上文中正则表达式优化所说的：`\s匹配任何空白字符，包括空格、制表符、换页符等等。等价于 [ \f\n\r\t\v]。`，但是如果我们用\s来替换[ \f\n\r\t\v]，得到的表达式性能极差，但是如果我们换用确定性的[ \f\n\r\t\v]来替换，则性能则变得很好了，这也许就是Java正则表达式引擎做的优化手段吧？


# 实现

以下代码仅作参考，实则在某些sql条件下会有性能问题
如果有更好的正则表达式的写法，欢迎联系我

```java
/**
 * SQL查询Limit改写器
 * Created by luopeng on 2017/8/1.
 * 因正则写得不太好，匹配时可能会出现性能问题，导致CPU 100%
 */
@Deprecated
public class QuerySqlLimiter {

	//Limit关键字匹配: limit 10
    public static final String LIMIT_REPR_1 = "^(.|[ \\f\\n\\r\\t\\v])+[ \f\n\r\t\v]+limit[ \f\n\r\t\v]+(\\d+)$";

    //Limit关键字匹配: limit 10,100
    public static final String LIMIT_REPR_2 = "^(.|[ \\f\\n\\r\\t\\v])+[ \f\n\r\t\v]+limit[ \f\n\r\t\v]+\\d+[ \f\n\r\t\v]*,[ \f\n\r\t\v]*(\\d+)$";

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
		String result = resolveMatcher(querySql, queryLimit, p1.matcher(querySql.toLowerCase()), 2);
		if (result != null)
			return result;
		return null;
	}

	private static String matchPatter_2(String querySql, int queryLimit) {
		String result = resolveMatcher(querySql, queryLimit, p2.matcher(querySql.toLowerCase()), 2);
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

# 2017-08-02更新

 - 在上线过后的小段时间内线上服务器出现了栈溢出`java.lang.StackOverflowError`的情况。

> 注意StackOverflowError（A note about the StackOverflowError）

  有时regex包中的Pattern类会抛出StackOverflowError。这是已知的bug #5050507的表现，它自从Java 1.4就存在于java.util.regex包中。这个bug仍然存在，因为它是“won't fix”的状态。这个错误的出现是因为，Pattern类把一个正则表达式编译为一个用来寻找匹配的小程序。这个程序被递归调用，有时太多的递归就会导致该错误的出现。更多细节请参考bug描述。看起来大部分是在使用选择（alternation）的出现。
  如果你碰到了这个错误，尝试重写正则表达式，或者分为几个子表达式，然后分别单独执行。后者有时设置会提高性能。

 - 同时CPU开始飙高到了90%，NFA的回缩确实被验证了。以下SQL是导致SQL飙高的其中一条：

 `
 select t1.userid,t1.keyword,c.cityname,b.usermobile from( select t.userid,substring(visit_time,1,10) as visit_timeed,current_path, split_part(split_part(url_extract_query(current_url),'kw=',2),'&',1) as keyword from hive.bdc_dwd.dw_fact_galog_pv_daily t where acct_day >= '2017-01-01' and acct_day <= '2017-08-01' and ((t.current_domainname='search.xxx.com' and current_path like '%kw=%')or(t.current_domainname='list.xxx.com' and current_path like '%key=%')) )t1 left join (select user_id,usermobile from hive.bdc_dwd.dw_mb_account_p group by user_id,usermobile)b on t1.userid=b.user_id left join (select user_id,cityname from hive.bdc_dwd.dw_mb_info group by user_id,cityname)c on c.user_id=t1.userid where t1.keyword in('app开发','微信开发','软件开发','UI设计','手机游戏开发','商城开发','办公系统开发','餐饮系统','教育','直播','支付系统','酒店系统开发','电商系统') group by t1.userid,t1.keyword,c.cityname,b.usermobile
 `

 出于对正则表达式的不精通，这样处理下去可能会有其它的问题出现，所以决定换一种解决思路：

 - 对语句进行整形，去除一些分析干扰，比如将语句进行trim去掉两端空白，去掉语句最后的';'分号等
 - 检测sql中是否有limit关键字，这个可以使用Java字符串的lastIndexOf，这种确定性搜索还是相当高效的
 - 当没有检索到limit关键字时，就可以直接在语句后添加Limit并加上限制数量了
 - 当有检索到limit关键字后，需要判断limit后是否是形如limit xxx 或limit xxx,xxx的形式，这里需要注意的是这期间可能会有空格，回车换行符，制表符等空白字符。这个时候可以使用正则表达式，这个时候正则的匹配范围就变得很小很小了，效率很高
 - 当上一步匹配到limit的形式后，就可以用以前的办法，对limit进行比较和替换了

 # 最新实现

 ```java
 /**
 * SQL查询Limit改写器
 * Created by luopeng on 2017/8/2.
 */
public class QuerySqlLimiter {


	public static final List<Character> SPLIT_CHARS = Arrays.asList(' ', '1', '2', '3', '4', '5', '6', '7', '8', '9', '0',',', '\n', '\r',
			'\t');

	public static final String LIMIT_NUM_REPR_1 = "^[ \\f\\n\\r\\t\\v]+(\\d+)$";
	public static final String LIMIT_NUM_REPR_2 = "^[ \\f\\n\\r\\t\\v]+(\\d+)[ \\f\\n\\r\\t\\v]*,[ \\f\\n\\r\\t\\v]*(\\d+)$";

	private static Pattern p3 = Pattern.compile(LIMIT_NUM_REPR_1);
	private static Pattern p4 = Pattern.compile(LIMIT_NUM_REPR_2);
	private static String keyword = "limit";

	public static String rewriteSql(String querySql, int queryLimit) {

		String originSql = querySql;
		querySql = trimDelimiter(querySql).toLowerCase();
		int length = querySql.length();

		int limitIdx = StringUtils.lastIndexOf(querySql, keyword);

		if (limitIdx == -1) {
			//直接添加limit
			return originSql + " LIMIT " + queryLimit;
		}

		int limitIdxEnd = limitIdx + keyword.length();

		for (int i = limitIdxEnd; i < length - 1; ++i) {
			char c = querySql.charAt(i);
			if (!SPLIT_CHARS.contains(c)) {
				return originSql + " LIMIT " + queryLimit;
			}
		}
		//当以上的遍历完成时，证明存在LIMIT结尾，通过substring进一步分析limit数量
		String limitStr = StringUtils.substring(querySql, limitIdxEnd);
		Matcher matcher = p3.matcher(limitStr);
		if (matcher.matches()) {
			int limit = Integer.parseInt(matcher.group(1));
			if (limit > queryLimit) {
				return StringUtils.substring(originSql, 0,limitIdx) + " LIMIT " + queryLimit;
			} else {
				return originSql;
			}
		}
		matcher = p4.matcher(limitStr);
		if (matcher.matches()) {
			int offset = Integer.parseInt(matcher.group(1));
			int limit = Integer.parseInt(matcher.group(2));
			if (limit > queryLimit) {
				return StringUtils.substring(originSql, 0, limitIdx) + " LIMIT " + offset + "," + queryLimit;
			} else {
				return originSql;
			}
		}

		//直接添加limit
		return originSql + " LIMIT " + queryLimit;
	}

	private static String trimDelimiter(String querySql) {
		querySql = StringUtils.trim(querySql);
		if (StringUtils.endsWith(querySql, ";")) {
			querySql = StringUtils.substring(querySql, 0, querySql.length() - 1);
		}
		return querySql;
	}
}
 ```

 # 总结

 以前一直不觉得正则表达式会有什么问题，实际运用中也没有遇到和去理解正则表达式的原理。才导致出现了上面所说的问题，有时候采用比正则表达式更原始更粗暴的方式解决问题可能反而会更加高效，这也许也是自己不精通正则表达式编写的问题，后续需要多多关注。
