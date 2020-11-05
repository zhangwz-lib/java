

# AQS详解 

## AQS说明

AQS在实现时主要有四种 共享模式、独占模式、公平锁、非公平锁

> > AQS 实现的基础 是一个双端链表以及一个state，双端链表存储了等待获取资源的线程，state 可以看作是许可证书，如果你获得了许可证书就表示 你获取了资源。
> >
> > 共享锁可以看作是这个许可证书有多个，可以有多个线程获取，独占模式 只有一个许可证书，只有一个线程可以获取。
> >
> > 公平与非公平则是按照请求获取锁的先后顺序来划分的

## AQS CLH  (ASQ.node())

```java
static final class Node {
    
        /**表示节点为共享节点 */
        static final Node SHARED = new Node();
        /** 表示节点为独占节点 */
        static final Node EXCLUSIVE = null;

       /** 表示节点已经被取消，被取消的节点不会再被唤醒*/
        static final int CANCELLED =  1;
        /** 表示下一个节点需要被唤醒。所以当新的节点插入时都会将上一个节点的状态设置为SINGAL，否则后续节点不会被唤醒。当前一个节点处于该状态时，释放锁或者状态设置为CANCELLED 时，则会唤醒下一个节点 */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * 表示在共享模式下 可以唤醒后续节点 
         */
        static final int PROPAGATE = -3;

        /**
        
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
      
    }
```

## AQS.acquire()



```
 public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    
    
```

## AQS.acquireShared()

```java
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

_____

```java

private void doAcquireShared(int arg) {
    /** 添加一个 共享模式的节点到队列 */
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                /** 获取当前节点的前一个节点 */
                final Node p = node.predecessor();
                if (p == head) {
                    /** 再次尝试获取锁 */
                    int r = tryAcquireShared(arg);
                    /** r >=0 说明由于其他线程释放了锁， 也就是此时的当前线程获取到的head节点可能已经被释放*/ 
                    if (r >= 0) {
                        /**  设置header 满足一定条件下解锁后续节点*/
                        setHeadAndPropagate(node, r);
                    
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                /** 获取失败 阻塞当前线程*/
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

   
```

_________

```java
private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
    
    /**满足以下条件时 执行doReleaseShared()  */
    
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                /** 唤醒后续节点*/
                doReleaseShared();   
        }
    /**
       1. propagate > 0    propagate 是tryAcquireShared(arg)返回的值，此值大于0 表示剩余的资源数大于0，所以后续节点仍然可以获取
       2. h == null || h.waitStatus < 0 说明 先前的头节点被其他线程释放，后续节点可以获取资源
       3. (h = head) == null || h.waitStatus < 0 同上，说明新的头节点被释放，节点可以释放资源
    */
    }
```



## AQS.release()

```
 public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

## AQS.releaseShared()

```
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
}

```

____

```java
  private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                /**  singal 表示在节点释放时需要唤醒下一个节点。所以，每个节点添加的时候都会将前一个节点的状态置为singal
                可以参见 doAcquireShared */
                // cas 修改节点状态状态为0 所以如果节点的状态是0 则要么是未节点 要么是头节点已经释放资源
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    /**唤醒下一个节点 */
                    unparkSuccessor(h);
                }
                /**Node.PROPAGATE 状态表示在共享模式下可以唤醒后续节点 这也是为什么在 setHeadAndPropagate() 方法中 h.waitStatus() 状态 < 0 的原因 */
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            /** 如果 h!=head 说明 head节点已经被其他线程更新*/ 
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

____

```java
 private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

       /**  s.waitStatus > 0 说明节点的状态已经取消 ，这个循环的目的是为了跳过节点状态为取消的节点*/
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
     // 唤醒被阻塞的线程  虽然被阻塞的线程被唤醒但是 该线程不一定能够获取到cpu的时间片，可能会重新阻塞
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

## 公平锁与非公平锁实现区别

### 独占非公平锁

```java

            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
```

___

### 独占公平锁

```java
 		int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
```

> > 从上面可以看出两者之间最大的区别在于 **hasQueuedPredecessors()**  方法

```java
/**该方法的主要目的是 判断头节点的下一个节点是否 是当前节点*/
public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
/**
1. h!=t 说明 头节点有后置节点 
2.(s = h.next) == null 说明当前线程获取到的头节点已经被修改，也就是说已经有其他线程获取到了锁
3.(s = h.next) == null || s.thread != Thread.currentThread()   说明头节点的后置节点不是当前节点
*/
```

> > **之所以这样判断，是因为头节点是当前持有锁的节点，为保证公平性唤醒他的下一个节点,如果下一个节点不是当前节点则获取锁失败**

## ConditionObject 相关实现