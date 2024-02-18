

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
  	//context数量超过上限
    if (context instanceof NullContext) { 
        return new CtEntry(resourceWrapper, null, context);
    }
		//创建Context
    if (context == null) { 
        context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
    }
 		//开关是否打开
    if (!Constants.ON) {
        return new CtEntry(resourceWrapper, null, context);
    }

   //最多6000
    ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper); //构建slot处理链

    if (chain == null) {
        return new CtEntry(resourceWrapper, null, context);
    }

    Entry e = new CtEntry(resourceWrapper, chain, context);
    try {
      	//执行slot
        chain.entry(context, resourceWrapper, null, count, prioritized, args);
    } catch (BlockException e1) { 
      	//触发流控，抛出BlockException
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
  	// 构建链路
    ProcessorSlotChain chain = new DefaultProcessorSlotChain();
    chain.addLast(new NodeSelectorSlot()); 
    //用于构建资源的 ClusterNode
    chain.addLast(new ClusterBuilderSlot()); 
    chain.addLast(new LogSlot());
    chain.addLast(new StatisticSlot());
    chain.addLast(new AuthoritySlot());
    chain.addLast(new SystemSlot());
  	//主要完成限流
    chain.addLast(new FlowSlot());
  	//熔断,主要针对资源的平均响应时间以及异常比率
    chain.addLast(new DegradeSlot());
    return chain;
}
```

FlowSlot

com.alibaba.csp.sentinel.slots.block.flow.FlowRuleChecker#checkFlow

```java
public void checkFlow(Function<String, Collection<FlowRule>> ruleProvider, ResourceWrapper resource,
                      Context context, DefaultNode node, int count, boolean prioritized) throws BlockException {
    if (ruleProvider == null || resource == null) {
        return;
    }
    Collection<FlowRule> rules = ruleProvider.apply(resource.getName());
    if (rules != null) {
        for (FlowRule rule : rules) {
            if (!canPassCheck(rule, context, node, count, prioritized)) {
                throw new FlowException(rule.getLimitApp(), rule);
            }
        }
    }
}
```

```java
public boolean canPassCheck(/*@NonNull*/ FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                                boolean prioritized) {
    String limitApp = rule.getLimitApp();
    if (limitApp == null) { //流控针对的调用来源，若为 default 则不区分调用来源
        return true;
    }

    if (rule.isClusterMode()) { // 标识是否为集群限流配置
        return passClusterCheck(rule, context, node, acquireCount, prioritized);
    }

    return passLocalCheck(rule, context, node, acquireCount, prioritized);
}
```

```java
//本地流量检测
private static boolean passLocalCheck(FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                      boolean prioritized) {
    Node selectedNode = selectNodeByRequesterAndStrategy(rule, context, node);
    if (selectedNode == null) {
        return true;
    }

    return rule.getRater().canPass(selectedNode, acquireCount, prioritized);
}
```

com.alibaba.csp.sentinel.slots.block.degrade.DegradeRuleManager#checkDegrade

```java
public static void checkDegrade(ResourceWrapper resource, Context context, DefaultNode node, int count)
    throws BlockException {

    Set<DegradeRule> rules = degradeRules.get(resource.getName());
    if (rules == null) {
        return;
    }

    for (DegradeRule rule : rules) {
        if (!rule.passCheck(context, node, count)) {
            throw new DegradeException(rule.getLimitApp(), rule);
        }
    }
}
```

com.alibaba.csp.sentinel.slots.block.degrade.DegradeRule#passCheck

```java
public boolean passCheck(Context context, DefaultNode node, int acquireCount, Object... args) {
    //熔断中
    if (cut.get()) { 
        return false;
    }

    //资源节点
    ClusterNode clusterNode = ClusterBuilderSlot.getClusterNode(this.getResource());
    if (clusterNode == null) {
        return true;
    }

    if (grade == RuleConstant.DEGRADE_GRADE_RT) { //响应时间
        double rt = clusterNode.avgRt();
        if (rt < this.count) { 
            passCount.set(0);//未超时
            return true;
        }

        if (passCount.incrementAndGet() < rtSlowRequestAmount) {
            return true;
        }
    } else if (grade == RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO) { //异常比例
        double exception = clusterNode.exceptionQps();
        double success = clusterNode.successQps();
        double total = clusterNode.totalQps();
        // If total amount is less than minRequestAmount, the request will pass.
        if (total < minRequestAmount) {
            return true;
        }

        if (realSuccess <= 0 && exception < minRequestAmount) {
            return true;
        }

        if (exception / success < count) {
            return true;
        }
    } else if (grade == RuleConstant.DEGRADE_GRADE_EXCEPTION_COUNT) { //异常次数
        double exception = clusterNode.totalException();
        if (exception < count) {
            return true;
        }
    }

    if (cut.compareAndSet(false, true)) { //触发熔断
        ResetTask resetTask = new ResetTask(this);
        pool.schedule(resetTask, timeWindow, TimeUnit.SECONDS); //经过timeWindow关闭熔断
    }

    return false;
}
```