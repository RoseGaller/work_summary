# Disruptor源码分析

* [概述](#概述)
* [WorkerPool](#workerpool)
  * [创建WorkerPool](#创建workerpool)
  * [创建RingBuffer](#创建ringbuffer)
  * [填充环形数组](#填充环形数组)
  * [发布事件](#发布事件)
  * [启动消费者](#启动消费者)
  * [消费者处理逻辑](#消费者处理逻辑)
* [消费者等待策略](#消费者等待策略)
  * [YieldingWaitStrategy](#yieldingwaitstrategy)
  * [SleepingWaitStrategy](#sleepingwaitstrategy)
  * [TimeoutBlockingWaitStrategy](#timeoutblockingwaitstrategy)
  * [PhasedBackoffWaitStrategy](#phasedbackoffwaitstrategy)
      * [LockBlockingStrategy](#lockblockingstrategy)
    * [SleepBlockingStrategy](#sleepblockingstrategy)
  * [BlockingWaitStrategy](#blockingwaitstrategy)
  * [BusySpinWaitStrategy](#busyspinwaitstrategy)


# 概述

1、环形数组，数组长度必须是的2的次方，方便位运算，计算数组下标。初始化时会对数组填充对象，对象可以重复使用，避免了频繁的创建销毁。数组会被分配连续内存存储对象，可以很好利用空间局部性原理。

2、缓存行填充，避免了伪共享。

3、无锁，通过 CAS ，去进行序号的自增和对比。CAS是一个 CPU 硬件支持的机器指令，没有上下文切换、没有操作系统锁。但是 CAS 忙等待（Busy-Wait）的方式，会使得CPU 始终满负荷运转。

# WorkerPool

## 创建WorkerPool

com.lmax.disruptor.WorkerPool#WorkerPool(EventFactory<T>, ExceptionHandler, WorkHandler<T>...)

```java
public WorkerPool(final EventFactory<T> eventFactory, //事件生产工厂
                  final ExceptionHandler exceptionHandler, //异常处理器
                  final WorkHandler<T>... workHandlers) //事件处理器
{//多生产者多消费者模式
    ringBuffer = RingBuffer.createMultiProducer(eventFactory, 1024, new BlockingWaitStrategy());//创建环形数组
    final SequenceBarrier barrier = ringBuffer.newBarrier(); //创建屏障
    final int numWorkers = workHandlers.length; //消费者数量
    //将workhandler封装成WorkProcessor
    workProcessors = new WorkProcessor[numWorkers]; 
    for (int i = 0; i < numWorkers; i++)
    {
        workProcessors[i] = new WorkProcessor<T>(ringBuffer,
                                                 barrier,
                                                 workHandlers[i],
                                                 exceptionHandler,
                                                 workSequence);
    }
    //各个消费者的sequence
    ringBuffer.addGatingSequences(getWorkerSequences());
}
```

## 创建RingBuffer

com.lmax.disruptor.RingBuffer#createMultiProducer(EventFactory<E>, int, WaitStrategy)

```java
public static <E> RingBuffer<E> createMultiProducer(EventFactory<E> factory, 
                                                    int             bufferSize, 
                                                    WaitStrategy    waitStrategy)
{		
    //创建多生产者
    MultiProducerSequencer sequencer = new MultiProducerSequencer(bufferSize, waitStrategy);
    //创建RingBuffer
  	RingBuffer<E> ringBuffer = new RingBuffer<E>(factory, sequencer);
    return ringBuffer;
}
```

初始化RingBuffer

com.lmax.disruptor.RingBuffer#RingBuffer

```java
private RingBuffer(EventFactory<E> eventFactory, 
                   Sequencer       sequencer)
{
    this.sequencer    = sequencer;
    this.bufferSize   = sequencer.getBufferSize();
    if (bufferSize < 1)
    {
        throw new IllegalArgumentException("bufferSize must not be less than 1");
    }
    //数组的大小必须是2的次幂
    if (Integer.bitCount(bufferSize) != 1)
    {
        throw new IllegalArgumentException("bufferSize must be a power of 2");
    }
    this.indexMask = bufferSize - 1;
    //创建数组
    this.entries   = new Object[sequencer.getBufferSize()];
    //填充数组
    fill(eventFactory);
}
```

## 填充环形数组

```java
private void fill(EventFactory<E> eventFactory)//填充数组
{
    for (int i = 0; i < entries.length; i++)
    {
        entries[i] = eventFactory.newInstance();//eventFactory负责创建事件
    }
}
```

## 发布事件

```java
long sequence = ringBuffer.next(); //获取下一个sequence
* try {
*     Event e = ringBuffer.getPreallocated(sequence);//从数组中后获取对应sequence的事件
*     // Do some work with the event. //填充事件该有的属性信息
* } finally {
*     ringBuffer.publish(sequence); //发布事件，该sequence可以被消费者消费
* }
```

## 启动消费者

com.lmax.disruptor.WorkerPool#start

```java
public RingBuffer<T> start(final Executor executor)
{
    if (!started.compareAndSet(false, true))
    {
        throw new IllegalStateException("WorkerPool has already been started and cannot be restarted until halted.");
    }
		//生产消息的位置
    final long cursor = ringBuffer.getCursor();
    workSequence.set(cursor);

    for (WorkProcessor<?> processor : workProcessors)
    {	
        //每个WorkProcessor都有独立的Sequence，标志消费的位置
        processor.getSequence().set(cursor);
        executor.execute(processor);
    }
    return ringBuffer;
}
```

## 消费者处理逻辑

com.lmax.disruptor.WorkProcessor#run

```java
public void run()
{
    if (!running.compareAndSet(false, true))
    {
        throw new IllegalStateException("Thread is already running");
    }
    sequenceBarrier.clearAlert();
    notifyStart();
    boolean processedSequence = true;
    long nextSequence = sequence.get();
    T event = null;
    while (true)
    {
        try
        {
            if (processedSequence)
            {
                processedSequence = false;
                //workSequence被多个消费者共享，获取消费的位置
                nextSequence = workSequence.incrementAndGet();
                sequence.set(nextSequence - 1L);
            }
            //阻塞直到此位置已有消息生产，应对消费者消费速度大于生产者生产速度
            sequenceBarrier.waitFor(nextSequence);
            //获取事件
            event = ringBuffer.getPublished(nextSequence);
            //处理事件
            workHandler.onEvent(event);
            processedSequence = true;
        }
        catch (final AlertException ex)
        {
            if (!running.get())
            {
                break;
            }
        }
        catch (final Throwable ex)
        {
            exceptionHandler.handleEventException(ex, nextSequence, event);
            processedSequence = true;
        }
    }
    notifyShutdown();
    running.set(false);
}
```

# 消费者等待策略

## YieldingWaitStrategy

这种策略是性能和CPU资源之间的一种很好的折衷，不会引起显著的延迟峰值

先自旋，然后调用Thread的yield，让出CPU，让其他线程获得调度

com.lmax.disruptor.YieldingWaitStrategy#waitFor

```java
public long waitFor(final long sequence, Sequence cursor, final Sequence dependentSequence, final SequenceBarrier barrier)
    throws AlertException, InterruptedException
{
    long availableSequence;
    int counter = SPIN_TRIES; //默认100
    while ((availableSequence = dependentSequence.get()) < sequence)
    {
        counter = applyWaitMethod(barrier, counter);
    }
    return availableSequence;
}
```

```java
private int applyWaitMethod(final SequenceBarrier barrier, int counter)
    throws AlertException
{
    barrier.checkAlert();
    if (0 == counter)
    {
        Thread.yield();
    }
    else
    {
        --counter;
    }
    return counter;
}
```

## SleepingWaitStrategy

```java
public long waitFor(final long sequence, Sequence cursor, final Sequence dependentSequence, final SequenceBarrier barrier)
    throws AlertException, InterruptedException
{
    long availableSequence;
    int counter = RETRIES;
    while ((availableSequence = dependentSequence.get()) < sequence)
    {
        counter = applyWaitMethod(barrier, counter);
    }
    return availableSequence;
}
```

```java
private int applyWaitMethod(final SequenceBarrier barrier, int counter)
    throws AlertException
{
    barrier.checkAlert();
    if (counter > 100) //初始值200；大于100,自旋
    {
        --counter;
    }
    else if (counter > 0) //0<counter<100，yield让出CPU
    {
        --counter;
        Thread.yield();
    }
    else //休眠
    {
        LockSupport.parkNanos(1L);
    }
    return counter;
}
```

## TimeoutBlockingWaitStrategy

com.lmax.disruptor.TimeoutBlockingWaitStrategy#waitFor

```java
public long waitFor(final long sequence, 
                    final Sequence cursorSequence, 
                    final Sequence dependentSequence, 
                    final SequenceBarrier barrier)
    throws AlertException, InterruptedException
{
    long nanos = timeoutInNanos;
    long availableSequence;
    //阻塞，直到生产者在sequence处生产消息
    if ((availableSequence = cursorSequence.get()) < sequence)
    {
        lock.lock();
        try
        {
            while ((availableSequence = cursorSequence.get()) < sequence)
            {
                barrier.checkAlert();
                //阻塞
                nanos = processorNotifyCondition.awaitNanos(nanos);
                if (nanos <= 0) //超时
                {
                    return availableSequence;
                }
            }
        }
        finally
        {
            lock.unlock();
        }
    }
  	//阻塞，直到前面的消费者消费的sequence超过当前请求的sequence（消费者有前后关系，前面的消费者消费了此sequence处的消息之后，后面的消费者才能消费此sequence处的消息）
    //或者直到生产者在此sequence生产了消息（此消费者前面无消费者）
    while ((availableSequence = dependentSequence.get()) < sequence)
    {
        barrier.checkAlert();
    }

    return availableSequence;
}
```

##  PhasedBackoffWaitStrategy

分阶段等待策略，自旋、让出cpu、阻塞

```java
public long waitFor(long sequence, Sequence cursor, Sequence dependentSequence, SequenceBarrier barrier)
    throws AlertException, InterruptedException
{
    long availableSequence;
    long startTime = 0;
    int counter = SPIN_TRIES;

    do
    {
        if ((availableSequence = dependentSequence.get()) >= sequence)
        {
            return availableSequence;
        }
        if (0 == --counter) //默认10000，先自旋
        {
            if (0 == startTime)
            {
                startTime = System.nanoTime();
            }
            else
            {
                long timeDelta = System.nanoTime() - startTime;
                if (timeDelta > yieldTimeoutNanos) //阻塞
                {
                    return lockingStrategy.waitOnLock(sequence, cursor, dependentSequence, barrier);
                }
                else if (timeDelta > spinTimeoutNanos) //让出cpu
                {
                    Thread.yield();
                }
            }
            counter = SPIN_TRIES;
        }
    }
    while (true);
}
```

#### LockBlockingStrategy

```java
public long waitOnLock(long sequence,
                       Sequence cursorSequence,
                       Sequence dependentSequence,
                       SequenceBarrier barrier) throws AlertException, InterruptedException
{
    long availableSequence;
    lock.lock();
    try
    {
        ++numWaiters;
        while ((availableSequence = cursorSequence.get()) < sequence)
        {
            barrier.checkAlert();
            processorNotifyCondition.await(1, TimeUnit.MILLISECONDS);
        }
    }
    finally
    {
        --numWaiters;
        lock.unlock();
    }
    while ((availableSequence = dependentSequence.get()) < sequence)
    {
        barrier.checkAlert();
    }
    return availableSequence;
}
```

### SleepBlockingStrategy

```java
public long waitOnLock(final long sequence,
                        Sequence cursorSequence,
                        final Sequence dependentSequence, final SequenceBarrier barrier)
        throws AlertException, InterruptedException
{
    long availableSequence;
    while ((availableSequence = dependentSequence.get()) < sequence)
    {
        LockSupport.parkNanos(1); //休眠
    }
    return availableSequence;
}
```

## BlockingWaitStrategy

当吞吐量和低延迟不像CPU资源那么重要时，可以使用这个策略

```
lock/condition实现
```

```java
public long waitFor(long sequence, Sequence cursorSequence, Sequence dependentSequence, SequenceBarrier barrier)
    throws AlertException, InterruptedException
{
    long availableSequence;
    if ((availableSequence = cursorSequence.get()) < sequence)
    {
        lock.lock();
        try
        {
            while ((availableSequence = cursorSequence.get()) < sequence)
            {
                barrier.checkAlert();
                processorNotifyCondition.await();
            }
        }
        finally
        {
            lock.unlock();
        }
    }

    while ((availableSequence = dependentSequence.get()) < sequence)
    {
        barrier.checkAlert();
    }

    return availableSequence;
}
```

## BusySpinWaitStrategy

这个策略将使用CPU资源来避免可能引入延迟抖动的系统调用。最好是当线程可以绑定到特定的CPU内核时使用。

```java
public long waitFor(final long sequence, Sequence cursor, final Sequence dependentSequence, final SequenceBarrier barrier)
    throws AlertException, InterruptedException
{
    long availableSequence;

    while ((availableSequence = dependentSequence.get()) < sequence)
    {
        barrier.checkAlert();
    }

    return availableSequence;
}
```
