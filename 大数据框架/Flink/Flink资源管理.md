Flink涉及到的资源：集群资源和Flink自身资源，前者由资源管理器管理，Flink从资源管理器申请资源容器(Yarn container、K8s Pod)，一个容器中运行一个TaskManager进程。    
为了进行细粒度资源分配，单个容器会被多个Flink任务共享。    
Flink对申请到的资源切分，每份叫做Task Slot（线程）。分成多少份在配置文件里指定，这个值也就意味着一个TM最多可以运行多少Task。**这里不要太纠结Flink中Task Slot里运行意味着什么，
也许就是一个task，即类似keyBy算子或者由于OperationChain机制的存在是多个算子，但由于Slot共享机制，也许是一整个pipeline**。     
JobMaster是资源使用者，像RM申请，RM负责分配资源和释放资源，TM是Slot的持有者，其Slot清单里记录Slot分配给了哪个作业的哪个Task。   
>* SlotManager: RM的组件，记录tm、slot使用信息、基于分配策略分配slot，资源不够时就去通知rm启动更多tm，等到tm注册到slotmanager上后，slotmanager就能分配新的资源；
>* SlotProvider：定义请求模式：立即响应和排队模式
>* SlotPool: 是JobMaster中记录当前作业从TaskManager获取的Slot的集合。JM的调度器从SP获取slot调度任务，SP中没有足够资源时会去通过RM获取资源。等到JM
申请到资源后它就会本地持有slot防止RM异常导致作业运行失败。
>* Slot共享：对应SlotSharingManager，可以在一个Slot中运行Task组成的pipeline。
