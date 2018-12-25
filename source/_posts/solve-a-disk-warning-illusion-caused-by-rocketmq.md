---
title: solve-a-disk-warning-illusion-caused-by-rocketmq
tags:
  - rocketmq
categories:
  - 原创文章
originContent: >-
  最近一段时间在运维部署rocketmq的过程中，启动时频繁报一个奇怪的错：


  ```

  2018-12-19 22:20:58 INFO StoreScheduledThread1 - begin to delete before 336
  hours file. timeup: false spacefull: true manualDeleteFileSeveralTimes: 0
  cleanAtOnce: false

  2018-12-19 22:20:58 WARN StoreScheduledThread1 - disk space will be full soon,
  but delete file failed.

  2018-12-19 22:21:08 INFO StoreScheduledThread1 - physic disk maybe full soon,
  so reclaim space, -1.0

  2018-12-19 22:21:08 INFO StoreScheduledThread1 - begin to delete before 336
  hours file. timeup: false spacefull: true manualDeleteFileSeveralTimes: 0
  cleanAtOnce: false

  2018-12-19 22:21:08 WARN StoreScheduledThread1 - disk space will be full soon,
  but delete file failed.

  2018-12-19 22:21:15 INFO StoreStatsService - [STORETPS] put_tps 0.0
  get_found_tps 0.0 get_miss_tps 1.799730040493926 get_transfered_tps 0.0

  2018-12-19 22:21:15 INFO StoreStatsService - [PAGECACHERT] TotalPut 0,
  PutMessageDistributeTime [<=0ms]:0 [0~10ms]:0 [10~50ms]:0 [50~100ms]:0
  [100~200ms]:0 [200~500ms]:0 [500ms~1s]:0 [1~2s]:0 [2~3s]:0 [3~4s]:0 [4~5s]:0
  [5~10s]:0 [10s~]:0 

  2018-12-19 22:21:18 INFO StoreScheduledThread1 - physic disk maybe full soon,
  so reclaim space, -1.0

  2018-12-19 22:21:18 INFO StoreScheduledThread1 - begin to delete before 336
  hours file. timeup: false spacefull: true manualDeleteFileSeveralTimes: 0
  cleanAtOnce: false

  2018-12-19 22:21:18 WARN StoreScheduledThread1 - disk space will be full soon,
  but delete file failed.

  2018-12-19 22:21:28 INFO StoreScheduledThread1 - physic disk maybe full soon,
  so reclaim space, -1.0

  2018-12-19 22:21:28 INFO StoreScheduledThread1 - begin to delete before 336
  hours file. timeup: false spacefull: true manualDeleteFileSeveralTimes: 0
  cleanAtOnce: false

  2018-12-19 22:21:28 WARN StoreScheduledThread1 - disk space will be full soon,
  but delete file failed.

  2018-12-19 22:21:38 INFO StoreScheduledThread1 - physic disk maybe full soon,
  so reclaim space, -1.0

  2018-12-19 22:21:38 INFO StoreScheduledThread1 - begin to delete before 336
  hours file. timeup: false spacefull: true manualDeleteFileSeveralTimes: 0
  cleanAtOnce: false

  2018-12-19 22:21:38 WARN StoreScheduledThread1 - disk space will be full soon,
  but delete file failed.

  2018-12-19 22:21:48 INFO StoreScheduledThread1 - physic disk maybe full soon,
  so reclaim space, -1.0

  2018-12-19 22:21:48 INFO StoreScheduledThread1 - begin to delete before 336
  hours file. timeup: false spacefull: true manualDeleteFileSeveralTimes: 0
  cleanAtOnce: false

  2018-12-19 22:21:48 WARN StoreScheduledThread1 - disk space will be full soon,
  but delete file failed.

  2018-12-19 22:21:58 INFO StoreScheduledThread1 - physic disk maybe full soon,
  so reclaim space, -1.0


  ```


  而当时查看了磁盘的容量，远远没有达到rocketmq的磁盘容量警告阀值。剩余的磁盘空间还非常多，一开始是怀疑运维人员没有将rocketmq的存储目录挂载到数据盘，但是经过沟通后发现已经挂载了。


  最后没办法只能是通过阅读rocketmq源代码找原因：


  ```java

  private boolean isSpaceToDelete() {
              double ratio = DefaultMessageStore.this.getMessageStoreConfig().getDiskMaxUsedSpaceRatio() / 100.0;

              cleanImmediately = false;

              {
                  String storePathPhysic = DefaultMessageStore.this.getMessageStoreConfig().getStorePathCommitLog();
                  double physicRatio = UtilAll.getDiskPartitionSpaceUsedPercent(storePathPhysic);
                  if (physicRatio > diskSpaceWarningLevelRatio) {
                      boolean diskok = DefaultMessageStore.this.runningFlags.getAndMakeDiskFull();
                      if (diskok) {
                          DefaultMessageStore.log.error("physic disk maybe full soon " + physicRatio + ", so mark disk full");
                      }

                      cleanImmediately = true;
                  } else if (physicRatio > diskSpaceCleanForciblyRatio) {
                      cleanImmediately = true;
                  } else {
                      boolean diskok = DefaultMessageStore.this.runningFlags.getAndMakeDiskOK();
                      if (!diskok) {
                          DefaultMessageStore.log.info("physic disk space OK " + physicRatio + ", so mark disk ok");
                      }
                  }

                  if (physicRatio < 0 || physicRatio > ratio) {
                      DefaultMessageStore.log.info("physic disk maybe full soon, so reclaim space, " + physicRatio);
                      return true;
                  }
              }

              {
                  String storePathLogics = StorePathConfigHelper
                      .getStorePathConsumeQueue(DefaultMessageStore.this.getMessageStoreConfig().getStorePathRootDir());
                  double logicsRatio = UtilAll.getDiskPartitionSpaceUsedPercent(storePathLogics);
                  if (logicsRatio > diskSpaceWarningLevelRatio) {
                      boolean diskok = DefaultMessageStore.this.runningFlags.getAndMakeDiskFull();
                      if (diskok) {
                          DefaultMessageStore.log.error("logics disk maybe full soon " + logicsRatio + ", so mark disk full");
                      }

                      cleanImmediately = true;
                  } else if (logicsRatio > diskSpaceCleanForciblyRatio) {
                      cleanImmediately = true;
                  } else {
                      boolean diskok = DefaultMessageStore.this.runningFlags.getAndMakeDiskOK();
                      if (!diskok) {
                          DefaultMessageStore.log.info("logics disk space OK " + logicsRatio + ", so mark disk ok");
                      }
                  }

                  if (logicsRatio < 0 || logicsRatio > ratio) {
                      DefaultMessageStore.log.info("logics disk maybe full soon, so reclaim space, " + logicsRatio);
                      return true;
                  }
              }

              return false;
          }
  ```


  关键错误出现在：


  ```java

  double physicRatio =
  UtilAll.getDiskPartitionSpaceUsedPercent(storePathPhysic);

  ```

  这里出现返回-1的情况，仔细捋了一把这个工具类的源码：


  ```java

  public static double getDiskPartitionSpaceUsedPercent(final String path) {
          if (null == path || path.isEmpty())
              return -1;

          try {
              File file = new File(path);

              if (!file.exists())
                  return -1;

              long totalSpace = file.getTotalSpace();

              if (totalSpace > 0) {
                  long freeSpace = file.getFreeSpace();
                  long usedSpace = totalSpace - freeSpace;

                  return usedSpace / (double) totalSpace;
              }
          } catch (Exception e) {
              return -1;
          }

          return -1;
      }

  ```


  综合以上的现象发现只能是发生了异常，而在异常这里，rocketmq自己吃掉了异常，并返回了-1。


  而为什么在计算磁盘空间的时候会出现异常呢，目前能想到的一个原因可能是因为安全原因，导致问题出现，而在linux下selinux是产生文件方面安全问题的重要原因。


  解决方案：叫运维关闭selinux后，情况恢复正常。
