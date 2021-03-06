---
title: Java 并发编程基础
date: 2018-11-11 22:29:51
tags: [Java, 并发, 多线程]
categories: 后端
updated:
keywords:
description:
top_img:
comments: true
cover:
toc:
toc_number:
copyright:
copyright_author:
copyright_author_href:
copyright_url:
copyright_info:
mathjax:
katex:
aplayer:
highlight_shrink:
aside:
---

## 线程

现代操作系统在运行一个程序时，会为其创建一个进程。例如，启动一个 Java 程序，操作系统就会创建一个 Java 进程。现代操作系统调度的最小单元是线程，也叫轻量级进程（LightWeight Process），在一个进程里可以创建多个线程，这些线程都拥有各自的计数器、堆栈和局部变量等属性，并且能够访问共享的内存变量。处理器在这些线程上高速切换，让使用者感觉到这些线程在同时执行。  
一个 Java 程序从 main()方法开始执行，然后按照既定的代码逻辑执行，看似没有其他线程参与，但实际上 Java 程序天生就是多线程程序，因为执行 main()方法的是一个名称为 main 的线程。

<!--more-->

## 多线程优点

- 更好地利用处理器资源
- 加快响应时间
- 更好的编程模型

## synchronized

synchronized 作为最开始学习 Java 就会接触到的关键字，他的使用方法有三种

- 修饰成员方法，为当前对象加锁，进入同步方法前要获得当前对象的锁。
- 修饰静态方法，对当前类对象加锁，需要获取当前类对象的锁。
- 修饰代码块，指定加锁对象，对指定的对象加锁。

## CAS

CAS 全称 Compare And Set（或 Compare And Swap）,CAS 包含三个操作数：内存位置(V)、原值(A)、新值(B)。简单来说 CAS 操作就是一个虚拟机实现的原子操作，这个原子操作的功能就是将旧值(A)替换为新值(B)，如果旧值(A)未被改变，则替换成功，如果旧值(A)已经被改变则替换失败。

在 UnSafe 中我们可以找到 CAS 相关的方法。
我们使用`java.util.concurrent.atomic.AtomicInteger`来做个例子，首先在多线程下 i++一般得不到我们要的值，因为他不是一个原子操作，而使用 AtomicInteger 的 incrementAndGet 则不会出错

```java
    /**
     * Atomically increments by one the current value.
     *
     * @return the updated value
     */
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
```

再看看 UnSafe 的 getAndAddInt

```java
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }

```

可以不停的进行 CAS 操作，直到成功。

## volatile

volatile 是轻量级的 synchronized，它在多处理器开发中保证了共享变量的“可见性”。可见性的意思是当一个线程修改一个共享变量时，另外一个线程能读到这个修改的值如果 volatile 变量修饰符使用恰当的话，它比 synchronized 的使用和执行成本更低，因为它不会引起线程上下文的切换和调度。

## AQS（AbstractQueuedSynchronizer）

AQS 可以说是并发包的基石了，JUC 包中工具基本都是建立在 AQS 上面的。
首先来看看 AQS 中的成员变量（方便阅读，所以将注释也搬了过来，可以对比学习）

```java

    /**
     * Head of the wait queue, lazily initialized.  Except for
     * initialization, it is modified only via method setHead.  Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     * 同步队列 头节点
     */
    private transient volatile Node head;

    /**
     * Tail of the wait queue, lazily initialized.  Modified only via
     * method enq to add new wait node.
     * 同步队列 队尾节点
     */
    private transient volatile Node tail;

    /**
     * The synchronization state.
     * 同步状态
     */
    private volatile int state;
     /**
     * 独占模式下当前锁的持有者（在父类AbstractOwnableSynchronizer中）
     */
    private transient Thread exclusiveOwnerThread;
```

- state： 非常关键的一个变量，他表示了当前锁的状态
- tail： 同步队列的队尾
- head：队头，其实不算队头，因为他已经占有了锁，head 后面的才是在队列中等待的

阻塞队列的情况如下：

```bash
           +------+ prev  +-----+ prev  +-----+
      head |      | <---- |     | <---- |     |  tail
           |      | ----> |     | ----> |     |
           +------+ next  +-----+ next  +-----+
```

再来看看内部静态类 Node，想要获取锁的线程被包装在 node 节点中，node 节点又组成了一个队列

