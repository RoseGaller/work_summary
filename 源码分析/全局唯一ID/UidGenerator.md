# UidGenerator源码分析

* [DefaultUidGenerator](#defaultuidgenerator)
  * [初始化](#初始化)
  * [获取UID](#获取uid)
* [CachedUidGenerator](#cacheduidgenerator)
  * [初始化](#初始化-1)
    * [初始化workerId & bitsAllocator](#初始化workerid--bitsallocator)
    * [初始化RingBuffer](#初始化ringbuffer)
    * [填充RingBuffer](#填充ringbuffer)
      * [生成UID](#生成uid)
      * [填充RingBuffer](#填充ringbuffer-1)
  * [获取UID](#获取uid-1)
* [总结](#总结)


# DefaultUidGenerator

Snowflake算法：

​	sign(1bit)  最高位为0，固定1bit符号标识，即生成的UID为正数。

​	delta seconds (28 bits)  代表相对于时间基点"2016-05-20"的增量值，单位:秒，最多可支持约8.7年

​	worker id (22 bits)  机器id，最多可支持约420w次机器启动。内置实现为在启动时由数据库分配。

​	sequence (13 bits)   每秒下的并发序列，13 bits可支持每秒8192个并发。

## 初始化

com.baidu.fsg.uid.impl.DefaultUidGenerator#afterPropertiesSet

```java
public void afterPropertiesSet() throws Exception {
    // 创建BitsAllocator，生成Id
    bitsAllocator = new BitsAllocator(timeBits, workerBits, seqBits);

    //分配workerId
    workerId = workerIdAssigner.assignWorkerId();
    if (workerId > bitsAllocator.getMaxWorkerId()) {
        throw new RuntimeException("Worker id " + workerId + " exceeds the max " + bitsAllocator.getMaxWorkerId());
    }

    LOGGER.info("Initialized bits(1, {}, {}, {}) for workerID:{}", timeBits, workerBits, seqBits, workerId);
}
```

com.baidu.fsg.uid.worker.DisposableWorkerIdAssigner#assignWorkerId

```java
public long assignWorkerId() { //分配workerId
    // build worker node entity
    WorkerNodeEntity workerNodeEntity = buildWorkerNode();

    // add worker node for new (ignore the same IP + PORT)
    workerNodeDAO.addWorkerNode(workerNodeEntity);
    LOGGER.info("Add worker node:" + workerNodeEntity);

    return workerNodeEntity.getId();
}
```

## 获取UID

com.baidu.fsg.uid.impl.DefaultUidGenerator#getUID

```java
public long getUID() throws UidGenerateException {
    try {
        return nextId();
    } catch (Exception e) {
        LOGGER.error("Generate unique id exception. ", e);
        throw new UidGenerateException(e);
    }
}
```

com.baidu.fsg.uid.impl.DefaultUidGenerator#nextId

```java
protected synchronized long nextId() {
    //获取当前时间，单位秒
    long currentSecond = getCurrentSecond();
		//如果当前时间小于上次时间，说明发生了时间回退，抛出异常
    if (currentSecond < lastSecond) {
        long refusedSeconds = lastSecond - currentSecond;
        throw new UidGenerateException("Clock moved backwards. Refusing for %d seconds", refusedSeconds);
    }

    // 时间相同，增加sequence
    if (currentSecond == lastSecond) {
        sequence = (sequence + 1) & bitsAllocator.getMaxSequence();
        //达到sequence的最大值，阻塞直到下一秒
        if (sequence == 0) {
            currentSecond = getNextSecond(lastSecond);
        }
		//时间不同，sequence从0开始递增
    } else {
  	//优化:极端场景下每秒只生成几个id，容易导致数据倾斜.初始值可以设置为一个随机数
        sequence = 0L;
    }
	 
    lastSecond = currentSecond;
		//生成UID
    return bitsAllocator.allocate(currentSecond - epochSeconds, workerId, sequence);
}
```

com.baidu.fsg.uid.impl.DefaultUidGenerator#getCurrentSecond

```java
private long getCurrentSecond() { //获取当前时间
    long currentSecond = TimeUnit.MILLISECONDS.toSeconds(System.currentTimeMillis());
    if (currentSecond - epochSeconds > bitsAllocator.getMaxDeltaSeconds()) { //判断时间是否耗尽
        throw new UidGenerateException("Timestamp bits is exhausted. Refusing UID generate. Now: " + currentSecond);
    }

    return currentSecond;
}
```

com.baidu.fsg.uid.BitsAllocator#allocate

```java
public long allocate(long deltaSeconds, long workerId, long sequence) { //拼接Id
    return (deltaSeconds << timestampShift) | (workerId << workerIdShift) | sequence;
}
```

# CachedUidGenerator

1、**提前生成Id，缩短了请求时间。**

2、**存放于环形数组，可重复使用内存**

## 初始化

### 初始化workerId & bitsAllocator

com.baidu.fsg.uid.impl.CachedUidGenerator#afterPropertiesSet

```java
public void afterPropertiesSet() throws Exception {
    // initialize workerId & bitsAllocator
    super.afterPropertiesSet();
    
    // initialize RingBuffer & RingBufferPaddingExecutor
    this.initRingBuffer();
    LOGGER.info("Initialized RingBuffer successfully.");
}
```

### 初始化RingBuffer

com.baidu.fsg.uid.impl.CachedUidGenerator#initRingBuffer

```java
private void initRingBuffer() {
    // 环形数组的长度（2的次幂）
    int bufferSize = ((int) bitsAllocator.getMaxSequence() + 1) << boostPower;
    //创建RingBuffer，存放UID（paddingFactor:填充因子，默认50，可用UID不足一半，开始填充）
    this.ringBuffer = new RingBuffer(bufferSize, paddingFactor);
    LOGGER.info("Initialized ring buffer size:{}, paddingFactor:{}", bufferSize, paddingFactor);

    //创建RingBufferPaddingExecutor，填充RingBuffer
    boolean usingSchedule = (scheduleInterval != null);
    this.bufferPaddingExecutor = new BufferPaddingExecutor(ringBuffer, this::nextIdsForOneSecond, usingSchedule);
    if (usingSchedule) {
        bufferPaddingExecutor.setScheduleInterval(scheduleInterval);
    }
    
    LOGGER.info("Initialized BufferPaddingExecutor. Using schdule:{}, interval:{}", usingSchedule, scheduleInterval);
    
    //设置拒绝策略
    this.ringBuffer.setBufferPaddingExecutor(bufferPaddingExecutor);
    if (rejectedPutBufferHandler != null) {
        this.ringBuffer.setRejectedPutHandler(rejectedPutBufferHandler);
    }
    if (rejectedTakeBufferHandler != null) {
        this.ringBuffer.setRejectedTakeHandler(rejectedTakeBufferHandler);
    }
    
    // 填充RingBuffer
    bufferPaddingExecutor.paddingBuffer();
    
    //启动填充调度器
    bufferPaddingExecutor.start();
}
```

### 填充RingBuffer

com.baidu.fsg.uid.buffer.BufferPaddingExecutor#paddingBuffer

```java
public void paddingBuffer() {
    LOGGER.info("Ready to padding buffer lastSecond:{}. {}", lastSecond.get(), ringBuffer);

    // is still running
    if (!running.compareAndSet(false, true)) {
        LOGGER.info("Padding buffer is still running. {}", ringBuffer);
        return;
    }

    // fill the rest slots until to catch the cursor
    boolean isFullRingBuffer = false;
    while (!isFullRingBuffer) {
        List<Long> uidList = uidProvider.provide(lastSecond.incrementAndGet());
        for (Long uid : uidList) {
            isFullRingBuffer = !ringBuffer.put(uid);
            if (isFullRingBuffer) {
                break;
            }
        }
    }

    // not running now
    running.compareAndSet(true, false);
    LOGGER.info("End to padding buffer lastSecond:{}. {}", lastSecond.get(), ringBuffer);
}
```

#### 生成UID

com/baidu/fsg/uid/impl/CachedUidGenerator.java

```java
protected List<Long> nextIdsForOneSecond(long currentSecond) {
    // Initialize result list size of (max sequence + 1)
    int listSize = (int) bitsAllocator.getMaxSequence() + 1;
    List<Long> uidList = new ArrayList<>(listSize);

    // 生成当前秒的第一个Id
    long firstSeqUid = bitsAllocator.allocate(currentSecond - epochSeconds, workerId, 0L);
    for (int offset = 0; offset < listSize; offset++) {
        uidList.add(firstSeqUid + offset);
    }

    return uidList;
}
```

#### 填充RingBuffer

com.baidu.fsg.uid.buffer.RingBuffer#put

```java
public synchronized boolean put(long uid) {
    long currentTail = tail.get(); //填充的位置
    long currentCursor = cursor.get(); //消费的位置
		//RingBuffer已满，拒绝填充
    long distance = currentTail - (currentCursor == START_POINT ? 0 : currentCursor);
    if (distance == bufferSize - 1) {
        rejectedPutHandler.rejectPutBuffer(this, uid);
        return false;
    }
    // 1. 通过位运算，获取填充的数组下标
    int nextTailIndex = calSlotIndex(currentTail + 1); //下一个填充的位置
    if (flags[nextTailIndex].get() != CAN_PUT_FLAG) {//是否可以填充
        rejectedPutHandler.rejectPutBuffer(this, uid); //拒绝填充
        return false;
    }

    // 2. put UID in the next slot
    // 3. update next slot' flag to CAN_TAKE_FLAG
    // 4. publish tail with sequence increase by one
    slots[nextTailIndex] = uid; //设置UID
    flags[nextTailIndex].set(CAN_TAKE_FLAG); //设置可以消费的标志
    tail.incrementAndGet(); //填充的位置后移一位

    // The atomicity of operations above, guarantees by 'synchronized'. In another word,
    // the take operation can't consume the UID we just put, until the tail is published(tail.incrementAndGet())
    return true;
}
```

## 获取UID

com.baidu.fsg.uid.impl.CachedUidGenerator#getUID

```java
public long getUID() {
    try {
        return ringBuffer.take();
    } catch (Exception e) {
        LOGGER.error("Generate unique id exception. ", e);
        throw new UidGenerateException(e);
    }
}
```

com.baidu.fsg.uid.buffer.RingBuffer#take

```java
public long take() {
    // spin get next available cursor
    long currentCursor = cursor.get(); //当前消费的位置
    long nextCursor = cursor.updateAndGet(old -> old == tail.get() ? old : old + 1);//修改并获取消费的位置，如果当前消费的位置等于填充的位置，那么currentCursor等于nextCursor，否则currentCursor后移加1
	
    Assert.isTrue(nextCursor >= currentCursor, "Curosr can't move back");

    // 触发异步填充
    long currentTail = tail.get();
    if (currentTail - nextCursor < paddingThreshold) {//剩余可用的UID小于RingBuffer长度的一半
        LOGGER.info("Reach the padding threshold:{}. tail:{}, cursor:{}, rest:{}", paddingThreshold, currentTail,
                nextCursor, currentTail - nextCursor);
        bufferPaddingExecutor.asyncPadding();
    }
		//消费位置等于填充的位置，没有有效的UID可以获取，抛出异常
    if (nextCursor == currentCursor) {
        rejectedTakeHandler.rejectTakeBuffer(this);
    }

    // 1. check next slot flag is CAN_TAKE_FLAG
    int nextCursorIndex = calSlotIndex(nextCursor); //根据nextCursor计算所属的数据下标
    Assert.isTrue(flags[nextCursorIndex].get() == CAN_TAKE_FLAG, "Curosr not in can take status"); //当前的标志必须为可以CAN_TAKE_FLAG，才可以获取UID

    // 2. get UID from next slot
    // 3. set next slot flag as CAN_PUT_FLAG.
    long uid = slots[nextCursorIndex]; //获取UID
    flags[nextCursorIndex].set(CAN_PUT_FLAG); //设置可以填充的标志
    return uid;t
}
```

# References

https://gitee.com/miaoyiwang/uid-generator.git

# 总结

1、对频繁修改的共享变量进行缓存行填充，消除伪共享，避免了对共享数据的修改导致处于同一缓存行上的数据失效,降低缓存的命中率

2、提前生成Id存放于环形数组，由于数组元素在内存中是连续分配的，可最大程度利用CPU cache以提升性能

3、当可用的UID小于数组的长度一半的时候进行异步填充

