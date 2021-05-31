# JDK源码分析

* [ThreadPoolExecutor](#threadpoolexecutor)
  * [提交任务](#提交任务)
  * [执行任务](#执行任务)
  * [优雅关闭](#优雅关闭)
  * [强制关闭](#强制关闭)
  * [拒绝策略](#拒绝策略)
* [SynchronousQueue](#synchronousqueue)
* [LinkedTransferQueue](#linkedtransferqueue)
* [ReentrantLock](#reentrantlock)
  * [NonfairSync](#nonfairsync)
    * [lock实现分析](#lock实现分析)
      * [尝试获取锁](#尝试获取锁)
      * [加入双向链表](#加入双向链表)
      * [阻塞](#阻塞)
    * [unlock实现分析](#unlock实现分析)
      * [释放锁](#释放锁)
      * [唤醒阻塞线程](#唤醒阻塞线程)
  * [FairSync](#fairsync)
    * [获取公平锁](#获取公平锁)
* [CountDownLatch](#countdownlatch)
  * [countDown实现分析](#countdown实现分析)
  * [**await()**实现分析](#await实现分析)
* [ReentrantReadWriteLock](#reentrantreadwritelock)
  * [写锁](#写锁)
    * [非阻塞获取锁](#非阻塞获取锁)
    * [阻塞获取锁](#阻塞获取锁)
    * [释放锁](#释放锁-1)
  * [读锁](#读锁)
    * [非阻塞获取锁](#非阻塞获取锁-1)
    * [阻塞获取锁](#阻塞获取锁-1)


# ThreadPoolExecutor

## 提交任务

java.util.concurrent.ThreadPoolExecutor#execute

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
 
    int c = ctl.get(); // 线程池状态
    if (workerCountOf(c) < corePoolSize) {//当前线程数小于corePoolSize，启动新的线程
        if (addWorker(command, true)) //创建Worker，启动线程，command作为第一个任务运行
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {//当前线程数大于等于corePoolSize，将任务放入队列
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command)) //如果线程池停止，将任务移除，并执行拒绝策略
            reject(command);
        else if (workerCountOf(recheck) == 0) //当前无线程执行任务，开启线程
            addWorker(null, false);
    }
    else if (!addWorker(command, false))//启动新的线程执行任务
        reject(command); //线程数大于maxPoolSize，并且队列已满，调用拒绝策略
}
```

java.util.concurrent.ThreadPoolExecutor#workerCountOf

```java
private static int workerCountOf(int c)  { return c & CAPACITY; } //线程池线程数量
```

java.util.concurrent.ThreadPoolExecutor#addWorker

```java
private boolean addWorker(Runnable firstTask, boolean core) {//创建Worker，启动线程
    retry:
    for (;;) {
        int c = ctl.get(); 
        int rs = runStateOf(c);//线程池状态

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c); //获取线程数量
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize)) //线程数量大于corePoolSize或者大于maximumPoolSize
                return false;
            if (compareAndIncrementWorkerCount(c)) //CAS增加线程池数量
                break retry; //退出循环
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask); //创建Worker
        final Thread t = w.thread; //创建Worker时创建Thread
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock(); //获取线程池的锁
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get()); 

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) { //判断线程池状态
                    if (t.isAlive()) //线程已经启动，抛出异常
                        throw new IllegalThreadStateException();
                    workers.add(w); //维护线程池的线程
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock(); //释放锁
            }
            if (workerAdded) { //添加成功
                t.start(); //启动线程
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

## 执行任务

java.util.concurrent.ThreadPoolExecutor#runWorker

```java
final void runWorker(Worker w) { //不断从队列获取任务执行
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask; //此线程的第一个任务
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) { //线程的第一个任务不为空或者从队列中获取的任务不为空，则执行
            w.lock(); //获取线程锁
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                //任务执行前的钩子方法
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run(); //执行具体的任务
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //任务执行后的钩子方法
                    afterExecute(task, thrown);
                }
            } finally {
                task = null; //将任务置空
                w.completedTasks++; //完成任务数+1
                w.unlock(); //释放锁
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

从队列获取任务

java.util.concurrent.ThreadPoolExecutor#getTask

```java
private Runnable getTask() { //从队列获取任务
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c); //线程池状态

        //如果调用了shutDownNow，直接返回null
       //如果调用了shutDown，并且任务队列为空，返回null
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
				//线程池的线程数量
        int wc = workerCountOf(c);

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            //如果队列为空，就会阻塞
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

## 优雅关闭

不会清空任务队列，会等所有任务执行完并且中断空闲的线程

java.util.concurrent.ThreadPoolExecutor#shutdown

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock(); //获取锁
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN); //设置线程池状态为SHUTDOWN
        interruptIdleWorkers(); //中断空闲线程
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```

java.util.concurrent.ThreadPoolExecutor#interruptIdleWorkers()

```java
private void interruptIdleWorkers() {
    interruptIdleWorkers(false);
}
```

java.util.concurrent.ThreadPoolExecutor#interruptIdleWorkers(boolean)

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock(); //获取线程池的锁
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) { //线程没有被中断并且处于空闲状态。当线程执行任务时会先加锁，处于加锁状态说明线程正在执行任务，否则处于空闲状态，发送中断信号
                try {
                    t.interrupt(); //中断线程
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock(); //释放线程锁
                }
            }
            if (onlyOne) //是否只中断一个线程
                break;
        }
    } finally {
        mainLock.unlock(); //释放线程池的锁
    }
}
```

## 强制关闭

清空任务队列并且中断所有线程

java.util.concurrent.ThreadPoolExecutor#shutdownNow

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);//设置线程池状态为STOP
        interruptWorkers(); //中断所有线程
        tasks = drainQueue(); //启动队列
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

中断所有线程

java.util.concurrent.ThreadPoolExecutor#interruptWorkers

```java
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}
```

java.util.concurrent.ThreadPoolExecutor.Worker#interruptIfStarted

```java
void interruptIfStarted() {
    Thread t;
    if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
        try {
            t.interrupt();
        } catch (SecurityException ignore) {
        }
    }
}
```

清空任务队列

java.util.concurrent.ThreadPoolExecutor#drainQueue

```java
private List<Runnable> drainQueue() {
    BlockingQueue<Runnable> q = workQueue;
    ArrayList<Runnable> taskList = new ArrayList<Runnable>();
    q.drainTo(taskList); //把任务队列中的任务放入taskList
    if (!q.isEmpty()) {
        for (Runnable r : q.toArray(new Runnable[0])) { //一个一个的从任务队列移除，放入taskList
            if (q.remove(r))
                taskList.add(r);
        }
    }
    return taskList;
}
```

## 拒绝策略

AbortPolicy：线程池抛异常

java.util.concurrent.ThreadPoolExecutor.AbortPolicy#rejectedExecution

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    throw new RejectedExecutionException("Task " + r.toString() +
                                         " rejected from " +
                                         e.toString());
}
```

CallerRunsPolicy：调用者直接在自己的线程里执行，线程池不处理

java.util.concurrent.ThreadPoolExecutor.CallerRunsPolicy#rejectedExecution

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        r.run();
    }
}
```

DiscardOldestPolicy：删除队列中最早的任务，将当前任务入队列

java.util.concurrent.ThreadPoolExecutor.DiscardOldestPolicy#rejectedExecution

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        e.getQueue().poll();
        e.execute(r);
    }
}
```

DiscardPolicy：线程池直接丢掉任务

java.util.concurrent.ThreadPoolExecutor.DiscardPolicy#rejectedExecution

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
}
```

# SynchronousQueue

java.util.concurrent.SynchronousQueue#offer(E)

```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    return transferer.transfer(e, true, 0) != null;
}
```

java.util.concurrent.SynchronousQueue.TransferQueue#transfer

```java
E transfer(E e, boolean timed, long nanos) {

    QNode s = null; // constructed/reused as needed
    boolean isData = (e != null); //true:生产数据 false:消费数据

    for (;;) {
        QNode t = tail;
        QNode h = head;
        if (t == null || h == null)         // saw uninitialized value
            continue;                       // spin
				//队列中只存放put请求或者take请求，只存在其中一种请求
        if (h == t || t.isData == isData) { // 队列为空或者当前请求与队列中的请求类型一致
            QNode tn = t.next;
            if (t != tail)                  // 有新的请求入队
                continue;
            if (tn != null) {               // lagging tail
                advanceTail(t, tn);
                continue;
            }
            if (timed && nanos <= 0)        // 不阻塞，直接返回
                return null;
            if (s == null)
                s = new QNode(e, isData); //创建QNode
            if (!t.casNext(null, s))        // CAS修改tail的next引用
                continue;
						
            advanceTail(t, s);              // 修改tail引用，将s设置为tail
            Object x = awaitFulfill(s, e, timed, nanos); //先自旋再阻塞。匹配成功或者中断都会唤醒
            if (x == s) {  // 如果Qnode的item等于Qnode，表明已被删除
                clean(t, s);	//清除已被删除的Qnode
                return null;
            }

            if (!s.isOffList()) {           // not already unlinked
                advanceHead(t, s);          // unlink if head
                if (x != null)              // and forget fields
                    s.item = s;
                s.waiter = null;
            }
            return (x != null) ? (E)x : e;

        } else {                            // complementary-mode
            QNode m = h.next;               // node to fulfill
            if (t != tail || m == null || h != head) //有数据放入队列中
                continue;                   // inconsistent read

            Object x = m.item; //获取数据
            if (isData == (x != null) ||    // m already fulfilled
                x == m ||                   // item和node相同，表明已经被删除
                !m.casItem(x, e)) {         // CAS修改item，如果e是null，表明当前时take请求
                advanceHead(h, m);          // dequeue and retry
                continue;
            }

            advanceHead(h, m);              //成功配对
            LockSupport.unpark(m.waiter); //唤醒阻塞的线程（可能处于自选状态或者阻塞状态）
            return (x != null) ? (E)x : e;
        }
    }
}
```

java.util.concurrent.SynchronousQueue.TransferQueue#awaitFulfill

```java
Object awaitFulfill(QNode s, E e, boolean timed, long nanos) { //先自旋再阻塞
    /* Same idea as TransferStack.awaitFulfill */
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    int spins = ((head.next == s) ?
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0); //自旋次数
    for (;;) {
        if (w.isInterrupted()) //线程已被中断
            s.tryCancel(e); //将Qnode的item设置为Qnode，item与Qnode相同表明已被删除
        Object x = s.item; //当前值，如果queue中存放的是put请求，x不为空，如果是take请求，x为空
        if (x != e) //如果x不等于e表明其他线程对其进行了修改
            return x;
        if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) { 
                s.tryCancel(e);//阻塞超时，删除
                continue;
            }
        }
        if (spins > 0) //自旋
            --spins;
        else if (s.waiter == null)
            s.waiter = w;
        else if (!timed)
            LockSupport.park(this); //阻塞
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos); //带时间的阻塞
    }
}
```

# LinkedTransferQueue

put、add、offer，如果链表为空或者与链表中的请求类型一致，直接加入到链表。否则进行匹配，唤醒阻塞的线程。

take、timed poll，如果链表为空或者与链表中的请求类型一致，直接加入到链表并进入阻塞状态。否则进行匹配，匹配成功，直接返回。匹配失败，加入链表进行阻塞。

```java
private static final int NOW   = 0; // for untimed poll, tryTransfer
private static final int ASYNC = 1; // for offer, put, add
private static final int SYNC  = 2; // for transfer, take
private static final int TIMED = 3; // for timed poll, tryTransfer
```

java.util.concurrent.LinkedTransferQueue#xfer

```java
private E xfer(E e, boolean haveData, int how, long nanos) { //havaData为true，表明put请求
    if (haveData && (e == null))
        throw new NullPointerException();
    Node s = null;                        // the node to append, if needed

    retry:
    for (;;) {                            // restart on append race

        for (Node h = head, p = h; p != null;) { // find & match first node
            boolean isData = p.isData;
            Object item = p.item;
            if (item != p && (item != null) == isData) { // 判断节点的可用性，没有被删除，isData与item的值合法，isData为true时，item不能为空。为false时，item为空
                if (isData == haveData)   // 存放的操作类型与当前的操作类型相同，直接入队
                    break;
                if (p.casItem(item, e)) { // 操作类型不一致，匹配成功
                    for (Node q = p; q != h;) { //修改head引用
                        Node n = q.next;  // update by 2 unless singleton
                        if (head == h && casHead(h, n == null ? q : n)) {
                            h.forgetNext();
                            break;
                        }                 // advance and retry
                        if ((h = head)   == null ||
                            (q = h.next) == null || !q.isMatched())
                            break;        // unless slack < 2
                    }
                    LockSupport.unpark(p.waiter); //唤醒阻塞线程。当前
                    return LinkedTransferQueue.<E>cast(item);
                }
            }
            Node n = p.next;
            p = (p != n) ? n : (h = head); // Use head if p offlist
        }

        if (how != NOW) {                 // No matches available
            if (s == null)
                s = new Node(e, haveData); //创建Node，追加到队列尾部
            Node pred = tryAppend(s, haveData); //返回前一个node
            if (pred == null) //说明操作类型不一致
                continue retry;           // lost race vs opposite mode
            if (how != ASYNC) //不是put/add/offer,进行阻塞状态
                return awaitMatch(s, pred, e, (how == TIMED), nanos);
        }
        return e; // not waiting
    }
}
```

java.util.concurrent.LinkedTransferQueue#awaitMatch

```java
private E awaitMatch(Node s, Node pred, E e, boolean timed, long nanos) {//spins/yields/blocks
    final long deadline = timed ? System.nanoTime() + nanos : 0L; //请求过期时间
    Thread w = Thread.currentThread(); //当前线程
    int spins = -1; // initialized after first item and cancel checks
    ThreadLocalRandom randomYields = null; // bound if needed

    for (;;) {
        Object item = s.item;
        if (item != e) {                  // item被修改，匹配成功
            // assert item != s;
            s.forgetContents();           //item设置为空，waiter设置为空
            return LinkedTransferQueue.<E>cast(item);
        }
        if ((w.isInterrupted() || (timed && nanos <= 0)) &&
                s.casItem(e, s)) {    //被中断或者超时
            unsplice(pred, s);
            return e;
        }

        if (spins < 0) {                  // 初始化自旋的次数
            if ((spins = spinsFor(pred, s.isData)) > 0) 
                randomYields = ThreadLocalRandom.current();
        }
        else if (spins > 0) {             // 开始自旋
            --spins;
            if (randomYields.nextInt(CHAINED_SPINS) == 0)
                Thread.yield();           // occasionally yield
        }
        else if (s.waiter == null) { //自旋结束，设置waiter
            s.waiter = w;                 // request unpark then recheck
        }
        else if (timed) { //带有超时时间
            nanos = deadline - System.nanoTime(); //阻塞时间
            if (nanos > 0L)
                LockSupport.parkNanos(this, nanos); //阻塞指定的时间
        }
        else {
            LockSupport.park(this); //阻塞
        }
    }
}
```

java.util.concurrent.LinkedTransferQueue#spinsFor

```java
private static int spinsFor(Node pred, boolean haveData) {
    if (MP && pred != null) { //多核，并且前驱节点bu'wei'kong
        if (pred.isData != haveData)      // phase change
            return FRONT_SPINS + CHAINED_SPINS;
        if (pred.isMatched())             // probably at front
            return FRONT_SPINS;
        if (pred.waiter == null)          // pred apparently spinning
            return CHAINED_SPINS;
    }
    return 0;
}
```

# ReentrantLock

## NonfairSync

### lock实现分析

java.util.concurrent.locks.ReentrantLock.NonfairSync#lock

```java
final void lock() {
    if (compareAndSetState(0, 1)) //当前state=0，并且成功修改为1 
        setExclusiveOwnerThread(Thread.currentThread()); //设置获的锁的线程为当前线程
    else
        acquire(1);
}
```

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

#### 尝试获取锁

java.util.concurrent.locks.ReentrantLock.NonfairSync#tryAcquire

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

```java
final boolean nonfairTryAcquire(int acquires) {//不考虑队列中有没有其他线程，非公平
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) { //如果state=0，同时自旋成功，直接将当前线程设置为排他线程
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) { //如果state为0，并且排他线程就是当前线程，则直接设置state的值
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false; //获取锁失败
}
```

#### 加入双向链表

java.util.concurrent.locks.AbstractQueuedSynchronizer#addWaiter

```java
private Node addWaiter(Node mode) { //为当前线程生成一个Node，然后把Node放入双向链表的尾部
  //创建排他节点，设置当前线程  
  Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) { //尾节点不为空
        node.prev = pred; //指向前一个节点
        if (compareAndSetTail(pred, node)) { //将新节点设置为尾节点
            pred.next = node;  //前一个节点指向新的尾节点
            return node;
        }
    }
    //尾节点为空或者设置尾节点失败
    enq(node);
    return node;
}
```



```java
private Node enq(final Node node) { //自旋设置尾节点或者对双向队列进行初始化
    for (;;) {
        Node t = tail;
        if (t == null) { //初始化
            if (compareAndSetHead(new Node()))
                tail = head;
        } else { //自旋设置尾节点
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

#### 阻塞

java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireQueued

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
      	//中断标志
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            //如果前一个节点是头结点，则尝试获取锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //获取锁失败是否需要阻塞，阻塞唤醒之后检查中断
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true; //被中断唤醒
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

java.util.concurrent.locks.AbstractQueuedSynchronizer#shouldParkAfterFailedAcquire

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus; //初始值为0
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) { //前继节点已被删除
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
      //修改前继节点的等待状态为SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

### unlock实现分析

java.util.concurrent.locks.ReentrantLock#unlock

```java
public void unlock() {
    sync.release(1);
}
```

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) { //释放锁
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h); //唤醒阻塞线程
        return true;
    }
    return false;
}
```

#### 释放锁

java.util.concurrent.locks.ReentrantLock.Sync#tryRelease

```java
protected final boolean tryRelease(int releases) {
    //计算当前线程释放锁后的state值。
    int c = getState() - releases;
    //如果当前线程不是排他线程，则抛异常，因为只有获取锁的线程才可以进行释放 锁的操作。
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    //释放锁
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    //设置state的值
    setState(c);
    return free;
}
```

#### 唤醒阻塞线程

java.util.concurrent.locks.AbstractQueuedSynchronizer#unparkSuccessor

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) { //后继节点为空或者已被删除
        s = null;
       //从后往前过滤已被删除的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
  	//唤醒后继节点
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

## FairSync

### 获取公平锁

java.util.concurrent.locks.ReentrantLock.FairSync#lockh

```java
final void lock() {
    acquire(1);
}
```

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) { //如果state为0，并且队列中无线程等待，并且自旋成功，则设置当前线程为排他线程
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {//如果当前线程就是排他线程，直接设置state的值
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false; //获取锁失败
}
```

# CountDownLatch

## countDown实现分析

java.util.concurrent.CountDownLatch#countDown

```java
public void countDown() {
    sync.releaseShared(1);
}
```

java.util.concurrent.locks.AbstractQueuedSynchronizer#releaseShared

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) { //state=0，返回true
        doReleaseShared(); //唤醒阻塞的线程
        return true;
    }
    return false;
}
```

java.util.concurrent.CountDownLatch.Sync#tryReleaseShared

```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0) 
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc)) //CAS修改state
            return nextc == 0; //等于0，返回true
    }
}
```

## **await()**实现分析

java.util.concurrent.CountDownLatch#await()

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)//小于0，说明state>0,需要等待
        doAcquireSharedInterruptibly(arg);
}
```

```java
protected int tryAcquireShared(int acquires) { //如果state=0,不需要等待
    return (getState() == 0) ? 1 : -1;
}
```

java.util.concurrent.locks.AbstractQueuedSynchronizer#doAcquireSharedInterruptibly

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

# ReentrantReadWriteLock

## 写锁

### 非阻塞获取锁

java.util.concurrent.locks.ReentrantReadWriteLock.WriteLock#tryLock()

```java
public boolean tryLock( ) {
    return sync.tryWriteLock();//获取到锁，返回true，否则返回false，不会进行阻塞
}
```

java.util.concurrent.locks.ReentrantReadWriteLock.Sync#tryWriteLock

```java
final boolean tryWriteLock() {
    Thread current = Thread.currentThread(); //获取当前线程
    int c = getState(); 
    if (c != 0) {
        int w = exclusiveCount(c); //获取独占锁的个数
        if (w == 0 || current != getExclusiveOwnerThread()) //独占锁的个数等于0或者获取锁的不是当前线程
            return false;
        if (w == MAX_COUNT)//独占锁的个数达到上限
            throw new Error("Maximum lock count exceeded");
    }
    if (!compareAndSetState(c, c + 1)) //CAS修改state成功，表明获取锁
        return false;
    setExclusiveOwnerThread(current);//设置获取锁的线程
    return true;
}
```

### 阻塞获取锁

实现过程与ReentrantLock获取锁的过程相同

java.util.concurrent.locks.ReentrantReadWriteLock.WriteLock#lock

```java
public void lock() {
    sync.acquire(1);
}
```

java.util.concurrent.locks.AbstractQueuedSynchronizer#acquire

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

### 释放锁

java.util.concurrent.locks.ReentrantReadWriteLock.WriteLock#unlock

```java
public void unlock() {
    sync.release(1);
}
```

java.util.concurrent.locks.AbstractQueuedSynchronizer#release

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {//独占锁个数为0，返回true
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

java.util.concurrent.locks.ReentrantReadWriteLock.Sync#tryRelease

```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively()) //只有获取锁的线程与当前释放锁的线程相同时，才能释放锁，否则抛出异常
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0; //获取独占锁个数
    if (free)
        setExclusiveOwnerThread(null); //独占锁个数为0，返回true
    setState(nextc);
    return free;
}
```

## 读锁

### 非阻塞获取锁

java.util.concurrent.locks.ReentrantReadWriteLock.ReadLock#tryLock()

```java
public boolean tryLock() {
    return sync.tryReadLock();
}
```

java.util.concurrent.locks.ReentrantReadWriteLock.Sync#tryReadLock

```java
final boolean tryReadLock() {
    Thread current = Thread.currentThread();
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current) //独占锁个数不为0并且不是当前线程得到了独占锁，返回false
            return false;
        int r = sharedCount(c);//获取读锁个数
        if (r == MAX_COUNT) //达到上限
            throw new Error("Maximum lock count exceeded");
       //state高位代表读锁个数，低位代表独占锁个数
        if (compareAndSetState(c, c + SHARED_UNIT)) {//CAS修改读锁个数
            if (r == 0) { 
                firstReader = current; //第一个获取读锁的线程
                firstReaderHoldCount = 1; //持有读锁的个数
            } else if (firstReader == current) {//读重入
                firstReaderHoldCount++;
            } else { //其他线程获取读锁，readHolds维护每个线程获取读锁的个数
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
            }
            return true;
        }
    }
}
```

### 阻塞获取锁

java.util.concurrent.locks.ReentrantReadWriteLock.ReadLock#lock

```java
public void lock() {
    sync.acquireShared(1);
}
```

java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireShared

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

java.util.concurrent.locks.ReentrantReadWriteLock.Sync#tryAcquireShared

```java
protected final int tryAcquireShared(int unused) {
    /*
     * Walkthrough:
     * 1. If write lock held by another thread, fail.
     * 2. Otherwise, this thread is eligible for
     *    lock wrt state, so ask if it should block
     *    because of queue policy. If not, try
     *    to grant by CASing state and updating count.
     *    Note that step does not check for reentrant
     *    acquires, which is postponed to full version
     *    to avoid having to check hold count in
     *    the more typical non-reentrant case.
     * 3. If step 2 fails either because thread
     *    apparently not eligible or CAS fails or count
     *    saturated, chain to version with full retry loop.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)//独占锁个数不为0并且不是当前线程得到了独占锁，进行阻塞
        return -1;
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
```
