## HikariCP源码分析

### ConcurrentBag

#### 从连接池中获取连接com.zaxxer.hikari.util.ConcurrentBag#borrow

```java
public T borrow(long timeout, final TimeUnit timeUnit) throws InterruptedException
{
   // Try the thread-local list first
   // 首先查看threadLocal是否有空闲连接
   final List<Object> list = threadList.get();
   for (int i = list.size() - 1; i >= 0; i--) {
      final Object entry = list.remove(i);
      @SuppressWarnings("unchecked")
      final T bagEntry = weakThreadLocals ? ((WeakReference<T>) entry).get() : (T) entry;
      //本地线程的连接可能会被其他线程从共享队列获取到该线程的连接，需要CAS防止重复分配
      if (bagEntry != null && bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
         return bagEntry;
      }
   }
   //从共享队列获取空闲连接
   // Otherwise, scan the shared list ... then poll the  queue
   final int waiting = waiters.incrementAndGet();
   try {
      //本地线程无空闲连接，从共享连接队列获取
      for (T bagEntry : sharedList) {
         //如果共享队列有可用连接，直接返回
         if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
            // If we may have stolen another waiter's connection, request another bag add.
            if (waiting > 1) {
               listener.addBagItem(waiting - 1);
            }
            return bagEntry;
         }
      }
     //创建连接放入到sharedList中
      listener.addBagItem(waiting);
      //共享队列没有连接，则等待
      timeout = timeUnit.toNanos(timeout);
      do {
         final long start = currentTime();
         //从SynchronousQueue获取连接
         final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS);
         if (bagEntry == null || bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
            return bagEntry;
         }
         timeout -= elapsedNanos(start);
      } while (timeout > 10_000);
      return null;
   }
   finally {
      waiters.decrementAndGet();
   }
}
```

#### 归还连接com.zaxxer.hikari.util.ConcurrentBag#requite

```java
public void   requite(final T bagEntry)
{
   bagEntry.setState(STATE_NOT_IN_USE);
	 //有等待获取连接的线程，调用SynchronousQueue的offer方法
   for (int i = 0; waiters.get() > 0; i++) {
      if (bagEntry.getState() != STATE_NOT_IN_USE || handoffQueue.offer(bagEntry)) {
         return;
      }
      else if ((i & 0xff) == 0xff) {
         parkNanos(MICROSECONDS.toNanos(10));
      }
      else {
         yield();
      }
   }
	//存放到当前线程的ThreadLocal中
   final List<Object> threadLocalList = threadList.get();
   if (threadLocalList.size() < 50) {
      threadLocalList.add(weakThreadLocals ? new WeakReference<>(bagEntry) : bagEntry);
   }
}
```

### FastList

#### 获取数据com.zaxxer.hikari.util.FastList

```java
public T get(int index) 
{
	//没有对 index 参数进行越界检查，HiKariCP保证不会越界
   return elementData[index];
}
```

#### 移除数据com.zaxxer.hikari.util.FastList#remove(java.lang.Object)

```java
public boolean remove(Object element)
{
  //将顺序遍历查找优化为逆序查找
   for (int index = size - 1; index >= 0; index--) {
      if (element == elementData[index]) {
         final int numMoved = size - index - 1;
         if (numMoved > 0) {
            System.arraycopy(elementData, index + 1, elementData, index, numMoved);
         }
         elementData[--size] = null;
         return true;
      }
   }
   return false;
}
```

## 