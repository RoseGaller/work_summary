

com.alibaba.csp.sentinel.SphU#entry(java.lang.String)

```java
public static Entry entry(Method method, EntryType type) throws BlockException {
  //此方法进行资源初始化,限流 熔断等操作
  return Env.sph.entry(method, type, 1, OBJECTS0);
}
```

```java
public class Env {
    public static final Sph sph = new CtSph();
    static {
        InitExecutor.doInit();
    }
}
```

com.alibaba.csp.sentinel.init.InitExecutor#doInit

```java
public static void doInit() {
	  //判断是否是第一次初始化,不是则直接返回
    if (!initialized.compareAndSet(false, true)) {
        return;
    }
    try {
      //加载"META-INF/services/"目录下所配置的所有实现了InitFunc接口的类
        ServiceLoader<InitFunc> loader = ServiceLoaderUtil.getServiceLoader(InitFunc.class);
        List<OrderWrapper> initList = new ArrayList<OrderWrapper>();
        for (InitFunc initFunc : loader) {
            RecordLog.info("[InitExecutor] Found init func: " + initFunc.getClass().getCanonicalName());
          	//将加载完的所有实现类排序
            insertSorted(initList, initFunc);
        }
        for (OrderWrapper w : initList) {
          	//执行每个InitFunc实现类的init()方法,init()方法又会去加载其它所需资源
            w.func.init();
            RecordLog.info(String.format("[InitExecutor] Executing %s with order %d",
                w.func.getClass().getCanonicalName(), w.order));
        }
    } catch (Exception ex) {
        RecordLog.warn("[InitExecutor] WARN: Initialization failed", ex);
        ex.printStackTrace();
    } catch (Error error) {
        RecordLog.warn("[InitExecutor] ERROR: Initialization failed with fatal error", error);
        error.printStackTrace();
    }
}
```

com.alibaba.csp.sentinel.CtSph#entry(java.lang.String, com.alibaba.csp.sentinel.EntryType, int, java.lang.Object...)

```java
public Entry entry(String name, EntryType type, int count, Object... args) throws BlockException {
    StringResourceWrapper resource = new StringResourceWrapper(name, type);
    return entry(resource, count, args);
}
```

com.alibaba.csp.sentinel.CtSph#entry(com.alibaba.csp.sentinel.slotchain.ResourceWrapper, int, java.lang.Object...)

```java
public Entry entry(ResourceWrapper resourceWrapper, int count, Object... args) throws BlockException {
    return entryWithPriority(resourceWrapper, count, false, args);
}
```

com.alibaba.csp.sentinel.CtSph#entryWithPriority(com.alibaba.csp.sentinel.slotchain.ResourceWrapper, int, boolean, java.lang.Object...)

```java
private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count, boolean prioritized, Object... args)
    throws BlockException {
    Context context = ContextUtil.getContext();
    if (context instanceof NullContext) { //context数量超过上限
        return new CtEntry(resourceWrapper, null, context);
    }

    if (context == null) { //创建Context
        context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
    }

    if (!Constants.ON) { //开关是否打开
        return new CtEntry(resourceWrapper, null, context);
    }

    ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper); //构建slot处理链

    if (chain == null) {
        return new CtEntry(resourceWrapper, null, context);
    }

    Entry e = new CtEntry(resourceWrapper, chain, context);
    try {
        chain.entry(context, resourceWrapper, null, count, prioritized, args);//执行slot
    } catch (BlockException e1) { //触发流控，抛出BlockException
        e.exit(count, args);
        throw e1;
    } catch (Throwable e1) {
        // This should not happen, unless there are errors existing in Sentinel internal.
        RecordLog.info("Sentinel unexpected exception", e1);
    }
    return e;
}
```

com.alibaba.csp.sentinel.slots.DefaultSlotChainBuilder#build

```java
public ProcessorSlotChain build() {
    ProcessorSlotChain chain = new DefaultProcessorSlotChain();
    chain.addLast(new NodeSelectorSlot()); //入口Slot
    chain.addLast(new ClusterBuilderSlot()); //
    chain.addLast(new LogSlot());
    chain.addLast(new StatisticSlot());
    chain.addLast(new AuthoritySlot());
    chain.addLast(new SystemSlot());
    chain.addLast(new FlowSlot());
    chain.addLast(new DegradeSlot());
    return chain;
}
```