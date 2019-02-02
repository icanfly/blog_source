---
title: java并发之Striped64解析
tags:
  - java
  - 多线程
  - 并发
categories:
  - 原创文章
originContent: >-
  注：本文基于JDK1.8进行解析，其它JDK版本可能有所不同。


  早在JDK1.5的时候就已经引入了大神Doug
  Lea的并发包体系，其中包括各种显式锁及实现，原子类，原子引用等，极大的丰富了JDK的并发生态。让我们实现数据同步从“原始社会”的synchroinzed阶段一下子过度到了基于CAS的“现代社会”，JDK1.5的AQS堪称当代并发的一个神器级的工具，然而追求永远是无穷尽的，当人们在享受到原子类带来的性能提升的时候，大神Doug
  Lea又一次为原子操作的Long和Double带来新的成员：Striped64及它的子类。它的原理相对来说比较简单，也是JDK常用的方式，就是通过CAS以及“分段技术”努力地减少争用，尽最大可能提高并发度。


  Striped64该类维护了一个惰性初始化的列表和一个基础(base)的数值，列表的大小是2的次方，索引这个列表是通过基于每个线程的内部Probe算出一个Hashcode来确定。这个类的几乎所有的方法都是protected的，所以只有它的子类可以使用。


  ```java

  abstract class Striped64 extends Number {
    @sun.misc.Contended static final class Cell {
          volatile long value;
          Cell(long x) { value = x; }
          final boolean cas(long cmp, long val) {
              return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
          }

          // Unsafe mechanics
          private static final sun.misc.Unsafe UNSAFE;
          private static final long valueOffset;
          static {
              try {
                  UNSAFE = sun.misc.Unsafe.getUnsafe();
                  Class<?> ak = Cell.class;
                  valueOffset = UNSAFE.objectFieldOffset
                      (ak.getDeclaredField("value"));
              } catch (Exception e) {
                  throw new Error(e);
              }
          }
      }

      /** Number of CPUS, to place bound on table size */
      static final int NCPU = Runtime.getRuntime().availableProcessors();

      /**
       * Table of cells. When non-null, size is a power of 2.
       */
      transient volatile Cell[] cells;

      /**
       * Base value, used mainly when there is no contention, but also as
       * a fallback during table initialization races. Updated via CAS.
       */
      transient volatile long base;

      /**
       * Spinlock (locked via CAS) used when resizing and/or creating Cells.
       */
      transient volatile int cellsBusy;

      .... 相关方法省略

      // Unsafe mechanics
      private static final sun.misc.Unsafe UNSAFE;
      private static final long BASE;
      private static final long CELLSBUSY;
      private static final long PROBE;
      static {
          try {
              UNSAFE = sun.misc.Unsafe.getUnsafe();
              Class<?> sk = Striped64.class;
              BASE = UNSAFE.objectFieldOffset
                  (sk.getDeclaredField("base"));
              CELLSBUSY = UNSAFE.objectFieldOffset
                  (sk.getDeclaredField("cellsBusy"));
              Class<?> tk = Thread.class;
              PROBE = UNSAFE.objectFieldOffset
                  (tk.getDeclaredField("threadLocalRandomProbe"));
          } catch (Exception e) {
              throw new Error(e);
          }
      }
  }


  ```

  该类一个包本地类，只能在包范围内引用，包含支持64位值动态分段的类的通用表示和机制。该类同时继承至Number，因此具体的子类必须实现其接口方法。


  在Striped64内部，持有数据的是一个由叫做Cell的数据结构的一个列表实现，这个Cell数据结构通过使用@sun.misc.Contented这个注解来减少缓存行冲突，关于缓存冲突，缓存行，伪共享的描述可以参看相关资料。通常情况下，缓存行填充(Padding)对于大多数原子操作来说都是不必要的，因为它们散落在不规则的内存中。但是对于存在于一个数组内的原子对象来说，这样的情况会发生变化，它们会产生相互影响，原因是因为它们在内存中的布局会相互紧挨着，并存在大量的共享相同的缓存行，而共享缓存行对于性能的影响将是非常巨大的。


  相对来说Cell这个结构还是比较大的，所以我们尽量避免提前创建它们，除非在真正用到它们的时候。当没有竞争时，所有的更新操作都会应用到base字段上。当第一次产生争用时（在base字段上发生CAS失败），这个列表会被初始化，初始化大小为2。当后续仍然产生争用时，这个列表会被进一步扩展（除非到达了它的终极大小限制：列表大小的扩展到和CPU数量相当），列表中的Slot是空的，只有在使用它的时候才进行初始化。


  一个自旋锁cellsBusy被用于列表的初始化和扩容，以及Slot的填充。在这里没有必要使用阻塞，当锁不可用时，线程会尝试获取其它Slot的锁（或者尝试base字段）。在这些重试期间，争用是增加了但局部性争用是降低了，这仍然比替代方案更好。


  通过ThreadLocalRandom维护的Thread
  probe字段用作每线程哈希码。在未产生争用时，我们让它保持未初始化的值为0。当初始化时尽量保证这个值不与其它线程的值相冲突。执行更新操作时，失败的CAS会指示争用或列表冲突。当发生冲突时，如果此时列表的大小还没有达到极限大小限制，列表会进行扩容除非有其它的线程持有这把锁。如果被hashcode指定索引到的slot为空，并且锁是可用的，那么这个slot会被初始化为一个新的Cell。其它情况下，如果slot中存在Cell，那么就执行一次CAS操作来更新Cell中的值。重试通过“双重散列”进行，使用辅助散列（Marsaglia
  XorShift）尝试查找空闲插槽。


  列表大小是有限的，因为当线程多于CPU时，假设每个线程都绑定到CPU，就会存在一个完美的哈希函数，将线程映射到槽以消除冲突。
  当我们达到容量时，我们通过随机改变冲突线程的哈希码来搜索此映射。
  因为搜索是随机的，并且冲突仅通过CAS失败而变得已知，所以收敛可能很慢，并且因为线程通常不会永远地绑定到CPUS，所以可能根本不会发生。
  然而，尽管存在这些限制，但在这些情况下观察到的争用率通常较低。


  当曾经散列到它的线程终止时，以及在列表扩容导致没有线程在扩展掩码下散列到它的情况下，Cell可能会被释放。我们不会尝试检测或删除此类Cell，假设对于长期运行的实例，观察到的争用情况可能会再次出现，因此最终将再次需要Cell;
  对于短命的实例来说，没关系，GC帮我们清理这整个实例。


  在整个实现过程中大量使用CAS无锁操作，并运用Padding技术（缓存行填充）将一个原子化的Long操作性能发挥到极致，在普通无争用或者争用较少的情况下，可以用base以及少量的Cell就可以动态减少争用，并在争用激烈时通过扩容Cell列表的方式来分散争用。这种模式有点类似分段锁的方式，不同的是这种实现更高效，全程无锁无阻塞。


  Striped64类使用一个base和一个分散的Cell列表来实现对于Long型数值的操作，其核心的方法为longAccumulate和doubleAccumulate，其中这两个方法思路和模式均相同，只是一个针对于long类型，一个针对double类型。


  关于对longAccumulate方法的解析如下：


  ```java

  final void longAccumulate(long x, LongBinaryOperator fn,
                                boolean wasUncontended) {
          int h;
          if ((h = getProbe()) == 0) {
              ThreadLocalRandom.current(); // force initialization
              h = getProbe();
              wasUncontended = true;
          }
          boolean collide = false;                // True if last slot nonempty
          for (;;) {
              Cell[] as; Cell a; int n; long v;
              if ((as = cells) != null && (n = as.length) > 0) {
                  if ((a = as[(n - 1) & h]) == null) { 
                      if (cellsBusy == 0) {       
                          Cell r = new Cell(x);   // Optimistically create
                          if (cellsBusy == 0 && casCellsBusy()) {
                              boolean created = false;
                              try {               // Recheck under lock
                                  Cell[] rs; int m, j;
                                  if ((rs = cells) != null &&
                                      (m = rs.length) > 0 &&
                                      rs[j = (m - 1) & h] == null) {
                                      rs[j] = r;
                                      created = true;
                                  }
                              } finally {
                                  cellsBusy = 0;
                              }
                              if (created)
                                  break;
                              continue;           // Slot is now non-empty
                          }
                      }
                      collide = false;
                  }
                  else if (!wasUncontended)       // CAS already known to fail
                      wasUncontended = true;      // Continue after rehash
                  else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                               fn.applyAsLong(v, x))))
                      break;
                  else if (n >= NCPU || cells != as)
                      collide = false;            // At max size or stale
                  else if (!collide)
                      collide = true;
                  else if (cellsBusy == 0 && casCellsBusy()) {
                      try {
                          if (cells == as) {      // Expand table unless stale
                              Cell[] rs = new Cell[n << 1];
                              for (int i = 0; i < n; ++i)
                                  rs[i] = as[i];
                              cells = rs;
                          }
                      } finally {
                          cellsBusy = 0;
                      }
                      collide = false;
                      continue;                   // Retry with expanded table
                  }
                  h = advanceProbe(h);
              }
              else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                  boolean init = false;
                  try {                           // Initialize table
                      if (cells == as) {
                          Cell[] rs = new Cell[2];
                          rs[h & 1] = new Cell(x);
                          cells = rs;
                          init = true;
                      }
                  } finally {
                      cellsBusy = 0;
                  }
                  if (init)
                      break;
              }
              else if (casBase(v = base, ((fn == null) ? v + x :
                                          fn.applyAsLong(v, x))))
                  break;                          // Fall back on using base
          }
      }

  ```


  该类还有几个子类，通常我们在使用的时候一般会使用到的就是它的子类，包括：LongAdder，LongAccumulator，DoubleAdder，DoubleAccumulator，其中LongAdder和LongAccumulator只存在细微差异，Adder故名思意是求和的意思，LongAdder是指多次调用累加求和。而LongAccumulator是累积计算的意思，累积计算就不一定是求和了，也有可能是其它操作，这里它提供了一个二元操作接口了：

  ```java

  @FunctionalInterface

  public interface LongBinaryOperator {

      /**
       * Applies this operator to the given operands.
       *
       * @param left the first operand
       * @param right the second operand
       * @return the operator result
       */
      long applyAsLong(long left, long right);
  }

  ```

  用于控制在这个累积器中应该如何对long类数据进行操作。在Striped64的longAccumulate方法中我们也看到了LongBinaryOperator作为了参数传入，并在更新值时进行了计算，只是默认在传null的情况下，默认为累加，这也是LongAdder实现累加的原理：


  LongAdder类累加方法

  ```java

  /**
       * Adds the given value.
       *
       * @param x the value to add
       */
      public void add(long x) {
          Cell[] as; long b, v; int m; Cell a;
          if ((as = cells) != null || !casBase(b = base, b + x)) {
              boolean uncontended = true;
              if (as == null || (m = as.length - 1) < 0 ||
                  (a = as[getProbe() & m]) == null ||
                  !(uncontended = a.cas(v = a.value, v + x)))
                  longAccumulate(x, null, uncontended);
          }
      }
  ```


  DoubleAdder和DoubleAccumuator同LongAdder和LongAccumulator，这里不再累述。
