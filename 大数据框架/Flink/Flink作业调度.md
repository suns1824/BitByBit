[上文](Flink作业提交.md)有个被一笔带过的地方：
```text
clusterClient.submitJob(jobGraph)
```
个人觉得这里有必要梳理一波，两点：
>* RestClusterClient是如何submit的？
>* submitJob后发生了什么(集群状态的变化、Flink执行层面的逻辑)？

第一个问题没啥好说的，核心就是基于zk/k8s(我猜测是api server或者说etcd)去拿到JobManager的host和port，然后发起rpc调用，从源码来看，
这里的rpc并没有使用Akka，而是基于Netty去做([Netty服务端创建和线程模型](../../一些值得探究的框架和思想/Netty/source-read.md))。
第二个问题得从Dispatcher的submitJob方法入手，在restclient发起request后，server端的处理逻辑就在这个方法里。关于为啥从submitJob入手，查找
调用链路即可，这里涉及到[Flink RPC知识](./FlinkRPC.md)和[Flink状态管理](./Flink状态管理.md)。   
```text
submitJob(jobGraph, timeout) {
    internalSubmitJob(jobGraph) {
        persistAndRunJob(jobGraph) {
            jobGraphWriter.putJobGraph(jobGraph) {
                //znode查询是否存在，如果集群是基于k8s，那就是查看configmap
                final R currentVersion = jobGraphStateHandleStore.exists(name);
                if (!currentVersion.isExisting()) {
                    jobGraphStateHandleStore.addAndLock(name, jobGraph) {
                        //jobgraph存到hdfs上
                        RetrievableStateHandle<T> storeHandle = storage.store(jobGraph);
                        //序列化stateHandle，然后将其写到zk上，不是state本身，因为state可能会比较大，zk一般都是存kb级别的对象。
                        byte[] serializedStoreHandle = InstantiationUtil.serializeObject(storeHandle);
                        ...
                    }
                    addedJobGraphs.add(jobID);
                    success = true;
                } else if (addedJobGraphs.contains(jobID)) {
                    //znode更新
                    jobGraphStateHandleStore.replace(name, currentVersion, jobGraph);
                    success = true;
                }
            }
            runJob(jobGraph, ExecutionType.SUBMISSION) {
                //创建jobmaster,这里会初始化很多组件，rpc、高可用、heartbeat服务，调度器,slotpool, shufflemaster等，关于jm的作用见书的10.5。
                createJobManagerRunner(jobGraph, initializationTimestamp) {
                    leaderElectionService.start(this){
                        leader选举有两种，基于zk和基于k8s，这里以zk为例，用到了curator的leaderlatch和nodecache(flink为了解决版本的
                        问题，引入了flink-shaded这个黑魔法,某种程度上会造成学习源码的难度加大)。在leader选举成功后会调用
                        ZooKeeperLeaderElectionDriver的isLeader方法(注意该接口的继承关系)，在isLeader方法中最终调用了jobmaster的start方法。
                        leaderElectionDriverFactory.createLeaderElectionDriver(x,x, x) {
                            leaderLatch = new LeaderLatch(client, checkNotNull(latchPath));
                            cache = new NodeCache(client, leaderPath);
                            client.getUnhandledErrorListenable().addListener(this);
                            running = true;
                            leaderLatch.addListener(this);
                            leaderLatch.start();
                            cache.getListenable().addListener(this);
                            cache.start();
                            client.getConnectionStateListenable().addListener(listener);
                        }
                        
                    }
                }
            }
        }
    }
}
```
因为jobmaster需要做HA，所以就需要leader选举机制。Flink支持两种模式，一种通过zk的LeaderLatch和NodeCache还有一种通过k8s(见LeaderElectionDriverFactory)。
如果对Curator做leader选举的模式和Flink-shaded不了解，那么创建jobmaster的过程，也就是createJobManagerRunner方法就会比较难以琢磨，
因为你根本不知道jobmaster的start方法是怎么调用的，不断追溯发现调用的是一个isLeader方法，然而再走就走到了flink-shaded下的库里去，然而...所以这里会有一定的阻碍。
这样的情况在学习Flink其他有HA的组件模块可能也会有类似遭遇。   
所以，我们现在知道了JobMaster的start方法是从哪里发起的：isLeader方法，我们从这里开始阅读，顺着调用栈往下走，很容易追踪到JobMaster的start方法：
```text
public CompletableFuture<Acknowledge> start(final JobMasterId newJobMasterId) throws Exception {
        //发起rpc调用（akka、tell了一下），感觉没啥含义，就是一种确认actor状态正常的意味。
        start();
        startJobExecution(newJobMasterId), RpcUtils.INF_TIMEOUT) {
            startJobMasterServices() {
                //维持与rm、tm之间的心跳
                startHeartbeatServices();
                //启动slotpool
                slotPool.start(getFencingToken(), getAddress(), getMainThreadExecutor());
                //连上之前的rm
                reconnectToResourceManager(new FlinkException("Starting JobMaster component."));
                resourceManagerLeaderRetriever.start(new ResourceManagerLeaderListener());
            }
            //作业调度，具体参考下文
            resetAndStartScheduler() {
            }
        }
    }
```
resetAndStartScheduler介绍：
在理解作业调度之前还需要知道作业图转换也就是JobGraph-->ExecutionGraph(增加并行度概念)是在JobMaster中完成的（JobGraph的构建在client端）。   
ExecutionGraph的生成过程在SchedulerBase的构造方法中触发构建（从JobMaster中找寻SchedulerNG的初始化），
最终调用SchedulerBase的createExecutionGraph出发实际的构建动作，核心逻辑在ExecutionGraphBuilder：
```text
public static ExectionGraph buildGraph() {
    //对JobGraph中对JObVertex就行拓扑排序得到列表
    List<JobVertext> sortedTopology = jobGraph.getVerticesSortedTopologicallyFromSources();
    //构建ExecutionGraph
    executionGraph.attachJobGraph(sortedTopology) {
        //遍历
        for (JobVertex jobVertex : sortedTopology) {
             //创建ExecutionJobVertex
             ExecutionVertex ejv = ...
             //将ejv与前置的IntermediateResult连接起来
             ejv.connectToPredecessors(this.intermediateResults);
        }
    }
}
```
1.构建ExecutionGraph节点
ExecutionGraph中的ExecutionJobVertex和JobGraph中的JobVertex对应，EG(简称，笔记习惯)有并行度特性，一个EJV有多个EV，这个数量和并行度一致，
简单来讲ExectionVertex对应于Task。核心逻辑：
>* 设置并行度
>* 设置slot共享和CoLocationGroup
>* 构建当前EJV的中间结果以及中间结果分区。每对应一个下游JobEdge，创建一个中间结果。
>* 根据EJV的并行度创建对应数量的EV，运行的时候就会部署相应数量的Task到TM上。
>* 检查中间结果分区和EV之间有没有重复引用。
>* 对可切分的数据源进行切分。