```java
static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
        //------------标识锁类型-------------------------------------
        //共享锁
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        //独占锁
        static final Node EXCLUSIVE = null;
        //-----------------------------------------------------------
        //-------------------------等待状态---------------------------
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
        //----------------------------------------------------------

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
         //状态变量
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
         //当前节点的前驱
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
         //下一个节点 后驱
        volatile Node next;

        /**
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
         //当前线程 保存了想要获取锁的线程
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
         //下一个等待者
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
         //是否是共享节点
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
         //当前节点的前一个节点
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }
        //mode封装了锁类型 共享或者独占
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

接下以`ReentrantLock`为例说明 AQS 是如何运行的 。

### AQS 在 ReentrantLock 中的应用

ReentrantLock 提供的构造方法有两个。

```java
//无参构造方法，默认使用非公平锁
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    //如果为true则为公平锁，否则为非公平锁
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

```

#### NonfairSync

NonfairSync 类实现了非公平锁

```java
    /**
     * 非公平锁实现，继承Sync类
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * 首先调用加锁
         */
        final void lock() {
            //CAS操作加锁，将state值修改为 1则表示加锁成功（插队）
            if (compareAndSetState(0, 1))
            //将持有锁的线程设置为当前线程
                setExclusiveOwnerThread(Thread.currentThread());
            else
            //如果CAS操作失败（插队失败），尝试以独占锁方式加锁，acquire方法会调用tryAcquire(int acquires)方法尝试加锁，acquire()方法定义在AbstractQueuedSynchronizer（AQS）中
                acquire(1);
        }


    }
