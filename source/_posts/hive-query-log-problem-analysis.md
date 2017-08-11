title: Hive获取查询日志的问题解析
date: 2017-05-05

tags:
 - HIVE
 - 大数据
 - bug
categories:
 - 大数据

---

# 需求背景

最近这段时间一直在做数据查询系统的需求，最近接到一个需求：因为HIVE查询一般需要比较久的查询时间，这期间查询人员需要知道查询的进度，需要在界面上进行进度的展示。

# 探路过程

我们查询系统连接HIVE使用的是标准的JDBC接口，在标准的JDBC接口中并没有提供这样的一个获取查询日志的接口。翻阅了很多的资料后发现其实在HIVE Server的Thrift接口中是有提供这样的接口的：

```java
public List<String> getQueryLog(boolean incremental, int fetchSize)
      throws SQLException, ClosedOrCancelledStatementException {
    checkConnection("getQueryLog");
    if (isCancelled) {
      throw new ClosedOrCancelledStatementException("Method getQueryLog() failed. The " +
          "statement has been closed or cancelled.");
    }

    List<String> logs = new ArrayList<String>();
    TFetchResultsResp tFetchResultsResp = null;
    try {
      if (stmtHandle != null) {
        TFetchResultsReq tFetchResultsReq = new TFetchResultsReq(stmtHandle,
            getFetchOrientation(incremental), fetchSize);
        tFetchResultsReq.setFetchType((short)1);
        tFetchResultsResp = client.FetchResults(tFetchResultsReq);
        Utils.verifySuccessWithInfo(tFetchResultsResp.getStatus());
      } else {
        if (isQueryClosed) {
          throw new ClosedOrCancelledStatementException("Method getQueryLog() failed. The " +
              "statement has been closed or cancelled.");
        }
        if (isExecuteStatementFailed) {
          throw new SQLException("Method getQueryLog() failed. Because the stmtHandle in " +
              "HiveStatement is null and the statement execution might fail.");
        } else {
          return logs;
        }
      }
    } catch (SQLException e) {
      throw e;
    } catch (Exception e) {
      throw new SQLException("Error when getting query log: " + e, e);
    }

    RowSet rowSet = RowSetFactory.create(tFetchResultsResp.getResults(),
        connection.getProtocol());
    for (Object[] row : rowSet) {
      logs.add(String.valueOf(row[0]));
    }
    return logs;
  }
```
以上取至HIVE的JDBC接口实现`HiveStatement`这个类。这个类是标准`java.sql.Statement`的实现，但是`getQueryLog`这个方法并不是标准的JDBC方法，因为在我们的程序中运行的就是HIVE查询，所以我们可以在程序中进行强转得到HiveStatement这个类并调用这个方法获取到查询日志。 getQueryLog 这个方法中用到了整个HiveStatement中的一些变量，所以我们要进行HIVE查询日志的获取必须要对HiveStatement对象进行关联，同时一边在执行HIVE查询，一边还要从另一个线程中获取HIVE查询的日志过程。

<!--more-->

# 设计思路

- 前端查询页面在进行查询提交时同时生成一个UUID类似的唯一查询ID一并提交到查询后台

- 程序接到HIVE查询请求后，将HIVE查询请求通过标准的JDBC的方式进行提交，需要注意的这期间通过HiveConnection获取到的Statement对象需要被缓存到自己创建的一个HiveStatementHolderService类里并和第1步生成的唯一ID关联，以便于上面提到的日志线程池进行日志查询

- 为了不影响HIVE查询线程，HIVE的执行日志查询放到另一个线程(池)中进行

- 通过定时调度轮询，日志查询线程通过HiveStatement的`getQueryLogs`查询到日志后将日志写入集中缓存如redis有序集合中，key为查询ID，同时为了redis内存回收可以设置一个过期时间

- 查询页面在提交HIVE查询后，通过定时轮询的方式，携带查询提交时的查询ID轮询HiveStatementHolderService服务，HiveStatementHolderService服务根据查询ID到对应的redis中取得对应的日志序列集合，并返回给查询展示端

- 将HIVE查询结束后，将Statement从HiveStatementHolderService中移除掉

# 暴露的问题

一切都感觉很美好，但是现实呢？当我深入到HiveStatement内部，我发现了问题：<b>HiveStatement并不是一个线程安全的类！</b>也就是说这个类的实例在多线程环境下使用并不安全，可能会造成多线程访问出现数据上的问题或者报错，具体原因就是该类的各个方法，以及各个判断中都使用了类的局部属性，而这些属性的获取和设值并没有经过线程同步，所以可能会存在线程不同步的一些问题。基于这个问题我也google了一下，发现网上也有相关的issue：

[HIVE-16451](https://issues.apache.org/jira/browse/HIVE-16451)
> Thanks for finding this out Peter Vary. Although I didn't quite get how the patch fixes the race condition. The way I understand the issue is that there is a Logging thread and the thread executing the HiveStatement. Both these threads are accessing isLogBeingGenerated, isCancelled, isQueryClosed flags in the same HiveStatement object. None of these getters and setters are thread safe. I think there could be more undiscovered race-conditions in this execution path.

提到了HiveStatement的线程安全问题

[HIVE-16517](https://issues.apache.org/jira/browse/HIVE-16517)
> BeeLine, and Commands classes shares one HiveStatement between multiple threads for querying the logs, and running the queries.
We can not make the HiveStatement thread safe, but we should at least make sure that calling getQueryLog will not cause problems if it is called parallel with any of the followings: cancel, close, execute, executeAsync, executeQuery, executeUpdate, getUpdateCount and more interestingly for the HiveQueryResultSet.next too.

上面更是提到了queryLog的获取存在线程不同步的问题

[HIVE-15940](https://issues.apache.org/jira/browse/HIVE-15940)
最后这个ISSUE提出了后续可能的一个解决方案：Merge the query log operation as part of the getOperationStatus which also gets the Progress update.
将查询日志作为`getOperationStatus`调用的一部分，具体的怎么设计估计还得等官方的具体实现了。

# 总结

目前看来，此方案并没有完美，HiveStatement存在线程安全问题，不过我们应该可以暂时忍受一些线程不同步带来的很多问题，毕竟只是一个日志显示的问题，哪怕出错，报出什么异常，我们也可以暂时用粗暴的方式来解决：一旦出现异常就直接把该Statement的日志获取给停掉。


