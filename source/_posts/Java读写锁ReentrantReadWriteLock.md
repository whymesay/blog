---
title: Java读写锁ReentrantReadWriteLock
date: 2019-01-23 15:12:22
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

## 读写锁

> 读写锁实际是一种特殊的自旋锁，它把对共享资源的访问者划分成读者和写者，读者只对共享资源进行读访问，写者则需要对共享资源进行写操作。这种锁相对于自旋锁而言，能提高并发性，因为在多处理器系统中，它允许同时有多个读者来访问共享资源，最大可能的读者数为实际的逻辑 CPU 数。写者是排他性的，一个读写锁同时只能有一个写者或多个读者（与 CPU 数相关），但不能同时既有读者又有写者。  
> 如果读写锁当前没有读者，也没有写者，那么写者可以立刻获得读写锁，否则它必须自旋在那里，直到没有任何写者或读者。如果读写锁没有写者，那么读者可以立即获得该读写锁，否则读者必须自旋在那里，直到写者释放该读写锁。

## ReedtrantReadWriteLock 的基本使用

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class RWLockTest {

    /**
     * 读写锁
     */
    private ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    private volatile int num = 0;

    public static void main(String[] args) {
        //新建一个实例
        RWLockTest rwLockTest = new RWLockTest();
        Thread t1 = new Thread(() -> {
            for (int i = 1; i <= 10; i++) {
                rwLockTest.write(i);
            }
        });
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                rwLockTest.read();
            }
        });
        //启动线程
        t1.start();
        t2.start();
    }


    public void read() {
        readWriteLock.readLock().lock();
        try {
            System.out.println("读取：num=" + num);

        } finally {
            readWriteLock.readLock().unlock();
        }
    }

    public void write(int i) {
        readWriteLock.writeLock().lock();
        try {
            num = i;
            System.out.println("写入" + i + ":num=" + num);
        } finally {
            readWriteLock.writeLock().unlock();
        }
    }

}
```

## ReedtrantReadWriteLock 解析

ReedtrantReadWriteLock 变量和构造方法

```java
    /** 读锁 */
    private final ReentrantReadWriteLock.ReadLock readerLock;
    /** 写锁 */
    private final ReentrantReadWriteLock.WriteLock writerLock;
    final Sync sync;

    /**
     * 默认为非公平锁
     */
    public ReentrantReadWriteLock() {
        this(false);
    }

    /**
     *
     * @param fair 如果为true则为公平锁
     */
    public ReentrantReadWriteLock(boolean fair) {
        //设置为公平锁或者非公平锁
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
```

从上面的构造方法可以看出 ReedtrantReadWriteLock 拥有两个锁，一个写锁（排它锁），一个读锁（共享锁）。

```java
//返回写锁
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
//返回读锁
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```

lock()方法或者 unlock()方法。

```java
//这里只列出了主要的两个方法
public static class WriteLock implements Lock, java.io.Serializable {
        private final Sync sync;

//构造方法，保存Sync
        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
        //加锁
        public void lock() {
            //写锁加锁 独占锁  在AQS中被实现
            sync.acquire(1);
        }

        public void unlock() {
            //写锁解锁
            sync.release(1);
        }
}
public static class ReadLock implements Lock, java.io.Serializable {
     private final Sync sync;

        protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }

        public void lock() {
            //读锁解锁 共享锁
            sync.acquireShared(1);
        }

        public void unlock() {
            //读锁解锁
            sync.releaseShared(1);
        }
}

```

从上面的代码中可以得出 ReentrantReadWriteLock 有两个锁，读锁和写锁，而读锁和写锁都是通过 Sync 来进行加锁，不同的点在于调用 Sync 的方法不同

Sync：

```java
abstract static class Sync extends AbstractQueuedSynchronizer {

    //******************************************************************************************
        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        //最大锁数量  65535 2的16次方减一
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        //二进制值为  0000 0000 0000 0000 1111 1111 1111 1111
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** 返回共享锁的数量  也就是读锁 */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** 返回排它锁的数量  也就是写锁  */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
        /**
            对于上面的这段代码，同学们可能不是很清楚
            我们知道AQS只有一个变量state代表锁状态，但是读锁和写锁有两个锁，如何表示两个锁呢？
            首先state是int型 32位，这里将state的高16位作为读锁，低16位作为读锁
            再看看 c >>> SHARED_SHIFT 将state无符号右移16位，是不是就只剩下高位16位，也就是读锁的数量。
            然后  c & EXCLUSIVE_MASK 我们知道EXCLUSIVE_MASK的值为16个1，前面16位都是0，然后进行按位与计算，
            最后的结果就是低16位的值，也就是写锁的数量。
        */
        //用于记录当前线程的读锁持有数量
        private transient ThreadLocalHoldCounter readHolds;
        //缓存最后一个获取读锁的线程的读锁重入次数
        private transient HoldCounter cachedHoldCounter;
        //第一个获取读锁的线程，必须持有读锁，释放之后就不算第一个了。
        private transient Thread firstReader = null;
        //第一个获取读锁的线程的锁重入数
        private transient int firstReaderHoldCount;
    //-----------------------------------------------------------