toc: false
date: 2019-02-02 16:42:44
author:
thumbnail:
blogexcerpt:
---

注：本文基于JDK1.8进行解析，其它JDK版本可能有所不同。

早在JDK1.5的时候就已经引入了大神Doug Lea的并发包体系，其中包括各种显式锁及实现，原子类，原子引用等，极大的丰富了JDK的并发生态。让我们实现数据同步从“原始社会”的synchroinzed阶段一下子过度到了基于CAS的“现代社会”，JDK1.5的AQS堪称当代并发的一个神器级的工具，然而追求永远是无穷尽的，当人们在享受到原子类带来的性能提升的时候，大神Doug Lea又一次为原子操作的Long和Double带来新的成员：Striped64及它的子类。它的原理相对来说比较简单，也是JDK常用的方式，就是通过CAS以及“分段技术”努力地减少争用，尽最大可能提高并发度。

Striped64该类维护了一个惰性初始化的列表和一个基础(base)的数值，列表的大小是2的次方，索引这个列表是通过基于每个线程的内部Probe算出一个Hashcode来确定。这个类的几乎所有的方法都是protected的，所以只有它的子类可以使用。

```java
abstract class Striped64 extends Number {
  @sun.misc.Contended static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> ak = Cell.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (ak.getDeclaredField("value"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }

    /** Number of CPUS, to place bound on table size */
    static final int NCPU = Runtime.getRuntime().availableProcessors();

    /**
     * Table of cells. When non-null, size is a power of 2.
     */
    transient volatile Cell[] cells;

    /**
     * Base value, used mainly when there is no contention, but also as
     * a fallback during table initialization races. Updated via CAS.
     */
    transient volatile long base;

    /**
     * Spinlock (locked via CAS) used when resizing and/or creating Cells.
     */
    transient volatile int cellsBusy;

    .... 相关方法省略

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long BASE;
    private static final long CELLSBUSY;
    private static final long PROBE;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> sk = Striped64.class;
            BASE = UNSAFE.objectFieldOffset
                (sk.getDeclaredField("base"));
            CELLSBUSY = UNSAFE.objectFieldOffset
                (sk.getDeclaredField("cellsBusy"));
            Class<?> tk = Thread.class;
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomProbe"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}

```
该类一个包本地类，只能在包范围内引用，包含支持64位值动态分段的类的通用表示和机制。该类同时继承至Number，因此具体的子类必须实现其接口方法。