toc: false
author:
thumbnail:
blogexcerpt:
---

最近一段时间在运维部署rocketmq的过程中，启动时频繁报一个奇怪的错：

```
2018-12-19 22:20:58 INFO StoreScheduledThread1 - begin to delete before 336 hours file. timeup: false spacefull: true manualDeleteFileSeveralTimes: 0 cleanAtOnce: false
2018-12-19 22:20:58 WARN StoreScheduledThread1 - disk space will be full soon, but delete file failed.
2018-12-19 22:21:08 INFO StoreScheduledThread1 - physic disk maybe full soon, so reclaim space, -1.0
2018-12-19 22:21:08 INFO StoreScheduledThread1 - begin to delete before 336 hours file. timeup: false spacefull: true manualDeleteFileSeveralTimes: 0 cleanAtOnce: false
2018-12-19 22:21:08 WARN StoreScheduledThread1 - disk space will be full soon, but delete file failed.
2018-12-19 22:21:15 INFO StoreStatsService - [STORETPS] put_tps 0.0 get_found_tps 0.0 get_miss_tps 1.799730040493926 get_transfered_tps 0.0
2018-12-19 22:21:15 INFO StoreStatsService - [PAGECACHERT] TotalPut 0, PutMessageDistributeTime [<=0ms]:0 [0~10ms]:0 [10~50ms]:0 [50~100ms]:0 [100~200ms]:0 [200~500ms]:0 [500ms~1s]:0 [1~2s]:0 [2~3s]:0 [3~4s]:0 [4~5s]:0 [5~10s]:0 [10s~]:0 
2018-12-19 22:21:18 INFO StoreScheduledThread1 - physic disk maybe full soon, so reclaim space, -1.0
2018-12-19 22:21:18 INFO StoreScheduledThread1 - begin to delete before 336 hours file. timeup: false spacefull: true manualDeleteFileSeveralTimes: 0 cleanAtOnce: false
2018-12-19 22:21:18 WARN StoreScheduledThread1 - disk space will be full soon, but delete file failed.
2018-12-19 22:21:28 INFO StoreScheduledThread1 - physic disk maybe full soon, so reclaim space, -1.0
2018-12-19 22:21:28 INFO StoreScheduledThread1 - begin to delete before 336 hours file. timeup: false spacefull: true manualDeleteFileSeveralTimes: 0 cleanAtOnce: false
2018-12-19 22:21:28 WARN StoreScheduledThread1 - disk space will be full soon, but delete file failed.
2018-12-19 22:21:38 INFO StoreScheduledThread1 - physic disk maybe full soon, so reclaim space, -1.0
2018-12-19 22:21:38 INFO StoreScheduledThread1 - begin to delete before 336 hours file. timeup: false spacefull: true manualDeleteFileSeveralTimes: 0 cleanAtOnce: false
2018-12-19 22:21:38 WARN StoreScheduledThread1 - disk space will be full soon, but delete file failed.
2018-12-19 22:21:48 INFO StoreScheduledThread1 - physic disk maybe full soon, so reclaim space, -1.0
2018-12-19 22:21:48 INFO StoreScheduledThread1 - begin to delete before 336 hours file. timeup: false spacefull: true manualDeleteFileSeveralTimes: 0 cleanAtOnce: false
2018-12-19 22:21:48 WARN StoreScheduledThread1 - disk space will be full soon, but delete file failed.
2018-12-19 22:21:58 INFO StoreScheduledThread1 - physic disk maybe full soon, so reclaim space, -1.0

```

