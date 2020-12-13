#### AQS的具体实现二：

一、简介

​	CountDownLatch（线程计数器）与CyclicBarrier（可重复使用的栅栏），线程计数器和循环栅栏是两个用来进行同步的类，都是通过await方法来进行线程的阻塞，当执行线程数达到具体的数量时才会执行释放阻塞队列的方法，都是依赖于AbstractQueuedSynchronizer框架进行实现，存在于java.util.concurrent包下。
二、区别

​	1、但CountDownLatch内部Sync通过实现AbstractQueuedSynchronizer来实现然后通过state关键字来计算具体的线程实现数量，CyclicBarrier是通过ReentrantLock类来实现，然后通过count变量计算线程数量。

​	2、CyclicBarrier可重复使用，通过调用nextGeneration方法实现，CountDownLatch不能够实现。

​	3、CyclicBarrier中await方法调用然后增加count的值来实现，而CountDownLatch是countDown方法实现的线程计数，增加state关键字的数量，不会去阻塞线程。

​	4、CyclicBarrier内部初始化时可以传入runnable实现类，然后进行调用。

三、具体实现

​	CountDownLatch类具体实现

```java
public class CountDownLatch {
    
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        //这里是实现线程计数器的关键，只有线程数达到初始化的时候才会去将挂起的线程重新运行
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        //当线程占用数减至0时才会返回成功，然后才能进行后续的线程执行操作
        protected boolean tryReleaseShared(int releases) {
            // 循环release知道state为0或者设置nextc成功
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

    //初始化时设置state值
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    //等待释放线程
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    //设置等待时间的await函数
    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    //每次countdown将线程占用数减去一
    public void countDown() {
        sync.releaseShared(1);
    }

    //等待线程数
    public long getCount() {
        return sync.getCount();
    }
    public String toString() {
        return super.toString() + "[Count = " + sync.getCount() + "]";
    }
}
```

​	CyclicBarrier具体实现。


```java
public class CyclicBarrier {
    private static class Generation {
        boolean broken = false;
    }
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition trip = lock.newCondition();
    private final int parties;
    private final Runnable barrierCommand;
    private Generation generation = new Generation();    
    private int count;
	//下一个generation
    private void nextGeneration() {
        //释放所有的等待线程
        trip.signalAll();
        //需要的线程数重新初始化
        count = parties;
        //新的generation
        generation = new Generation();
    }

    //中断当前generation
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }

    //等待方法具体实现
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();
            //这里和后面的设置中断状态配合，保证同一线程只能等待一次
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;
            if (index == 0) {  
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    //初始化的任务
                    if (command != null)
                        command.run();
                    ranAction = true;
                    nextGeneration();
                    return 0;
                } finally {
                  //若是等于false却未调用nextGeneration，手动调用breakBarrier。保证不会重复调用
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    //加入等待队列
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // 这里是新的genertion或者broken为true表示当前genertion结束，直接加上						//中断标志，跟前面检测匹配进行处理
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            //释放锁
            lock.unlock();
        }
    }

    //初始化，可以包含任务的初始化
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

    //仅包含最多等待线程的初始化
    public CyclicBarrier(int parties) {
        this(parties, null);
    }

    //获取初始化的等待最大条数
    public int getParties() {
        return parties;
    }

    //线程同步等待塞方法
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); 
        }
    }

    //带时间的等待方法
    public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }

    //当前generation是否中断
    public boolean isBroken() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return generation.broken;
        } finally {
            lock.unlock();
        }
    }

    //开启新的generation
    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier();   // break the current generation
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }

    //等待线程数
    public int getNumberWaiting() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return parties - count;
        } finally {
            lock.unlock();
        }
	}
  }
}
```

​	