    //----------------------写锁加锁与解锁-----------------------
        protected final boolean tryAcquire(int acquires) {
            //获取当前线程
            Thread current = Thread.currentThread();
                //获取锁整体状态
            int c = getState();
            //求出写锁状态
            int w = exclusiveCount(c);
            //如果state不等于0
            if (c != 0) {
                // 如果整体状态 c 不等于0，并且写锁 w 为0，那么读锁一定不为 0
                //所以在w==0或者写锁占有者不是当前线程都加锁失败
                if (w == 0 || current != getExclusiveOwnerThread())
                //在读锁存在或者有其他写锁占有的情况下，写锁加锁失败
                //返回false 进入阻塞队列，如果忘了 请回顾上一章内容
                    return false;
                //走到这一步说明没有读锁，并且持有写锁的线程为自己
                //如果继续为当前线程加锁（锁重入），加锁数大于了最大值65535 报错
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // 更新锁状态
                setState(c + acquires);
                return true;
            }
            //走到这一步的时候 说明c==0 没有读锁 也没有写锁
            //根据是否是公平锁来选择是否优先为写锁加锁
            if (writerShouldBlock() ||
            //尝试CAS获取锁 （插队）
                !compareAndSetState(c, c + acquires))
                return false;
                //如果写锁加锁成功，设置持有锁的线程为当前线程
            setExclusiveOwnerThread(current);
            return true;
        }

        //锁释放 在AQS中被调用
        protected final boolean tryRelease(int releases) {
            //如果释放锁的线程不是持有锁的线程 报错
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
                //获取锁整体状态并减去对应的值
            int nextc = getState() - releases;
            //如果最后的写锁状态为0，说明锁已经释放
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
            //设置锁持有者为null
                setExclusiveOwnerThread(null);
                //更新state
            setState(nextc);
            return free;
        }
    //----------------------写锁加锁与解锁结束----------------------------

    //----------------------------读锁加锁与解锁----------------------------
    /**
    由读锁加锁时调用
    acquireShared方法在AQS中的实现
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
    */
        protected final int tryAcquireShared(int unused) {
            //获取当前线程
            Thread current = Thread.currentThread();
            //获取锁整体状态
            int c = getState();
            //如果写锁存在并且持有写锁的线程不是当前线程，直接返回失败
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                //返回 -1 即失败
                return -1;
            //获取读锁
            int r = sharedCount(c);
            //判断读线程是否应该阻塞（可以自己下来了解该方法，这里不做讲解）
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                //如果读线程不应该阻塞，尝试加上读锁
                compareAndSetState(c, c + SHARED_UNIT)) {
                    //如果读线程为0，说明之前没有其他读线程
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                    //如果读线程为获取读锁的第一个线程，读锁重入数加一
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    //最后一个获取读锁的线程
                    HoldCounter rh = cachedHoldCounter;
                    //如果最后一个获取读锁的线程不是当前线程 ，设置缓存为当前线程
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            //如果读锁加锁失败调用接下来的方法自旋获取读锁
            return fullTryAcquireShared(current);
        }

        final int fullTryAcquireShared(Thread current) {
            HoldCounter rh = null;
            //自旋
            for (;;) {
                //获取锁整体状态
                int c = getState();
                //如果写锁被占用
                if (exclusiveCount(c) != 0) {
                    //写锁的持有线程不是当前线程
                    if (getExclusiveOwnerThread() != current)
                    //返回 -1  
                        return -1;
                    //判断读锁是否应该被阻塞，公平锁与非公平锁锁有不同的实现
                } else if (readerShouldBlock()) {
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                //读锁数量达到最大值  报错
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                //CAS设置读锁
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    //接下来的代码基本和前面的代码差不多，不再说明了
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }

        //释放锁
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            //如果当前线程为第一个获取了读锁的线程
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                //如果锁获取数量为1 则将firstReader置为null 否则将锁的持有数量减一
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                //更新当前线程读锁的持有数量
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            //自旋
            for (;;) {
                int c = getState();
                //减少读锁
                int nextc = c - SHARED_UNIT;
                //CAS更新
                if (compareAndSetState(c, nextc))
                    // 如果读锁全部释放（也就是0） 返回true
                    return nextc == 0;
            }
        }
        //----------------------------读锁加锁与解锁解锁----------------------------
    }
```

## 锁降级

官方示例
```java
class CachedData {
   Object data;
   volatile boolean cacheValid; //用于检测数据是否已经修改过了，如果被修改过了，就不许修改了 
   ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

   void processCachedData() {
     rwl.readLock().lock();//@1
     if (!cacheValid) {
        // Must release read lock before acquiring write lock
        rwl.readLock().unlock();//@4
        rwl.writeLock().lock();//@2
        // Recheck state because another thread might have acquired
        //   write lock and changed state before we did.
        if (!cacheValid) {//@3
          data = ...
          cacheValid = true;
        }
        // Downgrade by acquiring read lock before releasing write lock
        rwl.readLock().lock();
        rwl.writeLock().unlock(); // Unlock write, still hold read
     }

     use(data);
     rwl.readLock().unlock();
   }
 }
```
锁降级主要是为了保证在使用时，保证数据没有被其他线程重写，能获取到最新的数据。主要操作就是在写锁没有释放之前获取读锁。