而当时查看了磁盘的容量，远远没有达到rocketmq的磁盘容量警告阀值。剩余的磁盘空间还非常多，一开始是怀疑运维人员没有将rocketmq的存储目录挂载到数据盘，但是经过沟通后发现已经挂载了。

最后没办法只能是通过阅读rocketmq源代码找原因：

```java
private boolean isSpaceToDelete() {
            double ratio = DefaultMessageStore.this.getMessageStoreConfig().getDiskMaxUsedSpaceRatio() / 100.0;

            cleanImmediately = false;

            {
                String storePathPhysic = DefaultMessageStore.this.getMessageStoreConfig().getStorePathCommitLog();
                double physicRatio = UtilAll.getDiskPartitionSpaceUsedPercent(storePathPhysic);
                if (physicRatio > diskSpaceWarningLevelRatio) {
                    boolean diskok = DefaultMessageStore.this.runningFlags.getAndMakeDiskFull();
                    if (diskok) {
                        DefaultMessageStore.log.error("physic disk maybe full soon " + physicRatio + ", so mark disk full");
                    }

                    cleanImmediately = true;
                } else if (physicRatio > diskSpaceCleanForciblyRatio) {
                    cleanImmediately = true;
                } else {
                    boolean diskok = DefaultMessageStore.this.runningFlags.getAndMakeDiskOK();
                    if (!diskok) {
                        DefaultMessageStore.log.info("physic disk space OK " + physicRatio + ", so mark disk ok");
                    }
                }

                if (physicRatio < 0 || physicRatio > ratio) {
                    DefaultMessageStore.log.info("physic disk maybe full soon, so reclaim space, " + physicRatio);
                    return true;
                }
            }

            {
                String storePathLogics = StorePathConfigHelper
                    .getStorePathConsumeQueue(DefaultMessageStore.this.getMessageStoreConfig().getStorePathRootDir());
                double logicsRatio = UtilAll.getDiskPartitionSpaceUsedPercent(storePathLogics);
                if (logicsRatio > diskSpaceWarningLevelRatio) {
                    boolean diskok = DefaultMessageStore.this.runningFlags.getAndMakeDiskFull();
                    if (diskok) {
                        DefaultMessageStore.log.error("logics disk maybe full soon " + logicsRatio + ", so mark disk full");
                    }

                    cleanImmediately = true;
                } else if (logicsRatio > diskSpaceCleanForciblyRatio) {
                    cleanImmediately = true;
                } else {
                    boolean diskok = DefaultMessageStore.this.runningFlags.getAndMakeDiskOK();
                    if (!diskok) {
                        DefaultMessageStore.log.info("logics disk space OK " + logicsRatio + ", so mark disk ok");
                    }
                }

                if (logicsRatio < 0 || logicsRatio > ratio) {
                    DefaultMessageStore.log.info("logics disk maybe full soon, so reclaim space, " + logicsRatio);
                    return true;
                }
            }

            return false;
        }
```

关键错误出现在：

```java
double physicRatio = UtilAll.getDiskPartitionSpaceUsedPercent(storePathPhysic);
```
这里出现返回-1的情况，仔细捋了一把这个工具类的源码：

```java
public static double getDiskPartitionSpaceUsedPercent(final String path) {
        if (null == path || path.isEmpty())
            return -1;

        try {
            File file = new File(path);

            if (!file.exists())
                return -1;

            long totalSpace = file.getTotalSpace();

            if (totalSpace > 0) {
                long freeSpace = file.getFreeSpace();
                long usedSpace = totalSpace - freeSpace;

                return usedSpace / (double) totalSpace;
            }
        } catch (Exception e) {
            return -1;
        }

        return -1;
    }

```

综合以上的现象发现只能是发生了异常，而在异常这里，rocketmq自己吃掉了异常，并返回了-1。

这里个人感觉rocketmq团队在这里处理的方式非常不友好，不仅吃掉了异常而且还返回了一个没意义的值！

而为什么在计算磁盘空间的时候会出现异常呢，目前能想到的一个原因可能是因为安全原因，导致问题出现，而在linux下selinux是产生文件方面安全问题的重要原因。

解决方案：叫运维关闭selinux后，情况恢复正常。