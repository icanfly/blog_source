---
title: 合并写(write combining)
tags:
  - java
  - 多线程
  - 并发
categories:
  - 转载文章
originContent: >-
  **转载自并发编程网 – ifeve.com 本文链接地址: [合并写(write
  combining)](http://ifeve.com/writecombining/) 译者：无叶 校对：丁一**


  现代CPU采用了大量的技术来抵消内存访问带来的延迟。读写内存数据期间，CPU能执行成百上千条指令。


  多级SRAM缓存是减小这种延迟带来的影响的主要手段。此外，SMP系统采用消息传递协议来实现缓存之间的一致性。遗憾的是，现代的CPU实在是太快了，即使是使用了缓存，有时也无法跟上CPU的速度。因此，为了进一步减小延迟的影响，一些鲜为人知的缓冲区派上了用场。


  本文将探讨“合并写存储缓冲区（write combining store buffers）”，以及如何写出有效利用它们的代码。


  CPU缓存是一种高效的非链式结构的hash map，每个桶（bucket）通常是64个字节。这就是一个“缓存行（cache
  line）”。缓存行是内存交换的实际单位。例如，主存中地址A会映射到一个给定的缓存行C。



  如果CPU需要访问的地址hash后的行尚不在缓存中，那么缓存中对应位置的缓存行会被清除，以便载入新的行。例如，如果我们有两个地址，通过hash算法hash到同一缓存行，那么新的值会覆盖老的值。


  当CPU执行存储指令（store）时，它会尝试将数据写到离CPU最近的L1缓存。如果此时出现缓存未命中，CPU会访问下一级缓存。此时，无论是英特尔还是许多其它厂商的CPU都会使用一种称为“合并写（write
  combining）”的技术。


  在请求L2缓存行的所有权尚未完成时，待存储的数据被写到处理器自身的众多跟缓存行一样大小的存储缓冲区之一。这些芯片上的缓冲区允许CPU在缓存子系统准备好接收和处理数据时继续执行指令。当数据不在任何其它级别的缓存中时，将获得最大的优势。


  当后续的写操作需要修改相同的缓存行时，这些缓冲区变得非常有趣。在将后续的写操作提交到L2缓存之前，可以进行缓冲区写合并。
  这些64字节的缓冲区维护了一个64位的字段，每更新一个字节就会设置对应的位，来表示将缓冲区交换到外部缓存时哪些数据是有效的。


  也许你要问，如果程序要读取已被写入缓冲区的某些数据，会怎么样？我们的硬件工程师已经考虑到了这点，在读取缓存之前会先去读取缓冲区的。


  这一切对我们的程序意味着什么？


  如果我们能在缓冲区被传输到外部缓存之前将其填满，那么将大大提高各级传输总线的效率。如何才能做到这一点呢？好的程序将大部分时间花在循环处理任务上。


  这些缓冲区的数量是有限的，且随CPU模型而异。例如在Intel
  CPU中，同一时刻只能拿到4个。这意味着，在一个循环中，你不应该同时写超过4个不同的内存位置，否则你将不能享受到合并写（write
  combining）的好处。


  ```java

  public final class WriteCombining {

      private static final int    ITERATIONS = Integer.MAX_VALUE;
      private static final int    ITEMS      = 1 << 24;
      private static final int    MASK       = ITEMS - 1;

      private static final byte[] arrayA     = new byte[ITEMS];
      private static final byte[] arrayB     = new byte[ITEMS];
      private static final byte[] arrayC     = new byte[ITEMS];
      private static final byte[] arrayD     = new byte[ITEMS];
      private static final byte[] arrayE     = new byte[ITEMS];
      private static final byte[] arrayF     = new byte[ITEMS];

      public static void main(final String[] args) {
          for (int i = 1; i <= 3; i++) {
              out.println(i + " SingleLoop duration (ns) = " + runCaseOne());
              out.println(i + " SplitLoop duration (ns) = " + runCaseTwo());
          }
          int result = arrayA[1] + arrayB[2] + arrayC[3] + arrayD[4] + arrayE[5] + arrayF[6];
          out.println("result = " + result);
      }

      public static long runCaseOne() {
          long start = System.nanoTime();
          int i = ITERATIONS;

          while (--i != 0) {
              int slot = i & MASK;
              byte b = (byte) i;
              arrayA[slot] = b;
              arrayB[slot] = b;
              arrayC[slot] = b;
              arrayD[slot] = b;
              arrayE[slot] = b;
              arrayF[slot] = b;
          }
          return System.nanoTime() - start;
      }

      public static long runCaseTwo() {
          long start = System.nanoTime();
          int i = ITERATIONS;
          while (--i != 0) {
              int slot = i & MASK;
              byte b = (byte) i;
              arrayA[slot] = b;
              arrayB[slot] = b;
              arrayC[slot] = b;
          }
          i = ITERATIONS;
          while (--i != 0) {
              int slot = i & MASK;
              byte b = (byte) i;
              arrayD[slot] = b;
              arrayE[slot] = b;
              arrayF[slot] = b;
          }
          return System.nanoTime() - start;
      }
  }


  ```

  这个程序在我的Windows 7 64位英特尔酷睿i7860@2.8 GHz系统上产生的输出如下：


  > 1 SingleLoop duration (ns) = 14019753545
   1 SplitLoop  duration (ns) = 8972368661
   2 SingleLoop duration (ns) = 14162455066
   2 SplitLoop  duration (ns) = 8887610558
   3 SingleLoop duration (ns) = 13800914725
   3 SplitLoop  duration (ns) = 7271752889

  上面的例子说明：如果在一个循环中修改6个数组位置（内存地址），程序的运行时间明显长于将任务拆分的方式，即，先写前3个位置，再修改后3个位置。


  通过拆分循环，我们做了更多的工作，但程序花费的时间更少！欢迎利用神奇的“合并写（write
  combining）”。通过使用CPU架构的知识，正确的填充这些缓冲区，我们可以利用底层硬件加速我们的程序。


  不要忘了超线程（hyper-threading），可能会有2个线程竞争同一个核的缓冲区。


  转载自并发编程网 – ifeve.com 本文链接地址: [合并写(write
  combining)](http://ifeve.com/writecombining/)
toc: false
date: 2019-01-31 10:46:33
author:
thumbnail:
blogexcerpt:
---

**转载自并发编程网 – ifeve.com 本文链接地址: [合并写(write combining)](http://ifeve.com/writecombining/) 译者：无叶 校对：丁一**

现代CPU采用了大量的技术来抵消内存访问带来的延迟。读写内存数据期间，CPU能执行成百上千条指令。

多级SRAM缓存是减小这种延迟带来的影响的主要手段。此外，SMP系统采用消息传递协议来实现缓存之间的一致性。遗憾的是，现代的CPU实在是太快了，即使是使用了缓存，有时也无法跟上CPU的速度。因此，为了进一步减小延迟的影响，一些鲜为人知的缓冲区派上了用场。

本文将探讨“合并写存储缓冲区（write combining store buffers）”，以及如何写出有效利用它们的代码。

CPU缓存是一种高效的非链式结构的hash map，每个桶（bucket）通常是64个字节。这就是一个“缓存行（cache line）”。缓存行是内存交换的实际单位。例如，主存中地址A会映射到一个给定的缓存行C。


如果CPU需要访问的地址hash后的行尚不在缓存中，那么缓存中对应位置的缓存行会被清除，以便载入新的行。例如，如果我们有两个地址，通过hash算法hash到同一缓存行，那么新的值会覆盖老的值。

当CPU执行存储指令（store）时，它会尝试将数据写到离CPU最近的L1缓存。如果此时出现缓存未命中，CPU会访问下一级缓存。此时，无论是英特尔还是许多其它厂商的CPU都会使用一种称为“合并写（write combining）”的技术。

在请求L2缓存行的所有权尚未完成时，待存储的数据被写到处理器自身的众多跟缓存行一样大小的存储缓冲区之一。这些芯片上的缓冲区允许CPU在缓存子系统未准备好接收和处理数据时继续执行指令。当数据不在任何其它级别的缓存中时，将获得最大的优势。

当后续的写操作需要修改相同的缓存行时，这些缓冲区变得非常有趣。在将后续的写操作提交到L2缓存之前，可以进行缓冲区写合并。 这些64字节的缓冲区维护了一个64位的字段，每更新一个字节就会设置对应的位，来表示将缓冲区交换到外部缓存时哪些数据是有效的。

也许你要问，如果程序要读取已被写入缓冲区的某些数据，会怎么样？我们的硬件工程师已经考虑到了这点，在读取缓存之前会先去读取缓冲区的。

这一切对我们的程序意味着什么？

如果我们能在缓冲区被传输到外部缓存之前将其填满，那么将大大提高各级传输总线的效率。如何才能做到这一点呢？好的程序将大部分时间花在循环处理任务上。

这些缓冲区的数量是有限的，且随CPU模型而异。例如在Intel CPU中，同一时刻只能拿到4个。这意味着，在一个循环中，你不应该同时写超过4个不同的内存位置，否则你将不能享受到合并写（write combining）的好处。

```java
public final class WriteCombining {

    private static final int    ITERATIONS = Integer.MAX_VALUE;
    private static final int    ITEMS      = 1 << 24;
    private static final int    MASK       = ITEMS - 1;

    private static final byte[] arrayA     = new byte[ITEMS];
    private static final byte[] arrayB     = new byte[ITEMS];
    private static final byte[] arrayC     = new byte[ITEMS];
    private static final byte[] arrayD     = new byte[ITEMS];
    private static final byte[] arrayE     = new byte[ITEMS];
    private static final byte[] arrayF     = new byte[ITEMS];

    public static void main(final String[] args) {
        for (int i = 1; i <= 3; i++) {
            out.println(i + " SingleLoop duration (ns) = " + runCaseOne());
            out.println(i + " SplitLoop duration (ns) = " + runCaseTwo());
        }
        int result = arrayA[1] + arrayB[2] + arrayC[3] + arrayD[4] + arrayE[5] + arrayF[6];
        out.println("result = " + result);
    }

    public static long runCaseOne() {
        long start = System.nanoTime();
        int i = ITERATIONS;

        while (--i != 0) {
            int slot = i & MASK;
            byte b = (byte) i;
            arrayA[slot] = b;
            arrayB[slot] = b;
            arrayC[slot] = b;
            arrayD[slot] = b;
            arrayE[slot] = b;
            arrayF[slot] = b;
        }
        return System.nanoTime() - start;
    }

    public static long runCaseTwo() {
        long start = System.nanoTime();
        int i = ITERATIONS;
        while (--i != 0) {
            int slot = i & MASK;
            byte b = (byte) i;
            arrayA[slot] = b;
            arrayB[slot] = b;
            arrayC[slot] = b;
        }
        i = ITERATIONS;
        while (--i != 0) {
            int slot = i & MASK;
            byte b = (byte) i;
            arrayD[slot] = b;
            arrayE[slot] = b;
            arrayF[slot] = b;
        }
        return System.nanoTime() - start;
    }
}

```
这个程序在我的Windows 7 64位英特尔酷睿i7860@2.8 GHz系统上产生的输出如下：

> 1 SingleLoop duration (ns) = 14019753545
 1 SplitLoop  duration (ns) = 8972368661
 2 SingleLoop duration (ns) = 14162455066
 2 SplitLoop  duration (ns) = 8887610558
 3 SingleLoop duration (ns) = 13800914725
 3 SplitLoop  duration (ns) = 7271752889

上面的例子说明：如果在一个循环中修改6个数组位置（内存地址），程序的运行时间明显长于将任务拆分的方式，即，先写前3个位置，再修改后3个位置。

通过拆分循环，我们做了更多的工作，但程序花费的时间更少！欢迎利用神奇的“合并写（write combining）”。通过使用CPU架构的知识，正确的填充这些缓冲区，我们可以利用底层硬件加速我们的程序。

不要忘了超线程（hyper-threading），可能会有2个线程竞争同一个核的缓冲区。

转载自并发编程网 – ifeve.com 本文链接地址: [合并写(write combining)](http://ifeve.com/writecombining/)