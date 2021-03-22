## 前言
最近公司基于zeeplin搭了一套Flink SQL开发平台，开发人员只需写SQL就完了。简单的尝试后我遇到了一些问题，我想扩展Flink SQL的能力，
比如我想实现流与维表的join，基于开源的Flink，似乎目前依旧无法实现(我在zeeplin上尝试使用PERIOD FOR SYSTEM_TIME这样的语法直接报语法错误)，在研究
相关的[开源实现](https://github.com/DTStack/flinkStreamSQL) 时感觉收获颇多。该项目实现了一套完整的Flink作业提交流程：  
**解析参数-->解析自定义SQL树-->注册UDF-->注册表和schema-->执行SQL-->env.execute**  
目前该框架支持local、yarn-perjob、yarn-session、standalone模式，也有一些改进，比如将jar直接放到hdfs上，个人感觉理念类似application模式。
随着研究的深入，我觉得自己很有必要去理解一下Flink job提交流程，同时也想要扩展实现一套支持flink on kubernetes的方案，对于实现一个SQL化的
开发平台也有了一些理解。俗话说，好记性不如烂笔头，还是做份笔记让自己能够更加深入理解一下Flink的设计思想。  
在真正执行Flink作业之前，我们需要搞明白几件事：
>* Flink作业是如何提交到集群的？
>* Flink集群是如何在资源管理集群上启动起来的？
>* Flink的计算资源如何分配给作业？ 
>* Flink作业提交后是如何启动的？   
**这些问题一言难尽，三言两语可以说个大概，但是精华往往在细节之处。我们跟随代码走进去！**
## 正文
关于Flink job在yarn-session模式下的作业提交流程，就不得不说到两个类：FlinkYarnSessionCli和CliFrontend。
### FlinkYarnSessionCli
在Flink的bin目录下，有一个脚本叫yarn-session.sh,执行这个脚本，就会创建一个yarn-session模式的Flink集群。在这个脚本内，有一个关键的
字眼：**FlinkYarnSessionCli**，它的核心方法就是run方法（这里只展示其主要流程，细节会在 下文展开说明）：   
```text
public int run(String[] args) throws CliArgsException, FlinkException {
    //1.将配置和传入参数合并成新的全量Configuration;
    effectiveConfiguration.addAll(commandLineConfiguration);
    //2.基于spi机制加载YarnClusterClientFactory，这是一个封装了对yarn cluster操作的工厂类;
    final ClusterClientFactory<ApplicationId> yarnClusterClientFactory =
                    clusterClientServiceLoader.getClusterClientFactory(effectiveConfiguration);
    //3.创建YarnClusterDescriptor，其构造方法内有一个yarnclient对象
    final YarnClusterDescriptor yarnClusterDescriptor = 
                    (YarnClusterDescriptor)
                            yarnClusterClientFactory.createClusterDescriptor(effectiveConfiguration);
    //Flink on yarn集群在创建(执行yarn-session.sh)后会产生一个applicationId，如果启动参数中夹带了这个值，说明已存在一个flink on yarn集群，不需要再启动；
    if (cmd.hasOption(applicationId.getOpt())) {
        //4-1.这里发生了yarn的一次rpc调用，返回的applicationreport中包括了jobmanager的host和port等信息，基于此封装了一个RestClusterClient作为返回；
        clusterClientProvider = yarnClusterDescriptor.retrieve(yarnApplicationId);
    } else {
        //4-2.根据configuration构造clusterSpecification，包括jobmanager的内存、taskmanager的内存和每个taskmanager的taskslot大小；
        final ClusterSpecification clusterSpecification =
                                    yarnClusterClientFactory.getClusterSpecification(
                                            effectiveConfiguration);
        //5.启动一个yarn-session模式的Flink集群，下文重点讲述；
        clusterClientProvider =
                            yarnClusterDescriptor.deploySessionCluster(clusterSpecification);
        //略
        ClusterClient<ApplicationId> clusterClient = clusterClientProvider.getClusterClient();
        yarnApplicationId = clusterClient.getClusterId();
        //6.确认启动成功后，将重要信息封装成properties文件，并持久化至本地磁盘
        writeYarnPropertiesFile(yarnApplicationId, dynamicPropertiesEncoded);
    }
    if (detached模式) {
    } else {
        //7.创建一个yarn-session集群的monitor，这里加了一个hook，在集群退出时能够有所感知，做一些资源清理的工作；
        final YarnApplicationStatusMonitor yarnApplicationStatusMonitor = ......
    }
    //8.运行交互式客户端，循环获取集群状态，并做相应处理；
    runInteractiveCli(yarnApplicationStatusMonitor, acceptInteractiveInput);   
    //收尾工作，略。
}
```
**以上是从client端视角对yarn-session模式的Flink集群启动的观察，对于细节我们再对上面8个步骤中的重点部分做进一步的分析，这里会涉及到Yarn的源码的学习。**     
#### 步骤3
```text
private YarnClusterDescriptor getClusterDescriptor(Configuration configuration) {
        //创建了YarnClient对象，具体实现类是YarnClientImpl。值得一提的是Yarn对于一些长期存在的对象都将其服务化，实现了Service接口。
        final YarnClient yarnClient = YarnClient.createYarnClient();
        final YarnConfiguration yarnConfiguration = new YarnConfiguration();
        //包括了配置和组件的初始化
        yarnClient.init(yarnConfiguration);
        //创建rmClient，其实就是一个RPC client
        yarnClient.start();
        return new YarnClusterDescriptor(
                configuration,
                yarnConfiguration,
                yarnClient,
                YarnClientYarnClusterInformationRetriever.create(yarnClient),
                false);
    }
```
#### 步骤4-1
```text
public ClusterClientProvider<ApplicationId> retrieve(ApplicationId applicationId) {
    //这里会借助rmClient去实现远程调用，server端的实现参见ClientRMService.getApplicationReport,server端会做权限校验，
    //然后查询RM中该applicationId对应的Flink集群的状态；
    final ApplicationReport report = yarnClient.getApplicationReport(applicationId);
    //将response中的一些集群信息添加到configuration中；
    setClusterEntrypointInfoToConfig(report);
    //返回RestClusterClient
    return () -> {   
                    return new RestClusterClient<>(flinkConfiguration, report.getApplicationId());
                 };
}    
```
#### 步骤5
核心方法实现：
```text
deployInternal(ClusterSpecification clusterSpecification, String applicationName, String yarnClusterEntrypoint,
    @Nullable JobGraph jobGraph, boolean detached) throws Exception {
        //<font color=#FF0000>这里会通过yarnclient rpc去查到numYarnMaxVcores，然后根据配置判断yarn集群目前cpu核数资源是否足够，同时也有hadoop和yarn配置文件的校验；</font>
        isReadyForDeployment(ClusterSpecification clusterSpecification);
        //确认指定的queue是否ok；
        checkYarnQueues(yarnClient);
        //通过yarnclient创建application,这里会走到rmClient.getNewApplication方法中， 具体其实得看server端的逻辑，这个实现在RMClientService中，
        //主要就是两件事：获取ApplicationId跟查询资源的上下限；
        final YarnClientApplication yarnApplication = yarnClient.createApplication();
        final GetNewApplicationResponse appResponse = yarnApplication.getNewApplicationResponse();
        Resource maxRes = appResponse.getMaximumResourceCapability();
        //yarnclient去获取yarn上各个节点的容量内存和使用内存，计算出可用内存；
        freeClusterMem = getCurrentFreeClusterResources(yarnClient);
        //基于获取到的yarn集群资源信息和之前定义的clusterSpecification进行比对，确认是否满足资源条件，并调整clusterSpecification相应参数。
        validClusterSpecification = validateClusterResources(clusterSpecification, yarnMinAllocationMB, maxRes, freeClusterMem);
        /**
        ApplicationReport report = startAppMaster(
                                flinkConfiguration,
                                applicationName,
                                yarnClusterEntrypoint,
                                jobGraph,
                                yarnClient,
                                yarnApplication,
                                validClusterSpecification);
        
    }
        
```