在Striped64内部，持有数据的是一个由叫做Cell的数据结构的一个列表实现，这个Cell数据结构通过使用@sun.misc.Contented这个注解来减少缓存行冲突，关于缓存冲突，缓存行，伪共享的描述可以参看相关资料。通常情况下，缓存行填充(Padding)对于大多数原子操作来说都是不必要的，因为它们散落在不规则的内存中。但是对于存在于一个数组内的原子对象来说，这样的情况会发生变化，它们会产生相互影响，原因是因为它们在内存中的布局会相互紧挨着，并存在大量的共享相同的缓存行，而共享缓存行对于性能的影响将是非常巨大的。

相对来说Cell这个结构还是比较大的，所以我们尽量避免提前创建它们，除非在真正用到它们的时候。当没有竞争时，所有的更新操作都会应用到base字段上。当第一次产生争用时（在base字段上发生CAS失败），这个列表会被初始化，初始化大小为2。当后续仍然产生争用时，这个列表会被进一步扩展（除非到达了它的终极大小限制：列表大小的扩展到和CPU数量相当），列表中的Slot是空的，只有在使用它的时候才进行初始化。

一个自旋锁cellsBusy被用于列表的初始化和扩容，以及Slot的填充。在这里没有必要使用阻塞，当锁不可用时，线程会尝试获取其它Slot的锁（或者尝试base字段）。在这些重试期间，争用是增加了但局部性争用是降低了，这仍然比替代方案更好。

