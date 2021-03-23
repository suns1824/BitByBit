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
    1.将配置和传入参数合并成新的全量Configuration;
    effectiveConfiguration.addAll(commandLineConfiguration);
    2.基于spi机制加载YarnClusterClientFactory，这是一个封装了对yarn cluster操作的工厂类;
    final ClusterClientFactory<ApplicationId> yarnClusterClientFactory =
                    clusterClientServiceLoader.getClusterClientFactory(effectiveConfiguration);
    3.创建YarnClusterDescriptor，其构造方法内有一个yarnclient对象
    final YarnClusterDescriptor yarnClusterDescriptor = 
                    (YarnClusterDescriptor)
                            yarnClusterClientFactory.createClusterDescriptor(effectiveConfiguration);
    4.Flink on yarn集群在创建(执行yarn-session.sh)后会产生一个applicationId，如果启动参数中夹带了这个值，说明已存在一个flink on yarn集群，不需要再启动；
    if (cmd.hasOption(applicationId.getOpt())) {
        4-1.这里发生了yarn的一次rpc调用，返回的applicationreport中包括了jobmanager的host和port等信息，基于此封装了一个RestClusterClient作为返回；
        clusterClientProvider = yarnClusterDescriptor.retrieve(yarnApplicationId);
    } else {
        4-2.根据configuration构造clusterSpecification，包括jobmanager的内存、taskmanager的内存和每个taskmanager的taskslot大小；
        final ClusterSpecification clusterSpecification =
                                    yarnClusterClientFactory.getClusterSpecification(
                                            effectiveConfiguration);
        5.启动一个yarn-session模式的Flink集群，下文重点讲述；
        clusterClientProvider =
                            yarnClusterDescriptor.deploySessionCluster(clusterSpecification);
        略
        ClusterClient<ApplicationId> clusterClient = clusterClientProvider.getClusterClient();
        yarnApplicationId = clusterClient.getClusterId();
        6.确认启动成功后，将重要信息封装成properties文件，并持久化至本地磁盘，最重要的信息为applicationId，将在之后的Flink Job提交时用于查找Cluster信息；
        writeYarnPropertiesFile(yarnApplicationId, dynamicPropertiesEncoded);
    }
    if (detached模式) {
    } else {
        7.创建一个yarn-session集群的monitor，这里加了一个hook，在集群退出时能够有所感知，做一些资源清理的工作；
        final YarnApplicationStatusMonitor yarnApplicationStatusMonitor = ......
    }
    8.运行交互式客户端，循环获取集群状态，并做相应处理；
    runInteractiveCli(yarnApplicationStatusMonitor, acceptInteractiveInput);   
    收尾工作，略。
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
        1.这里会通过yarnclient rpc去查到numYarnMaxVcores，然后根据配置判断yarn集群目前cpu核数资源是否足够，同时也有hadoop和yarn配置文件的校验；
        isReadyForDeployment(ClusterSpecification clusterSpecification);
        2.确认指定的queue是否ok；
        checkYarnQueues(yarnClient);
        3.通过yarnclient创建application,这里会走到rmClient.getNewApplication方法中， 具体其实得看server端的逻辑，
        这个实现在RMClientService中，主要就是两件事：获取ApplicationId跟查询资源的上下限；
        final YarnClientApplication yarnApplication = yarnClient.createApplication();
        final GetNewApplicationResponse appResponse = yarnApplication.getNewApplicationResponse();
        Resource maxRes = appResponse.getMaximumResourceCapability();
        4.通过yarnclient去获取yarn上各个节点的容量内存和使用内存，计算出可用内存；
        freeClusterMem = getCurrentFreeClusterResources(yarnClient);
        5.基于获取到的yarn集群资源信息和之前定义的clusterSpecification进行比对，确认是否满足资源条件，并调整clusterSpecification相应参数。
        validClusterSpecification = validateClusterResources(clusterSpecification, yarnMinAllocationMB, maxRes, freeClusterMem);
        6.启动ApplicationMaster：下文详细介绍；
        ApplicationReport report = startAppMaster(
                                flinkConfiguration,
                                applicationName,
                                yarnClusterEntrypoint,
                                jobGraph,
                                yarnClient,
                                yarnApplication,
                                validClusterSpecification);
        7.基于report更新flinkconfiguration；
        setClusterEntrypointInfoToConfig(report);
        8.返回restclusterclient
        return new RestClusterClient<>(flinkConfiguration, report.getApplicationId());
    }
        
