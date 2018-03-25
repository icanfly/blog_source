title: Java并发：AbstractQueuedSynchronizer解析（一）
date: 2017-06-24
tags:
 - 多线程
 - 并发
 - 源码解读

categories:
 - 原创文章
toc: true
---

> 注：本文基于JDK 8，全文中所有的叙述都是基于该版本。

# 概述

`AbstractQueuedSynchronizer`类是JUC类库的核心实现，它是实现Java并发核心库众多并发工具的基础，基于它及它的衍生品的并发核心包括`ReentrantReadWriteLock`,`ArrayBlockingQueue`,`CopyOnWriteArrayList`,`CountDownLatch`,`CyclicBarrier`等等。它为实现同步锁以及相关的基于FIFO等待队列的同步器（如：semaphores,events等）提供了一个统一框架。该类被设计用来为绝大多数基于一个原子int型状态值的同步器提供有用的基础设施。子类必须实现protected方法来改变这个状态值，该状态值决定了对象是被acquired还是released。该类的其它方法提供了所有的队列和阻塞管理。子类可以维护其它的状态字段，但是只有通过使用`getState`,`setState`,`compareAndSetState`方法才能被同步追踪。

<!-- more -->

子类应该被定义为非公开的内部帮助类来实现它闭包类的同步属性。`AbstractQueuedSynchronizer`未实现任何同步接口。相反，它定义了一些方法比如`acquireInterruptibly`被用于具体的锁或者同步器来实现它们的public方法。

该类提供了两种模式：排它和共享，当处于排它模式时，其它线程的尝试请求(attempted acquires)将不会成功，而对于共享模式来说，其它线程的请求则会成功（但不是必须）。该类并不理解这些不同点当一个共享模式获取成功时，下一个等待纯种（如果存在）也必须确定是否可以也获取到。不同的模式都共享相同的FIFO队列，通常，子类只实现这两种模式中的一种，但是这两者都可以在一个例子中发挥作用，比如ReadwriteLock。只支持排它或者共享模式的子类，不用去实现那些不支持模式的方法，因为也不会用到这些方法。

该类定义了一个嵌套的`ConditionObject`类，该类实现了`Condition`接口并被用于子类方法的排它模式。该类提供了检查和监控的方法以便于使用到`AbstractQueuedSynchronizer`的这些类可以很方便的使用。

该类的序列化只会存储一个维护状态的原子int值，所以对于反序列化来说，反序列化对象中线程等待队列将会为空，当然子类也可以自己定义一个`readObject`方法用于自定义类的状态恢复。

# 用法

使用该类作为同步器的一个基础，需要实现以下方法（这些方法可以使用`getState`,`setState`,`compareAndSetState`作为检测和修改同步状态的方法）：

- tryAcquire
- tryRelease
- tryAcquireShared
- tryReleaseShared
- isHeldExlusively

以上这些方法默认在AQS中被实现为抛出`UnsupportedOperationException`，这些方法的实现必须是内部线程安全的，并且应该是快速的且不被阻塞的。这些方法是唯一该类的方法，其实方法都被定义成`final`，因为它们的逻辑不能被继承子类修改。你也许也会发现该类从`AbstractOwnableSynchronizer`类继承下来的一些方法对于保持对排它同步器的线程追踪非常有帮助，同时也鼓励使用这些方法来加强对持有锁的线程的监控。

虽然该类是基于内部的一个FIFO队列，但是它不会自动的采用，排它同步器的核心采用以下形式：

获取锁（伪代码）：
```
while (!tryAcquire(arg)) {
         enqueue thread if it is not already queued;
         possibly block current thread;
}
```

释放锁（伪代码）：
```
if (tryRelease(arg))
         unblock the first queued thread;
```

因为获取锁时检查是在入队之前，所以新的获取锁的线程会被放到等待队列的最前面。然而如果你愿意，你也可以重新定义`tryAcquire`或者`tryAcquireShared`来改写这样的规则来提供一个公平的FIFO顺序。在特殊情况下，大部分公平同步器会在`hasQueuedPredecessors`返回true的情况下将`tryAcquire`方法定义为返回false。

对于默认的取锁方式（也称之为贪婪），吞吐量和扩展性通常情况都是最高的。然而这里并不保证公平，也不保证没有饥饿的出现，更早入队的线程是可以再次争抢锁的，并且拥有和新进线程一样的机会。而且通常意义上的自旋是在被阻塞之前它们可能会多次执行`tryAcquire`，这些可能会穿插在其它计算之间。自旋对于排它锁持有的时间通常很短时将发挥非常大的用处，然而对于持有锁时间比较长的情况下，这样就会造成非常大的计算浪费。如果需要，你也可以提前检测`hasContended`或者`hasQueuedThreads`这些`快速路径`来确认锁不会被竞争。

该类提供了一个有效的易扩展的工具，凡是使用int型状态作为同步状态同时使用一个FIFO队列的的同步器都可以使用。如果这些都不能满足你的话，你也可以使用`java.util.concurrent.atomic`包，你自己定义的`java.util.Queue`实现类以及`LockSupport`阻塞支持来自己实现一个同步器。

# 例子

下面这个例子是一个不可重入的排它锁实现，它使用int型的0值表示未被加锁，1值表示被加锁。

```java
class Mutex implements Lock, java.io.Serializable {

    // Our internal helper class
    private static class Sync extends AbstractQueuedSynchronizer {
      // Reports whether in locked state
      protected boolean isHeldExclusively() {
        return getState() == 1;
      }

      // Acquires the lock if state is zero
      public boolean tryAcquire(int acquires) {
        assert acquires == 1; // Otherwise unused
        if (compareAndSetState(0, 1)) {
          setExclusiveOwnerThread(Thread.currentThread());
          return true;
        }
        return false;
      }

      // Releases the lock by setting state to zero
      protected boolean tryRelease(int releases) {
        assert releases == 1; // Otherwise unused
        if (getState() == 0) throw new IllegalMonitorStateException();
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
      }

      // Provides a Condition
      Condition newCondition() { return new ConditionObject(); }

      // Deserializes properly
      private void readObject(ObjectInputStream s)
          throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
      }
    }

    // The sync object does all the hard work. We just forward to it.
    private final Sync sync = new Sync();

    public void lock()                { sync.acquire(1); }
    public boolean tryLock()          { return sync.tryAcquire(1); }
    public void unlock()              { sync.release(1); }
    public Condition newCondition()   { return sync.newCondition(); }
    public boolean isLocked()         { return sync.isHeldExclusively(); }
    public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
    public void lockInterruptibly() throws InterruptedException {
      sync.acquireInterruptibly(1);
    }
    public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
      return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
  }
```

下面这个例子是一个类似`CountDownLatch`的Latch。

```java
class BooleanLatch {

    private static class Sync extends AbstractQueuedSynchronizer {
      boolean isSignalled() { return getState() != 0; }

      protected int tryAcquireShared(int ignore) {
        return isSignalled() ? 1 : -1;
      }

      protected boolean tryReleaseShared(int ignore) {
        setState(1);
        return true;
      }
    }

    private final Sync sync = new Sync();
    public boolean isSignalled() { return sync.isSignalled(); }
    public void signal()         { sync.releaseShared(1); }
    public void await() throws InterruptedException {
      sync.acquireSharedInterruptibly(1);
    }
  }
 ```
