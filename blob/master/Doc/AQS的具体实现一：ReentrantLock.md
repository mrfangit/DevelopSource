##### AQS的具体实现一：ReentrantLock

​	AQS的实现有多种形式，包括ReentrantLock、CountDownLatch、CyclicBarrier、ReentrantLock、Semaphore和ThreadPoolExecutor。ReentrantLock 可重入的独占锁：可重入指同一线程可以多次进入此锁，独占表示同一时刻只能够有一个线程持有该锁。其实现方式又有两种，分为公平和非公平锁。

​	非公平和公平锁：如下图,我们能看到其内部有两个实现类，公平锁与非公平锁。		  	      

![非公平锁和公平锁](非公平锁和公平锁.png)    

​	ReentrantLock中构造方式有两种，无参的直接实现非公平锁，有参的根据参数选择实现。

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

​	我们看下公平锁和非公平锁的具体实现，锁的主要实现是通过实现AQS的tryAcquire方法和自身的lock方法。

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    //加锁，分为两步。先尝试获取锁，获取到将持有锁的线程设置为当前线程
  	//通过AbstractOwnableSynchronizer的setExclusiveOwnerThread实现
    final void lock() {
      	//这里如果获取成功直接将持有线程设置成自己就行，无其它操作
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
          	//加入阻塞队列
            acquire(1);
    }

  
  	//尝试获取锁，上篇博客中锁的不同实现就是这里的差异性实现
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

​	这里是nonfairTryAcquire尝试获取锁的具体实现。

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
  	//获取state
    int c = getState();
    if (c == 0) {
      	//为空的化直接尝试将其设置为acquires（acquires一般为1）
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
      	//不为空的话，若是当前线程就是持有线程。将state加acquires
        int nextc = c + acquires;
        if (nextc < 0) 
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

​	公平方式获取锁，其和非公平方式是一致的。唯一的不同在hasQueuedPredecessors。

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

​	这里我们能看到，若是阻塞队列不为空且第一个等待线程不为当前线程。则返回true，根据这里判断公平锁和非公平锁的区别是若是存在阻塞队列则不进行锁的获取，不存在才获取。

```java
public final boolean hasQueuedPredecessors() {
    Node t = tail; 
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

​	通过代码，我们能够理解，公平锁是指当前阻塞的线程为空才获取线程，非公平所不管是不是空都会去获取。可重入是指持有的锁持有的线程为当前线程，独占锁既是通过当前持有锁的线程来进行判断。

​	我们再来看下关于同步的condition类，可以看到它返回的是一个ConditionObject类，其实现在AQS中。

```java
public Condition newCondition() {
    return sync.newCondition();
}

final ConditionObject newCondition() {
	return new ConditionObject();
}
```

​	先看下await方法。

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
  	//加入等待队列
    Node node = addConditionWaiter();
  	//这里释放资源，将线程状态置为可销毁然后挂起线程
    int savedState = fullyRelease(node);
    int interruptMode = 0;
  	//查看是否在阻塞队列，否的话直接阻塞线程，由于signal导致的释放
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

​	向条件队列添加节点。

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    //判断最后一个元素是否属于条件状态
    if (t != null && t.waitStatus != Node.CONDITION) {
      	//不是，清理不为等待状态的节点
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
  	//创建等待节点
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
  	//加入队列末尾
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

​	循环删除不处于等待状态的元素节点。

```java
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
  	//循环去除不为condition的元素
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```
​	这里是释放资源，通过将线程的state占用状态置为可销毁。

```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
      	//释放资源
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
      	//释放资源失败则将等待状态置为可销毁
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

​	检测等待过程中是否中断过。

```java
//检测是否中断过，这里是acquired方法回中断线程。（添加中断标志）
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
```

​    将资源加入到阻塞队列，加入直接返回true。没加入且线程没释放资源，这时直接返回false。

```java
final boolean transferAfterCancelledWait(Node node) {
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    //
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

​	根据返回值报重新中断，或者抛出异常。

```java
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
```

​	signal方法。

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

​	循环将条件队列首个condition条件的元素释放（加入阻塞队列或者直接取消挂起状态）。

```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

​	singal方法具体处理。

```java
final boolean transferForSignal(Node node) {
    //
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    //加入到等待队列，若是前置节点状态为-1失败直接将当前线程挂起状态取消
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

​	ReentrantLock通过内部的NonfairSync与FairSync来实现线程的加锁和释放锁，内部new一个ConditionObject对象来进行同步。底层是通过AQS来实现，主要通过tryAcquire和tryRelease来判断是否加入到等待队列。