```
启动ApplicationMaster进程可以说是启动yarn-session模式的核心，在Flink-yarn模块中代码量巨大，整体可以归结为以下三个步骤：
>*
>*
>*

```text
private ApplicationReport startAppMaster(
      Configuration configuration,
      String applicationName,
      String yarnClusterEntrypoint,
      JobGraph jobGraph,
      YarnClient yarnClient,
      YarnClientApplication yarnApplication,
      ClusterSpecification clusterSpecification) throws Exception {
   //初始化文件系统
   //根据配置获取homeDir,其路径为/user/${currentUser}，一般在hdfs上，表示jar、log配置上传的路径；
   final FileSystem fs = FileSystem.get(yarnConfiguration);
   final Path homeDir = fs.getHomeDirectory();
   ApplicationSubmissionContext appContext = yarnApplication.getApplicationSubmissionContext();
   //YarnClusterDescriptor的构造函数中有addShipFiles方法，也是依据flinkConfiguration来找寻的，
   //systemShipFiles表示会被上传到hdfs的文件 并且被添加到classpath中；
   for (File file : shipFiles) {
      systemShipFiles.add(file.getAbsoluteFile());
   }
   final String logConfigFilePath = configuration.getString(YarnConfigOptionsInternal.APPLICATION_LOG_CONFIG_FILE);
   if (logConfigFilePath != null) {
      systemShipFiles.add(new File(logConfigFilePath));
   }
   //将flink_home/lib下的文件添加到systemShipFiles、通过-yt指定的文件也在里面；
   addLibFoldersToShipFiles(systemShipFiles);
   //将flink_home/plugins 下的文件添加到shipOnlyFiles，shipOnlyFiles也会被上传到hdfs，但不会被添加到classpath；
   addPluginsFoldersToShipFiles(shipOnlyFiles);

   final ApplicationId appId = appContext.getApplicationId();

   // zk-ha相关处理和重试策略逻辑
   ......
  //用户jar，jobGraph为空的话表示这不是per-job模式，否则从jobGraph中将userJars set进去，JobGraph的构建后续介绍；
   final Set<File> userJarFiles = (jobGraph == null)
         ? Collections.emptySet()
         : jobGraph.getUserJars().stream().map(f -> f.toUri()).map(File::new).collect(Collectors.toSet());

   //需要缓存的本地文件上传到hdfs，一般使用在文件共享中
   if (jobGraph != null) {
      for (Map.Entry<String, DistributedCache.DistributedCacheEntry> entry : jobGraph.getUserArtifacts().entrySet()) {
         org.apache.flink.core.fs.Path path = new org.apache.flink.core.fs.Path(entry.getValue().filePath);
         //只上传本地文件
         if (!path.getFileSystem().isDistributedFS()) {
            Path localPath = new Path(path.getPath());
            Tuple2<Path, Long> remoteFileInfo =
               Utils.uploadLocalFileToRemote(fs, appId.toString(), localPath, homeDir, entry.getKey());
            jobGraph.setUserArtifactRemotePath(entry.getKey(), remoteFileInfo.f0.toString());
         }
      }
      jobGraph.writeUserArtifactEntriesToConfiguration();
   }
   //表示启动appMaster需要的资源文件，会从hdfs上下载
   final Map<String, LocalResource> localResources = new HashMap<>(2 + systemShipFiles.size() + userJarFiles.size());
   // 访问hdfs的安全设置
   final List<Path> paths = new ArrayList<>(2 + systemShipFiles.size() + userJarFiles.size());
   // 启动taskManager需要的资源文件
   StringBuilder envShipFileList = new StringBuilder();

   //将systemShipFiles、shipOnlyFiles、用户jar上传到hdfs
   ......
   //classpath 排序相关
   ......
   // classPathBuilder: 存放classpath的信息
   StringBuilder classPathBuilder = new StringBuilder();
   //构建classpath: shipFile-jar、user-jar、log4j、yaml配置文件
    ......
   //指定jobmanager的内存设置，堆内外和metaspace等；
   final JobManagerProcessSpec processSpec =
                JobManagerProcessUtils.processSpecFromConfigWithNewOptionToInterpretLegacyHeap(
                        flinkConfiguration, JobManagerOptions.TOTAL_PROCESS_MEMORY);
  //初始化ContainerLaunchContext，设置启动命令，最重要的就是启动哪个类，这里就是yarnClusterEntrypoint；
   final ContainerLaunchContext amContainer = setupApplicationMasterContainer(
         yarnClusterEntrypoint,
         hasLogback,
         hasLog4j,
         hasKrb5,
         clusterSpecification.getMasterMemoryMB());
   amContainer.setLocalResources(localResources);
   //配置环境变量参数到appMasterEnv中，在启动启动YarnJobClusterEntrypoint时用到，例如：classpath、hadoopUser、appId等;
   amContainer.setEnvironment(appMasterEnv);
   // 略：设置提交任务队列、yarn任务名称的配置信息
   //设置如果deployment失败的hook；
   Thread deploymentFailureHook = new DeploymentFailureHook(yarnApplication, yarnFilesDir);
   Runtime.getRuntime().addShutdownHook(deploymentFailureHook);
   //构建ApplicationSubmissionContext，下文的appContext，里面包括了上面的amContainer信息等；
   ......
   //通过yarnclient提交创建AM的请求，最终是会走到yarn server端，也就是在RMClientService中的submitApplication中，这里有个核心方法rmAppManager.submitApplication，
   //通过事件调度器对新建Application进行处理，也就是对application的信息保存，这里涉及到yarn中的状态机和事件模型，最终执行的是yarnClusterEntrypoint中的main方法；
   yarnClient.submitApplication(appContext);
   //略，不断获取启动AM的状态。
}
```   
在调用了startAppMaster方法之后，一个叫YarnApplicationMasterRunner的进程就被创建起来了，其运行在yarn的某个工作节点上，需要注意的是，这个进程
不能算是一个严格意义上的ApplicationMaster，它内部包含了两个组件：ApplicationMaster和JobManager。在启动AM的同时也会启动JobManager，因为JobManager
和AM在同一个进程中，它会把JobManager的地址写到HDFS上，TaskManager启动的时候会去下载这个文件获取JobManager地址和它进行后续的通信。