2.构建ExecutionEdge
如上文代码所示，调用ejv.connectToPredecessors将EV和中间结果关联起来，运行时建立Task之间的数据交换就是以此为基础建立数据的物理传输通道。  
根据JobEdge的DistributionPattern属性创建ExecutionEdge，将EV和上游的中间结果分区连接起来（分为点对点连接和全连接）。

至此ExecutionGraph的构建讲述完毕，回到主干来：
```text
resetAndStartScheduler() {
    //核心逻辑
    FutureUtils.assertNoException(schedulerAssignedFuture.thenRun(this::startScheduling));
}
```
startScheduling最终走到：
```text
DefaultScheduler.java
protected void startSchedulingInternal() {
        prepareExecutionGraphForNgScheduling();
        //具体的调度策略解释见下文解释。
        schedulingStrategy.startScheduling();
    }
```
Flink提供了三种调度模式：
>* Eager调度：适用于流处理，一次性申请需要的所有资源
>* 分阶段调度：适用于批处理，从Source Task开始分阶段调度，一次性申请本阶段的所有资源。
>* 分阶段slot重用调度：适用于批处理，可以在资源不足情况下执行作业，前提是本阶段没有shuffle。
流批的调度策略不同，流是一次性申请全部资源，批处理可以分阶段申请，所以有具体两种调度策略的实现：
>* EagerSchelingStrategy
>* LazyFromSourceSchedulingStrategy

#### 流作业启动调度
流计算调度策略计算所有需要调度的ExecutionVertex，然后把需要调度的ExecutionVertex交给DefaultScheduler#allocateSlotsAndDeploy，
最终调用Execution#deploy开始部署作业，当所有Task启动之后就算作业启动成功。 
1. 
```text
//EagerSchedulingStrategy.java
//流作业申请slot部署task，申请之前先为所有的task构建部署信息
private void allocateSlotsAndDeploy(final Set<ExecutionVertexID> verticesToDeploy) {
        //所有EV一起调度
        final List<ExecutionVertexDeploymentOption> executionVertexDeploymentOptions =
                SchedulingStrategyUtils.createExecutionVertexDeploymentOptionsInTopologicalOrder(
                        schedulingTopology, verticesToDeploy, id -> deploymentOption);
        //具体执行deploy的逻辑在这里，顺着调用栈往下分析即可，下文概括
        schedulerOperations.allocateSlotsAndDeploy(executionVertexDeploymentOptions);
    }
```
allocateSlotsAndDeploy：流计算需要一次性部署所有task，所以会对所有的Execution异步获取slot，申请到slot后，最红调用Execution#deploy进
行实际的部署，实际上就是将Task部署相关的信息通过TaskManagerGateway发送给TM。
#### 批作业启动调度
略
#### TM启动Task
JM发起部署task的RPC后，TM会收到请求，TM从部署消息中获取Task执行所需要的信息，初始化Task，然后触发Task的执行。在部署和执行的过程中，
TaskExecutor(TM)和JM保持交互，将Task的状态汇报给JM，并接受JM的Task管理操作。具体参考书10.6.4和10.5.2-4。
