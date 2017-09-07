title:  Java并发：AbstractQueuedAsynchronizer解析（二）
date: 2017-06-29
tags:
 - 多线程
 - 并发
 - 源码解读
categories:
 - 原创文章

---

在上一篇文章中，主要讲解了AQS的大体结构和用法，在本篇文章中则主要讲述AQS中等待队列的原理及实现。希望通过对AQS源码的解读加深自己去AQS的原理理解以及对AQS使用的熟练度。

<!-- more -->

# Node类介绍

`AbstractQueuedAsynchronizer.Node`该类是CLH（Craig, Landin, and Hagersten）锁队列的一个变种。CLH锁通常用于自旋锁。对于阻塞的同步器，我们也使用相同的策略来掌握其上一个节点的线程的控制信息。每个节点都有一个`status`字段用来记录该节点的线程是否应该被阻塞。每当一个节点被释放后，它的下游节点将被唤醒。队列中的每个节点也将用作一个具有特定通知的监视器，该监视器保存一个等待线程。状态字段不控制线程是否是
授予锁等。队列中的第一个线程将会尝试获取锁，但是只保证有权利去竞争锁，并不保证一定会获取成功。所以当线程被唤醒后重新竞争也意味着它们可能会被重新进入等待队列。


# 实现原理

对于CLH队列的入队，你需要将新节点以原子的方式拼接到最后。对于出队，你只需要设置`head`指针就可以了。

队列的形式如下：

```
     +------+  prev +-----+       +-----+
head |      | <---- |     | <---- |     |  tail
     +------+       +-----+       +-----+
```

插入一个新节点到CLH队列只需要在队列尾部`tail`进行单个原子操作，所以有一个简单的原子点（耗费很少的时间）从无排队到排队。同样的，出队只涉及`head`指针的更新。然而，节点需要更多的消耗来确定他们的后继者，部分原因是由于超时和中断而可能的取消。

`prev`指针（在原来的CLH锁中未使用）主要用来处理取消。如果一个节点被取消了，那么它的后继者需要重新连接到一个没有被取消的前驱节点。关于自旋锁的解释，可以参看[Scott和Scherer的论文](http://www.cs.rochester.edu/u/scott/synchronization/)

我们同样使用一个`next`指针来实现阻塞机制。每个节点都保存了它自己的线程id，所以前驱节点可以根据`next`指针知道应该唤醒哪一个线程。后继者必须要避免和新入队的节点竞争`next`指针的设置。必要时通过从原子向后检查来解决这个问题，当节点的后继显示为空时更新`tail`。（或者说，不同的是，`next`指针是一个优化手段，这样我们通常不需要反向扫描。）

取消采用了一种比较保守的做法。由于我们必须轮询其它节点是否被取消，我们可以忽略取消的节点在我们前面还是后面，这取决于我们将后继节点接驳到一个没有被取消的前驱节点上，由这个前驱节点来承担后继节点的唤醒职责。

CLH队列需要一个虚拟头节点才能开始。 但是，我们不会在构造中创造它们，因为如果从来没有竞争，那么所做的这些工作都是浪费。 相反，我们只在第一次争用发生时设置头和尾指针。

等待条件的线程使用相同的节点，但是使用额外的指针。只需要使用普通的链表队列（非并发队列）来保存节点，因为它们是在排它情况下进行访问的。在等待时，将节点插入到等待队列。在唤醒时，节点被转换到待运行队列，节点的`status`值被用来标识该节点处于什么队列之中。

```java
static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;

        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;

        /**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified using CAS
         * (or when possible, unconditional volatile writes).
         */
        volatile int waitStatus;

        /**
         * Link to predecessor node that current node/thread relies on
         * for checking waitStatus. Assigned during enqueuing, and nulled
         * out (for sake of GC) only upon dequeuing.  Also, upon
         * cancellation of a predecessor, we short-circuit while
         * finding a non-cancelled one, which will always exist
         * because the head node is never cancelled: A node becomes
         * head only as a result of successful acquire. A
         * cancelled thread never succeeds in acquiring, and a thread only
         * cancels itself, not any other node.
         */
        volatile Node prev;

        /**
         * Link to the successor node that the current node/thread
         * unparks upon release. Assigned during enqueuing, adjusted
         * when bypassing cancelled predecessors, and nulled out (for
         * sake of GC) when dequeued.  The enq operation does not
         * assign next field of a predecessor until after attachment,
         * so seeing a null next field does not necessarily mean that
         * node is at end of queue. However, if a next field appears
         * to be null, we can scan prev's from the tail to
         * double-check.  The next field of cancelled nodes is set to
         * point to the node itself instead of null, to make life
         * easier for isOnSyncQueue.
         */
        volatile Node next;

        /**
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
        volatile Thread thread;

        /**
         * Link to next node waiting on condition, or the special
         * value SHARED.  Because condition queues are accessed only
         * when holding in exclusive mode, we just need a simple
         * linked queue to hold nodes while they are waiting on
         * conditions. They are then transferred to the queue to
         * re-acquire. And because conditions can only be exclusive,
         * we save a field by using special value to indicate shared
         * mode.
         */
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * Returns previous node, or throws NullPointerException if null.
         * Use when predecessor cannot be null.  The null check could
         * be elided, but is present to help the VM.
         *
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

在以上代码中：

- `prev`指针被用于检查`waitStatus`，该指针在入队的时候被设置，然后在出队的时候被置空。同时，在前驱节点被取消时，我们可以快速的通过遍历`prev`找到新的非取消节点作为新的前驱节点。非取消状态的前驱节点是一定存在的，因为`head`节点永远不会被取消：原因是一个节点成为`head`节点的前提是该节点成功获取到锁。一个被取消节点的线程永远不会获取到锁，并且线程只能取消属于它自己的节点，并不能取消其它不属于它的节点。

- `next`指针被用于`release`释放锁的阶段。该指针在入队，前驱节点被取消时被设置，以及在出队时被置空（有助于GC回收）。入队操作直到attachment后才设置前驱节点的`next`字段，所以看到一个空的`next`字段并不一定意味着该节点在队列的结尾。 但是，如果`next`字段看起来是空的，我们可以从尾部反向扫描prev，以进行双重检查。被取消节点的`next`字段被设置为指向节点本身而不是null，这更有利于AQS的isOnSyncQueue方法。

isOnSyncQueue方法：
```java
    /**
     * Returns true if a node, always one that was initially placed on
     * a condition queue, is now waiting to reacquire on sync queue.
     * @param node the node
     * @return true if is reacquiring
     */
    final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
        return findNodeFromTail(node);
    }

    /**
     * Returns true if node is on sync queue by searching backwards from tail.
     * Called only when needed by isOnSyncQueue.
     * @return true if present
     */
    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }
```

# 总结

AQS中Node类提供了一个双向链表结构，分别使用`prev`和`next`指针构成双向指针。 `prev`指针在入队时设置，在出队时置空；`next`指针在入队，前驱节点被取消被设置，在出队时被置空。由该类构造成了AQS的等待队列，并在AQS中发挥重要的作用。