通过ThreadLocalRandom维护的Thread probe字段用作每线程哈希码。在未产生争用时，我们让它保持未初始化的值为0。当初始化时尽量保证这个值不与其它线程的值相冲突。执行更新操作时，失败的CAS会指示争用或列表冲突。当发生冲突时，如果此时列表的大小还没有达到极限大小限制，列表会进行扩容除非有其它的线程持有这把锁。如果被hashcode指定索引到的slot为空，并且锁是可用的，那么这个slot会被初始化为一个新的Cell。其它情况下，如果slot中存在Cell，那么就执行一次CAS操作来更新Cell中的值。重试通过“双重散列”进行，使用辅助散列（Marsaglia XorShift）尝试查找空闲插槽。

列表大小是有限的，因为当线程多于CPU时，假设每个线程都绑定到CPU，就会存在一个完美的哈希函数，将线程映射到槽以消除冲突。 当我们达到容量时，我们通过随机改变冲突线程的哈希码来搜索此映射。 因为搜索是随机的，并且冲突仅通过CAS失败而变得已知，所以收敛可能很慢，并且因为线程通常不会永远地绑定到CPUS，所以可能根本不会发生。 然而，尽管存在这些限制，但在这些情况下观察到的争用率通常较低。

当曾经散列到它的线程终止时，以及在列表扩容导致没有线程在扩展掩码下散列到它的情况下，Cell可能会被释放。我们不会尝试检测或删除此类Cell，假设对于长期运行的实例，观察到的争用情况可能会再次出现，因此最终将再次需要Cell; 对于短命的实例来说，没关系，GC帮我们清理这整个实例。

在整个实现过程中大量使用CAS无锁操作，并运用Padding技术（缓存行填充）将一个原子化的Long操作性能发挥到极致，在普通无争用或者争用较少的情况下，可以用base以及少量的Cell就可以动态减少争用，并在争用激烈时通过扩容Cell列表的方式来分散争用。这种模式有点类似分段锁的方式，不同的是这种实现更高效，全程无锁无阻塞。

Striped64类使用一个base和一个分散的Cell列表来实现对于Long型数值的操作，其核心的方法为longAccumulate和doubleAccumulate，其中这两个方法思路和模式均相同，只是一个针对于long类型，一个针对double类型。

关于对longAccumulate方法的解析如下：

```java
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            if ((as = cells) != null && (n = as.length) > 0) {
                if ((a = as[(n - 1) & h]) == null) { 
                    if (cellsBusy == 0) {       
                        Cell r = new Cell(x);   // Optimistically create
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = advanceProbe(h);
            }
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }

```

该类还有几个子类，通常我们在使用的时候一般会使用到的就是它的子类，包括：LongAdder，LongAccumulator，DoubleAdder，DoubleAccumulator，其中LongAdder和LongAccumulator只存在细微差异，Adder故名思意是求和的意思，LongAdder是指多次调用累加求和。而LongAccumulator是累积计算的意思，累积计算就不一定是求和了，也有可能是其它操作，这里它提供了一个二元操作接口了：
```java
@FunctionalInterface
public interface LongBinaryOperator {

    /**
     * Applies this operator to the given operands.
     *
     * @param left the first operand
     * @param right the second operand
     * @return the operator result
     */
    long applyAsLong(long left, long right);
}
```
用于控制在这个累积器中应该如何对long类数据进行操作。在Striped64的longAccumulate方法中我们也看到了LongBinaryOperator作为了参数传入，并在更新值时进行了计算，只是默认在传null的情况下，默认为累加，这也是LongAdder实现累加的原理：

LongAdder类累加方法
```java
/**
     * Adds the given value.
     *
     * @param x the value to add
     */
    public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }
```

DoubleAdder和DoubleAccumuator同LongAdder和LongAccumulator，这里不再累述。