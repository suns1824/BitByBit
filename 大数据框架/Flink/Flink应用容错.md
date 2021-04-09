####容错保证语义
####检查点和保存点
前者根据配置周期性通知Stream中各个算子的状态生成检查点快照，当程序崩溃时就会恢复到最后一次快照的状态从此刻重新执行。检查点默认不会保留，
Flink只保留最近成功生成的一个检查点。   
后者往往用于应用升级、集群迁移等场景，保存点可以视为一个算子id->State的Map，使用如下：
```text
flink savepoint jobId [targetDirectory] [-yid yarnAppId]
flink run -s savepointPath [runArgs]
```
#### 轻量级异步分布式快照
Flink使用轻量级分布式快照实现应用容错。  
分布式快照最关键的是将数据流切分，在触发快照的时候，CheckpointCoordinator向Source算子注入barrier消息，Flink使用Barrier切分数据流。
两个barrier之间的数据流的数据隶属于同一检查点。每个barrier都携带一个其所属快照的ID编号，Barrier随着数据向下流动，不会打断数据流，因此非常
轻量。数据恢复就是从barrier所在位置重新处理数据，比如kafka，这个位置就是分区内的offset。  
ps：barrier从数据流源头向下游传递，当一个非数据源算子从所有的输入流中收到快照n的barrier，该算子就会对自己的state保存快照，并向下游广播
快照n的barrier，一旦sink算子收到barrier，分情况讨论：  
>* 引擎内严格一次：收到上游所有barrier后，sink算子对自己的state进行快照，然后通知checkpointcoordinator。
>* 端对端严格一次：多了sink算子提交事务的过程。

理解barrier对齐：为了达到引擎内严格一次和端对端严格一次两种保证语义，略。   

#### 检查点执行过程
JM中的CheckpointCoordinator组件专门负责检查点的管理，包括何时触发检查点、检查点完成的确认。检查点的具体执行者是作业的各个task，task将执行交给算子。  
a. JM的CheckpointCoordinator周期性触发检查点执行(然后通过akka rpc到taskexecutor)；  
b. TaskExecutor执行检查点： 
>* Task层面检查
>* StreamTask执行检查点
>* 算子生成快照：StreamTask异步出发OperatorChain中所有算子的检查点。算子开始从StateBackend中深度复制State数据，并持久化到外部存储中。注册回调，
向JM报告已完成。  

c. JM确认检查点。

#### 检查点恢复
略

#### 端到端严格一次
两阶段提交协议：p247
两阶段提交实现：p250
