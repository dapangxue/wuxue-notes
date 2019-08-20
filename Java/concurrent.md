# 并发

[TOC]

## 一、线程安全和线程不安全

怎么理解一个类是线程安全的？

多个线程访问某个类时，不管运行时环境采用何种调度方式或者这些线程如何交替执行，并且在主调代码中不要任何额外的同步或者协同，这个类都能表现出正确的行为，就称类是线程安全的。

## 二、AbstractQueuedSynchronizer

很多锁以及并发控制的类都是基于AQS实现的。它的内部有一个`state`的变量。

```java
    private volatile int state;
```

这个变量可以通过setState、getState、compareAndSetState来修改它的值，它表示了一个状态。具体表示什么状态需要类的实现者自己规定。

在j.u.c包下面的类中，

+ ReentrantLock中用state表示获得锁的线程获得该锁的次数（可重入锁）
+ Semaphore用state表示剩余的许可数量
+ FutureTask用state表示任务的状态（尚未开始、正在运行、已完成以及已取消）、
+ CountDownLatch用state表示允许拿到共享锁的线程数。

它内部提供了可以用于覆写的方法，用于采用了AQS的组件的实现者实现自己想要的逻辑。主要有

+ tryAcquire(int)，尝试独占式的获取锁
+ tryRelease(int)，尝试独占式的释放锁
+ tryRequireShared(int)，尝试共享式的获取锁
+ tryReleaseShared(int)，尝试共享式的释放锁
+ tryIsHeldExclusively()，判断锁是否被当前线程独占

AQS内部提供了通用的方法供实现类选择使用。

### acquire()

acquire()方法用于表示独占式的获取锁。

```JAVA
    public final void acquire(int arg) {
        // 如果tryAcquire(1)获取锁失败，条件继续执行，构造独占的节点
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

```JAVA
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            // 通过CAS保证了尾节点的正确添加，如果多个线程调用tryAcquire(arg)获取同步状态失败，那么此处就需要通过cas来保证同步队列的正确添加
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 上述尾节点设置失败后，在enq方法中给同步队列添加尾节点
        enq(node);
        return node;
    }
```

在addWaiter(Node)中添加节点失败后，会进入enq()方法，通过死循环不断地添加尾节点。

```JAVA
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

加入同步队列后的线程进入了一个自旋的状态，`如果当前节点的上一个节点是头结点，且当前节点获取同步状态成功`，那么就退出同步状态。

```JAVA
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
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

### release()

```JAVA
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

### acquireShared()

```JAVA
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

和tryAcquire(arg)返回的boolean值不同，tryAcquireShared(arg)返回的是一个int型的整数。如果返回值小于0，表示获取同步状态失败，那么自旋的获取同步状态，获取同步状态的终止条件就是tryAcquireShared(arg)大于等于0。

```JAVA
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    // 终止条件
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
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

### releaseShared()

``` JAVA
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```