```

#### AQS 中 acquire 方法

在非公平锁的模式下，首先会使用 CAS 获取锁，如果获取锁失败，会调用`acquire(int arg)`方法，而在公平锁模式下，则直接调用了`acquire(int arg)`方法，来看看 acquire(int arg)方法内部

```java
    /**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     */
    public final void acquire(int arg) {
        //尝试获取锁，调用tryAcquire方法 ，tryAcquire在FairSync和NonfairSync中被定义，如果尝试加锁失败（有其他线程抢占了锁就会加锁失败）
        if (!tryAcquire(arg) &&
        //将线程添加到阻塞队列中 ，首先运行的是addWaiter（）方法，addWaiter()方法会将当前线程以独占模式的方式封装进Node节点中，并将该Node节点添加到队尾，然后返回该节点
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            //中断线程
            selfInterrupt();
    }
```

#### NonfairSync 中 tryAcquire 方法

acquire 先调用 tryAcquire 方法

```java
        //尝试加锁 在ReentrantLock中，参数 acquires的值为 1
        protected final boolean tryAcquire(int acquires) {
            //以非公平锁的方式尝试加锁 nonfairTryAcquire方法的实现在父类Sync中
            return nonfairTryAcquire(acquires);
        }
```

#### `Sync`类中的`nonfairTryAcquire(int acquires)` 方法

```java

        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         * 非公平锁方式尝试获取锁
         */
        final boolean nonfairTryAcquire(int acquires) {
            //当前线程
            final Thread current = Thread.currentThread();
            //获取state，如果C！=0，代表有线程获取了锁
            int c = getState();
            //如果没有线程获取了锁
            if (c == 0) {
                //使用CAS操作加锁（插队）
                if (compareAndSetState(0, acquires)) {
                    //加锁成功，设置持有锁的线程为当前线程
                    setExclusiveOwnerThread(current);
                    //返回true，获取锁成功
                    return true;
                }
            }
            //如果有线程获取了锁，并且是当前线程持有锁
            else if (current == getExclusiveOwnerThread()) {
                //在state基础上继续增加值 （锁重入）
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                    //更新state值
                setState(nextc);
                return true;
            }
            //如果前面的操作都失败，则加锁失败
            return false;
        }

```

#### 添加到同步队列

当获取锁失败后添加到阻塞队列

```java
    /**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     *
     * 添加节点到队尾
     */
        private Node addWaiter(Node mode) {
            //以独占模式添加新Node
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;
        //将当前节点添加到队尾
        if (pred != null) {
            node.prev = pred;
            //设置队尾为当前节点
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //如果设置尾节点失败 ，使用enq方法设置当前节点为尾节点，enq方法内部有个死循环 ，总会成功设置当前节点为队尾并且返回
        enq(node);
        return node;
    }
    /**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     *
     * addWaiter方法运行完毕后就会运行本方法
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            //注意这里是一个死循环
            for (;;) {
                //获取当前节点的前一个节点
                final Node p = node.predecessor();
                //如果前一个节点是head节点，head节点为占有锁的线程，说明当前线程是距离获得锁最近的线程，尝试获取锁（公平锁和非公平锁的区别就在于tryAcquire）
                if (p == head && tryAcquire(arg)) {
                    //如果获取锁成功，那么将头节点设置为自己
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //进入到这里时说明锁已经被其他线程占用，判断是否需要挂起线程，如果需要挂起返回true
                if (shouldParkAfterFailedAcquire(p, node) &&
                //挂起线程  等待唤醒
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

> 总结：简单的说就是先尝试获取锁，如果获取锁失败，那么就将线程加入队列，加入队列后如果是在 head 节点之后的第一个节点，那么再次尝试获取锁，如果没有在 head 节点后，那么判断是否应该将线程挂起，如果线程被挂起，那么就需要等待前驱唤醒（前驱释放锁之后会唤醒后继节点）。

#### 公平锁实现`FairSync`

```java
    /**
     * 公平锁
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;
//加锁 和上面的 NonfairSync 的区别在于少了插队操作
        final void lock() {
            //以独占锁的方式加锁 acquire()方法会调用tryAcquire()方法
            acquire(1);
        }

        /**
         * 尝试加锁
         */
        protected final boolean tryAcquire(int acquires) {
            //获取当前线程
            final Thread current = Thread.currentThread();
            //获取state，c！=0 代表有线程持有锁，如果没有线程持有锁则为0
            int c = getState();
            if (c == 0) {
                //查看阻塞队列中是否有线程在排队等待获取锁，如果有其他线程在当前线程前面，则返回true，否则返回false
                if (!hasQueuedPredecessors() &&
                //如果当前线程前面没有其他线程排队，则使用CAS修改state变量，尝试加锁
                    compareAndSetState(0, acquires)) {
                // 加锁成功，设置持有锁的线程为当前线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //如果state不等于0 判断持有锁的线程是否是当前线程
            else if (current == getExclusiveOwnerThread()) {
                //如果是当前线程，在state基础上加对应的值 （锁重入）
                int nextc = c + acquires;
                //如果小于0，报错，state最小为0
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                //更新state值
                setState(nextc);
                return true;
            }
            //如果前面的操作都没有成功，则尝试加锁失败
            return false;
        }
    }
```

#### 解锁

根据前面内容知道了如果线程获取锁失败，会进入挂起状态，然后等待前驱节点的唤醒。前驱节点释放锁之后就会唤醒后继节点。

```java
    //调用解锁方法
    public void unlock() {
            //解锁
        sync.release(1);
    }
    public final boolean release(int arg) {
        //尝试释放锁
        if (tryRelease(arg)) {
            //如果释放锁成功
            //获取占有锁的节点
            Node h = head;
            if (h != null && h.waitStatus != 0)
            //唤醒后继节点
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    /**
     * Attempts to set the state to reflect a release in exclusive
     * mode.
     *
     * <p>This method is always invoked by the thread performing release.
     *
     * <p>The default implementation throws
     * {@link UnsupportedOperationException}.
     *
     * @param arg the release argument. This value is always the one
     *        passed to a release method, or the current state value upon
     *        entry to a condition wait.  The value is otherwise
     *        uninterpreted and can represent anything you like.
     * @return {@code true} if this object is now in a fully released
     *         state, so that any waiting threads may attempt to acquire;
     *         and {@code false} otherwise.
     * @throws IllegalMonitorStateException if releasing would place this
     *         synchronizer in an illegal state. This exception must be
     *         thrown in a consistent fashion for synchronization to work
     *         correctly.
     * @throws UnsupportedOperationException if exclusive mode is not supported
     *
     * 释放锁
     */
    protected final boolean tryRelease(int releases) {
        //获取释放后的state状态，c=0 才释放锁成功
        int c = getState() - releases;
        //如果要释放锁的线程不是拥有锁的线程，报错
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        //如果释放后的state为0
        if (c == 0) {
            free = true;
            //设置占有锁的线程为null
            setExclusiveOwnerThread(null);
        }
        //更新state
        setState(c);
        return free;
    }
```

当解锁成功之后，`unparkSuccessor(Node node)`方法会唤醒后继节点，通过前面的代码我们知道线程在`parkAndCheckInterrupt()`处被挂起了，当线程被唤醒后，`parkAndCheckInterrupt()`方法会判断线程是否处于停止状态，一般情况下返回 false，所以代码会重新进入循环（外面有个死循环），重新获取锁，如果获取失败，继续挂起，或者退出线程。
