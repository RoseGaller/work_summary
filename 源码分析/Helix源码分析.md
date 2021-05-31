# Helix源码分析

* [概述](#概述)
* [命令](#命令)
* [集群角色](#集群角色)
  * [CONTROLLER](#controller)
    * [Leader选举](#leader选举)
    * [注册监听器](#注册监听器)
      * [LiveInstanceChangeListener](#liveinstancechangelistener)
  * [PARTICIPANT](#participant)
    * [StateModel](#statemodel)


# 概述

http://helix.apache.org/

Apache Helix是一个通用的集群管理框架，用于自动管理集群上的分区、复制和分布式资源。在面对节点故障和恢复、集群扩展和重新配置时自动重新分配资源

# 命令

创建集群

```shell
 ./helix-admin.sh --zkSvr <zk_address> --addCluster <clustername>
```

org.apache.helix.tools.ClusterSetup#addCluster

```java
public void addCluster(String clusterName, boolean overwritePrevious)
{
  _admin.addCluster(clusterName, overwritePrevious); //连接zk，创建/clusterName节点

  //默认创建4类StateModelDefinition，MasterSlave、LeaderStandby、StorageSchemata、OnlineOffline，像uReplicator就选用OnlineOffline，实现Kafka集群间的数据迁移
  StateModelConfigGenerator generator = new StateModelConfigGenerator();
  addStateModelDef(clusterName,
                   "MasterSlave",
                   new StateModelDefinition(generator.generateConfigForMasterSlave()));
  addStateModelDef(clusterName,
                   "LeaderStandby",
                   new StateModelDefinition(generator.generateConfigForLeaderStandby()));
  addStateModelDef(clusterName,
                   "StorageSchemata",
                   new StateModelDefinition(generator.generateConfigForStorageSchemata()));
  addStateModelDef(clusterName,
                   "OnlineOffline",
                   new StateModelDefinition(generator.generateConfigForOnlineOffline()));
}
```

集群中添加节点

```shell
 ./helix-admin.sh --zkSvr <zk_address>  --addNode <clustername> <host:port>
```

集群中添加资源，指定分区数和StateModelName

```shell
./helix-admin.sh --zkSvr <zk_address> --addResource <clustername> <resourceName> <numPartitions> <StateModelName>
```

分配分区到节点

```shell
./helix-admin.sh --zkSvr <zk_address> --rebalance <clustername> <resourceName> <replication factor>
```

启动Helix Controller

```shell
./run-helix-controller.sh --zkSvr <zk_address> --cluster <clustername> 2>&1 > /tmp/controller.log &
```

启动Helix participant

```shell
./start-helix-participant.sh --zkSvr localhost:2199 --cluster MYCLUSTER --host localhost --port 12913 --stateModelType MasterSlave 2>&1 > /tmp/participant_12913.log
```

查看集群列表

```shell
./helix-admin.sh --zkSvr localhost:2199 --listClusters
```

查看集群视图

```shell
./helix-admin.sh --zkSvr <zk_address> --listClusterInfo <clusterName>
```

# 集群角色

## CONTROLLER

部署方式单机(STANDALONE)和集群(DISTRIBUTED)

监听集群中节点的上下线，触发重平衡操作，修改节点的状态

org.apache.helix.controller.HelixControllerMain#startHelixController

```java
public static HelixManager startHelixController(final String zkConnectString,
    final String clusterName, final String controllerName, final String controllerMode)
{
  HelixManager manager = null;
  try
  {
    if (controllerMode.equalsIgnoreCase(STANDALONE)) //单机部署
    {
      manager = HelixManagerFactory.getZKHelixManager(clusterName, controllerName,
          InstanceType.CONTROLLER, zkConnectString);
      manager.connect();
    } else if (controllerMode.equalsIgnoreCase(DISTRIBUTED)) //集群部署，保证CONTROLLER的高客通性。从多个CONTROLLER_PARTICIPANT类型的节点中选取一个作为leader，选中的节点创建CONTROLLER
    {
      manager = HelixManagerFactory.getZKHelixManager(clusterName, controllerName,
          InstanceType.CONTROLLER_PARTICIPANT, zkConnectString);
			//负责创建DistClusterControllerStateModel，监听CONTROLLER_PARTICIPANT类型的节点的状态变化
      DistClusterControllerStateModelFactory stateModelFactory = new DistClusterControllerStateModelFactory(
          zkConnectString);

      StateMachineEngine stateMach = manager.getStateMachineEngine();
      stateMach.registerStateModelFactory("LeaderStandby", stateModelFactory);
      manager.connect();
    } else
    {
      logger.error("cluster controller mode:" + controllerMode + " NOT supported");
      // throw new
      // IllegalArgumentException("Unsupported cluster controller mode:" +
      // controllerMode);
    }
  } catch (Exception e)
  {
    logger.error("Exception while starting controller",e);
  }

  return manager;
}
```

### Leader选举

负责在Controller集群中选举leader节点

org.apache.helix.participant.DistClusterControllerElection#onControllerChange

```java
public synchronized void onControllerChange(NotificationContext changeContext) {
HelixManager manager = changeContext.getManager();
if (manager == null) {
    LOG.error("missing attributes in changeContext. requires HelixManager");
    return;
}

InstanceType type = manager.getInstanceType();
//节点类型必须是CONTROLLER或者CONTROLLER_PARTICIPANT
if (type != InstanceType.CONTROLLER && type != InstanceType.CONTROLLER_PARTICIPANT) {
    LOG.error("fail to become controller because incorrect instanceType (was "
       + type.toString() + ", requires CONTROLLER | CONTROLLER_PARTICIPANT)");
    return;
}

try {
    if (changeContext.getType().equals(NotificationContext.Type.INIT)
       || changeContext.getType().equals(NotificationContext.Type.CALLBACK)) {
   // DataAccessor dataAccessor = manager.getDataAccessor();
   HelixDataAccessor accessor = manager.getHelixDataAccessor();
   Builder keyBuilder = accessor.keyBuilder();

   while (accessor.getProperty(keyBuilder.controllerLeader()) == null) { //尚无leader节点
       boolean success = tryUpdateController(manager); //尝试创建获取leader
       if (success) { //当前节点获取leader成功
      updateHistory(manager);
      if (type == InstanceType.CONTROLLER) { //注册GenericHelixController监听器
          HelixControllerMain.addListenersToController(manager, _controller);
          manager.startTimerTasks();
      } else if (type == InstanceType.CONTROLLER_PARTICIPANT) { //集群模式
          String clusterName = manager.getClusterName();
          String controllerName = manager.getInstanceName();
          _leader = HelixManagerFactory.getZKHelixManager(clusterName,
             controllerName, InstanceType.CONTROLLER, _zkAddr);

          _leader.connect();
          _leader.startTimerTasks();
          HelixControllerMain.addListenersToController(_leader, _controller);
      }

       }
   }
    } else if (changeContext.getType().equals(NotificationContext.Type.FINALIZE)) {

   if (_leader != null) {
       _leader.disconnect();
   }
    }

} catch (Exception e) {
    LOG.error("Exception when trying to become leader", e);
}
   }
```

### 注册监听器

org.apache.helix.controller.HelixControllerMain#addListenersToController

```java
public static void addListenersToController(HelixManager manager,
    GenericHelixController controller)
{
  try
  {
    manager.addConfigChangeListener(controller);
    manager.addLiveInstanceChangeListener(controller);
    manager.addIdealStateChangeListener(controller);
    manager.addExternalViewChangeListener(controller);
    manager.addControllerListener(controller);
  } catch (ZkInterruptedException e)
  {
    logger
        .warn("zk connection is interrupted during HelixManagerMain.addListenersToController(). "
            + e);
  } catch (Exception e)
  {
    logger.error("Error when creating HelixManagerContollerMonitor", e);
  }
}
```

#### LiveInstanceChangeListener

org.apache.helix.manager.zk.ZKHelixManager#addLiveInstanceChangeListener

```java
public void addLiveInstanceChangeListener(LiveInstanceChangeListener listener) throws Exception
{
  logger.info("ClusterManager.addLiveInstanceChangeListener()");
  checkConnected();
  final String path = _helixAccessor.keyBuilder().liveInstances().getPath();
  CallbackHandler callbackHandler =
      createCallBackHandler(path,
                            listener,
                            new EventType[] { 
                              EventType.NodeDataChanged, 
                              EventType.NodeChildrenChanged,
                              EventType.NodeDeleted, EventType.NodeCreated 
                            }, 	//监听的事件类型 
                            LIVE_INSTANCE);
  addListener(callbackHandler);
}
```

org.apache.helix.manager.zk.CallbackHandler#invoke

```java
public void invoke(NotificationContext changeContext) throws Exception
{
  // This allows the listener to work with one change at a time
  synchronized (_manager)
  {
    Builder keyBuilder = _accessor.keyBuilder();
    long start = System.currentTimeMillis();
    if (logger.isInfoEnabled())
    {
      logger.info(Thread.currentThread().getId() + " START:INVOKE "
      // + changeContext.getPathChanged()
          + _path + " listener:" + _listener.getClass().getCanonicalName());
    }

    if (_changeType == IDEAL_STATE)
    {

      IdealStateChangeListener idealStateChangeListener =
          (IdealStateChangeListener) _listener;
      subscribeForChanges(changeContext, _path, true, true);
      List<IdealState> idealStates = _accessor.getChildValues(keyBuilder.idealStates());

      idealStateChangeListener.onIdealStateChange(idealStates, changeContext);

    }
    else if (_changeType == CONFIG)
    {

      ConfigChangeListener configChangeListener = (ConfigChangeListener) _listener;
      subscribeForChanges(changeContext, _path, true, true);
      List<InstanceConfig> configs =
          _accessor.getChildValues(keyBuilder.instanceConfigs());

      configChangeListener.onConfigChange(configs, changeContext);

    }
    else if (_changeType == LIVE_INSTANCE)
    {
      LiveInstanceChangeListener liveInstanceChangeListener =
          (LiveInstanceChangeListener) _listener;
      subscribeForChanges(changeContext, _path, true, true);
      List<LiveInstance> liveInstances =
          _accessor.getChildValues(keyBuilder.liveInstances());

      liveInstanceChangeListener.onLiveInstanceChange(liveInstances, changeContext);

    }
    else if (_changeType == CURRENT_STATE)
    {
      CurrentStateChangeListener currentStateChangeListener;
      currentStateChangeListener = (CurrentStateChangeListener) _listener;
      subscribeForChanges(changeContext, _path, true, true);
      String instanceName = PropertyPathConfig.getInstanceNameFromPath(_path);
      String[] pathParts = _path.split("/");

      // TODO: fix this
      List<CurrentState> currentStates =
          _accessor.getChildValues(keyBuilder.currentStates(instanceName,
                                                            pathParts[pathParts.length - 1]));

      currentStateChangeListener.onStateChange(instanceName,
                                               currentStates,
                                               changeContext);

    }
    else if (_changeType == MESSAGE)
    {
      MessageListener messageListener = (MessageListener) _listener;
      subscribeForChanges(changeContext, _path, true, false);
      String instanceName = PropertyPathConfig.getInstanceNameFromPath(_path);
      List<Message> messages =
          _accessor.getChildValues(keyBuilder.messages(instanceName));

      messageListener.onMessage(instanceName, messages, changeContext);

    }
    else if (_changeType == MESSAGES_CONTROLLER)
    {
      MessageListener messageListener = (MessageListener) _listener;
      subscribeForChanges(changeContext, _path, true, false);
      List<Message> messages =
          _accessor.getChildValues(keyBuilder.controllerMessages());

      messageListener.onMessage(_manager.getInstanceName(), messages, changeContext);

    }
    else if (_changeType == EXTERNAL_VIEW)
    {
      ExternalViewChangeListener externalViewListener =
          (ExternalViewChangeListener) _listener;
      subscribeForChanges(changeContext, _path, true, true);
      List<ExternalView> externalViewList =
          _accessor.getChildValues(keyBuilder.externalViews());

      externalViewListener.onExternalViewChange(externalViewList, changeContext);
    }
    else if (_changeType == ChangeType.CONTROLLER)
    {
      ControllerChangeListener controllerChangelistener =
          (ControllerChangeListener) _listener;
      subscribeForChanges(changeContext, _path, true, false);
      controllerChangelistener.onControllerChange(changeContext);
    }
    else if (_changeType == ChangeType.HEALTH)
    {
      HealthStateChangeListener healthStateChangeListener =
          (HealthStateChangeListener) _listener;
      subscribeForChanges(changeContext, _path, true, true); // TODO: figure out
      // settings here
      String instanceName = PropertyPathConfig.getInstanceNameFromPath(_path);

      List<HealthStat> healthReportList =
          _accessor.getChildValues(keyBuilder.healthReports(instanceName));

      healthStateChangeListener.onHealthChange(instanceName,
                                               healthReportList,
                                               changeContext);
    }
    long end = System.currentTimeMillis();
    if (logger.isInfoEnabled())
    {
      logger.info(Thread.currentThread().getId() + " END:INVOKE " + _path
          + " listener:" + _listener.getClass().getCanonicalName() + " Took: "
          + (end - start) +"ms");
    }
  }
}
```

监听分区节点的变化

org.apache.helix.controller.GenericHelixController#onLiveInstanceChange

```java
public void onLiveInstanceChange(List<LiveInstance> liveInstances,
                                 NotificationContext changeContext)
{
  logger.info("START: Generic GenericClusterController.onLiveInstanceChange()");
  if (liveInstances == null)
  {
    liveInstances = Collections.emptyList();
  }
  // Go though the live instance list and make sure that we are observing them
  // accordingly. The action is done regardless of the paused flag.
  if (changeContext.getType() == NotificationContext.Type.INIT ||
      changeContext.getType() == NotificationContext.Type.CALLBACK)
  {
    checkLiveInstancesObservation(liveInstances, changeContext);
  }

  ClusterEvent event = new ClusterEvent("liveInstanceChange");
  event.addAttribute("helixmanager", changeContext.getManager());
  event.addAttribute("changeContext", changeContext);
  event.addAttribute("eventData", liveInstances);
  handleEvent(event);
  logger.info("END: Generic GenericClusterController.onLiveInstanceChange()");
}
```

org.apache.helix.controller.GenericHelixController#handleEvent

```java
protected synchronized void handleEvent(ClusterEvent event)
{
  HelixManager manager = event.getAttribute("helixmanager");
  if (manager == null)
  {
    logger.error("No cluster manager in event:" + event.getName());
    return;
  }

  if (!manager.isLeader())
  {
    logger.error("Cluster manager: " + manager.getInstanceName()
        + " is not leader. Pipeline will not be invoked");
    return;
  }

  if (_paused)
  {
    logger.info("Cluster is paused. Ignoring the event:" + event.getName());
    return;
  }

  NotificationContext context = null;
  if (event.getAttribute("changeContext") != null)
  {
    context = (NotificationContext) (event.getAttribute("changeContext"));
  }

  // Initialize _clusterStatusMonitor
  if (context != null)
  {
    if (context.getType() == Type.FINALIZE)
    {
      if (_clusterStatusMonitor != null)
      {
        _clusterStatusMonitor.reset();
        _clusterStatusMonitor = null;
      }
      
      stopRebalancingTimer();
      logger.info("Get FINALIZE notification, skip the pipeline. Event :" + event.getName());
      return;
    }
    else
    {
      if (_clusterStatusMonitor == null)
      {
        _clusterStatusMonitor = new ClusterStatusMonitor(manager.getClusterName());
      }
      
      event.addAttribute("clusterStatusMonitor", _clusterStatusMonitor);
    }
  }

  List<Pipeline> pipelines = _registry.getPipelinesForEvent(event.getName());
  if (pipelines == null || pipelines.size() == 0)
  {
    logger.info("No pipeline to run for event:" + event.getName());
    return;
  }

  for (Pipeline pipeline : pipelines)
  {
    try
    {
      pipeline.handle(event);
      pipeline.finish();
    }
    catch (Exception e)
    {
      logger.error("Exception while executing pipeline: " + pipeline
          + ". Will not continue to next pipeline", e);
      break;
    }
  }
}
```

## PARTICIPANT

监听当前节点的状态变更，执行对应的业务逻辑

```java
HelixManager manager = HelixManagerFactory.getZKHelixManager(CLUSTER_NAME,
          instanceName, InstanceType.PARTICIPANT, ZK_ADDRESS);
//创建StateMachineEngine
MasterSlaveStateModelFactory stateModelFactory = new MasterSlaveStateModelFactory(
  instanceName);
//当节点的状态发生变更时，执行相应的逻辑
StateMachineEngine stateMach = manager.getStateMachineEngine();
//注册MasterSlaveStateModelFactory
stateMach.registerStateModelFactory(STATE_MODEL_NAME, stateModelFactory);
manager.connect();
```

### StateModel

定义了状态流转对应的业务逻辑，比如当前的状态由slave转为master时，会调用StateModel的onBecomeMasterFromSlave方法

org.apache.helix.examples.MasterSlaveStateModelFactory#createNewStateModel

```java
public StateModel createNewStateModel(String partitionName) //创建StateModel
{
  MasterSlaveStateModel stateModel = new MasterSlaveStateModel();
  stateModel.setInstanceName(_instanceName);
  stateModel.setDelay(_delay);
  stateModel.setPartitionName(partitionName);
  return stateModel;
}
```

```java
public static class MasterSlaveStateModel extends StateModel
{
  int _transDelay = 0;
  String partitionName;
  String _instanceName = "";

  public String getPartitionName()
  {
    return partitionName;
  }

  public void setPartitionName(String partitionName)
  {
    this.partitionName = partitionName;
  }

  public void setDelay(int delay)
  {
    _transDelay = delay > 0 ? delay : 0;
  }

  public void setInstanceName(String instanceName)
  {
    _instanceName = instanceName;
  }
	//Offline -> Slave
  public void onBecomeSlaveFromOffline(Message message,
      NotificationContext context)
  {

    System.out.println(_instanceName + " transitioning from "
        + message.getFromState() + " to " + message.getToState() + " for "
        + partitionName);
    sleep();
  }

  private void sleep()
  {
    try
    {
      Thread.sleep(_transDelay);
    } catch (Exception e)
    {
      e.printStackTrace();
    }
  }
	//Master -> Slave
  public void onBecomeSlaveFromMaster(Message message,
      NotificationContext context)
  {
    System.out.println(_instanceName + " transitioning from "
        + message.getFromState() + " to " + message.getToState() + " for "
        + partitionName);
    sleep();

  }
	//Slave -> Master
  public void onBecomeMasterFromSlave(Message message,
      NotificationContext context)
  {
    System.out.println(_instanceName + " transitioning from "
        + message.getFromState() + " to " + message.getToState() + " for "
        + partitionName);
    sleep();

  }
	//Slave->Offline
  public void onBecomeOfflineFromSlave(Message message,
      NotificationContext context)
  {
    System.out.println(_instanceName + " transitioning from "
        + message.getFromState() + " to " + message.getToState() + " for "
        + partitionName);
    sleep();

  }
	//Offline -> Dropped
  public void onBecomeDroppedFromOffline(Message message,
      NotificationContext context)
  {
    System.out.println(_instanceName + " Dropping partition "
        + partitionName);
    sleep();

  }
}
```
