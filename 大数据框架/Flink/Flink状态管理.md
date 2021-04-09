使用场景：
>* sum求和
>* 去重
>* CEP

Flink的State提供了对State的操作接口，向上对接Flink的DataStream API，让用户在开发Flink应用的时候可以将临时数据保存在State中，从State中读取数据，
在运行的时候在运行层面上和算子、Function体系融合，自动对State备份（Checkpoint），一旦出现异常能够从保存的State中恢复状态，实现Exact-Once。  
要做状态管理，需要考虑几件事：
>* task内部如何高效保存状态数据和使用状态数据
>* 如何高效地将状态数据存下来，避免状态备份降低集群吞吐，在failover的时候恢复到失败前的状态
>* 状态数据的划分和动态扩容
>* 状态数据的清理

#### 状态类型
>* KeyedState:   支持的类型：ValueState、ListState、MapState、ReducingState、AggregationState、FoldingState
>* OperatorState：  支持的类型： ListState

KeyedState是Stream流上的每个key对应一个State，而后者和一个特定算子的一个实例绑定。  

#### 状态接口
>* 面向开发者的State接口：继承了State接口，具体实现为上文讲述的ValueState、MapState等，这些接口只提供对State中数据的更新、添加、删除等操作，用户无法访问状态的其他运行时的信息；
>* 内部State接口：命令方式为InternalxxxState，比如InternalMapState，不但继承了面向开发者的State接口，也继承了InternalKvState接口，既能访问MapState中保存的数据，也能访问
MapState运行时的信息。

那么UDF是如何访问状态的？   
状态保存在Statebackend中，Flink目前有三种状态后端，状态又分OperatorState和KeyedState。Flink抽象了两个状态访问接口：
>* OperatorStateStore:在内存中维护了一个Map<String, PartitionListState>
>* KeyedStateStore：使用RocksDBStateBackend或者HeapKeyedStateBackend保存数据。

#### 状态存储
无论什么类型的State，都需要持久化到可靠存储才具备应用级别的容错，State的存储在Flink中叫做状态后端（StateBackend），StateBackend需要具备两个能力：
>* 在计算过程中业务逻辑/用户代码能够使用StateBackend的接口访问State；
>* 能够将State持久化到外部存储，提供容错能力。

三种StateBackend：
>* a 纯内存： MemoryStateBackend，运行时State存在TM的内存中，执行检查点时会把State的快照数据保存到JM进程的内存中（同步或者异步都可，不好的地方在于
受限于JM的内存大小，每个State大小不能超过Akka Frame大小）。
>* b 内存+文件： FsStateBackend，运行时State存在TM的内存中，执行检查点时会把State的快照数据存到分布式文件系统或者本地文件系统中（异步写入）。
>* c RocksDB： RocksDBStateBackend，将State通过RocksDB存在本地磁盘，执行检查点时再将整个RocksDB中保存的State数据全量或者增量持久化到配置的fs中，
JM中会存一些checkpoint的元数据信息。

#### 状态持久化
>* 全量持久化策略
>* 增量持久化策略： RocksDBStateBackend支持增量持久化策略的原因要从LSM-Tree展开思考。   
  
#### 状态重分布

#### 状态过期
可以通过相关API设置过期策略。  




