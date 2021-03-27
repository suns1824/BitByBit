**上一篇介绍了Flink集群如何在Yarn上启动，本篇着重介绍用户编写的sql或者jar如何提交到Flink集群。**
举个例子：一个提交作业的cli：
```text
./bin/flink run -yid <application_id> ./examples/batch/WordCount.jar
```
flink cli具体详情参见[click here](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/cli.html)  
查看fink脚本看到是调用了CliFrontEnd的main方法，run命令会走到CliFrontEnd的run方法里：
```text
run(String[] args) {
    final ProgramOptions programOptions = ProgramOptions.create(commandLine);
    //拿到job对应的jar的url
    final List<URL> jobJars = getJobJarAndDependencies(programOptions);
    //构建configuration
    final Configuration effectiveConfiguration =
                    getEffectiveConfiguration(activeCommandLine, commandLine, programOptions, jobJars);
    //构建packageprogram，核心逻辑在buildProgram里,见下文；
    final PackagedProgram program = getPackagedProgram(programOptions, effectiveConfiguration);
    try {
        //这里我们重点研究ClientUtils.executeProgram方法，见下文：
        executeProgram(effectiveConfiguration, program);
    } finally {
        program.deleteExtractedLibraries();
    }
}
```
#### buildProgram讲解
在理解runOptions解析的逻辑里，需要熟悉flink cli的参数，比如-c、-C、-j等，建议学习上文提及的flink cli文档（会有版本差异）:
```text
PackagedProgram buildProgram(final ProgramOptions runOptions, final Configuration configuration) {
    //解析程序参数，有点抽象，略，没咋看懂。。。问题不大
    String[] programArgs = runOptions.getProgramArgs();
    //解析-j
    String jarFilePath = runOptions.getJarFilePath();
    //解析-C, 指定usercodeclassloader对应的classpath
    List<URL> classpaths = runOptions.getClasspaths();
    //解析-c，这个一般是指用户jar的主类，不一定要指明，如果用户jar的META-INF下的MAINIFEST.MF文件里没有写mainclass，这里就需要指定。
    String entryPointClass = runOptions.getEntryPointClassName();
    //用户jar文件
    File jarFile = jarFilePath != null ? getJarFile(jarFilePath) : null;
    return PackagedProgram.newBuilder()....build();
}
```
#### executeProgram讲解
这个方法是用来job提交的核心方法，在这之前我们需要思考一个问题，Flink的集群模式包括local、Yarn、K8s、StandAlone等很多模式，那么Flink是如何
根据不同的模式进行任务提交的呢？这里有个概念叫PipelineExecutor，字面意思是PipelineExecutor，是client端生成JobGraph之后，将作业提交给
集群的重要环节。PipelineExecutor是一个顶级接口，不同的集群模式分别实现了这个接口。而PipelineExecutorFactory（同样也是顶级接口，不同的模式下有不同的实现）
则用于在不同的模式下创建不同的PipelineExecutor，提交作业。PipelineExecutorFactory表示的一个创建执行器工厂接口，PipelineExecutor 表示一个执行器接口，典型的工厂设计模式。  
那么factory是如何选择的呢，又是SPI，不得不说，Flink里到处都是SPI，略。   
扯这个原因是因为executeProgram方法的第一次参数就是PipelineExecutorServiceLoader，不多说，进入正题： 
```text
public static void executeProgram(
            PipelineExecutorServiceLoader executorServiceLoader,
            Configuration configuration,
            PackagedProgram program,
            boolean enforceSingleJobExecution,
            boolean suppressSysout)
            throws ProgramInvocationException {
    //基于用户jar和传入的classpath参数来获取usercodeClassloader，classloader是在创建PackagedProgram的时候声明的，
    //使用了ClientUtils.buildUserCodeClassLoader方法，获取Flink自定义的classloader，这里其实涉及到了Flink的类加载策略，可以说是打破了
    //双亲委派，有两种策略：parent first和child first.
    final ClassLoader userCodeClassLoader = program.getUserCodeClassLoader();
    //每个运行中的线程都有一个成员contextClassLoader，用来在运行时动态地载入其它类
    final ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();
    Thread.currentThread().setContextClassLoader(userCodeClassLoader);
    //设置ExecutionEnvironment，用于批处理场景，在执行用户代码时用到，呼应用户代码最开始的ExecutionEnvironment.getExecutionEnvironment；
    ContextEnvironment.setAsContext(
                    executorServiceLoader, configuration, userCodeClassLoader,
                    enforceSingleJobExecution, suppressSysout);
    ////设置StreamExecutionEnvironment，用于流式计算，在执行用户代码时用到，呼应用户代码的StreamExecutionEnvironment.getExecutionEnvironment；
    StreamContextEnvironment.setAsContext(
                    executorServiceLoader, configuration, userCodeClassLoader,
                    enforceSingleJobExecution, suppressSysout);
    //调用用户jar的main函数，用户代码的main函数里干了啥，我们接着往下分析
    try {
            program.invokeInteractiveModeForExecution();
        } finally {
            ContextEnvironment.unsetAsContext();
            StreamContextEnvironment.unsetAsContext();
        }
    } finally {
        //恢复cl
        Thread.currentThread().setContextClassLoader(contextClassLoader);
    }
}
```
事情没有就此结束，仔细一想，JobGraph在哪构建的呢？在用户代码的最后往往会有这么样一行代码：  
```text
env.execute();
```
这里的env就是上文在executeProgram方法中设置的ExecutionEnvironment或者StreamExecutionEnvironment。Flink流批一体我的理解是在Table/SQL层面是实现了，
而在Dataset和DataStream层面依旧没法做到完全一致。所以前者依旧有分析的必要。  
不过现在我们理解了一点：Flink是如何提交作业的。
但是对于另一问题，似乎还是不够深入：Flink是如何根据用户程序是构建JobGraph，是如何将JobGraph转成ExecutionGraph的呢。这个问题我们专门去研究一下，走进execute方法里：
分为两种场景：
>* 流处理
>* 批处理

### 批处理：
批处理场景的ContextEnvironment，它的execute方法里最终调用了ExecutionEnvironment的executeAsync方法：
```text
public JobClient executeAsync(String jobName) throws Exception {
    //构造Plan，这是批处理才会有的概念，对应流式处理中的StreamGraph，两者继承于Pipeline接口。
    final Plan plan = createProgramPlan(jobName);
    //上文提到的SPI在这里用上了，这里会拿到对应的PipelineExecutorFactory，然后get对应的PipelineExecutor；
    final PipelineExecutorFactory executorFactory = executorServiceLoader.getExecutorFactory(configuration);
    //重点来了，execute方法！主要包括了两个步骤：构建jobgraph和client提交jobgraph，这里其实就和上一篇中的部分内容呼应上了，本文后续细说。
    CompletableFuture<JobClient> jobClientFuture = executorFactory.getExecutor(configuration)
        .execute(plan, configuration, userClassloader);
    //下面内容不解释，略
}
```
Plan是对source、transform、sink如何衔接的描述，方便后续被PipelineExecutor执行，当然Flink对Plan做了执行计划的优化，生成了OptimizedPlan。
额外说一句，Flink SQL使用了Java CC而不是Antlr做SQL的解析，当然这是因为Flink采用了Calcite而不像Spark SQL自己写了一整套Catalyst。
与StreamGraph的构建基于StreamNode和StreamEdge不同的是，Plan的构建从Sink集合入手，采用递归方式获取输入端的Operator也就是逆向获取source来实现。
#### execute方法分析
这里其实执行的是AbstractSessionClusterExecutor类的execute方法，流批在同一集群模式下都是执行的同一方法，重点只是pipeline的不同：
```text
public CompletableFuture<JobClient> execute(
            @Nonnull final Pipeline pipeline,
            @Nonnull final Configuration configuration,
            @Nonnull final ClassLoader userCodeClassloader) {
    //基于pipeline/plan构建jobgraph；
    final JobGraph jobGraph = PipelineExecutorUtils.getJobGraph(pipeline, configuration);
    //构建clusterDescriptor、clusterClientProvider、clusterClient，有种似曾相识的感觉，第一篇中介绍过；
    try (final ClusterDescriptor<ClusterID> clusterDescriptor =
            clusterClientFactory.createClusterDescriptor(configuration)) {
        final ClusterID clusterID = clusterClientFactory.getClusterId(configuration);
        final ClusterClientProvider<ClusterID> clusterClientProvider =
                clusterDescriptor.retrieve(clusterID);
        ClusterClient<ClusterID> clusterClient = clusterClientProvider.getClusterClient();
    //提交job
        return clusterClient
                .submitJob(jobGraph)
                .thenApplyAsync()
                .thenApplyAsync()
                .whenComplete();
    }
}
```
### 流处理：
流处理对应的是StreamContextEnvironment，它的execute方法如下(将其调用栈平展开来描述)：  
```text
execute() {
    StreamGraph sg = getStreamGraph(jobName) {
        //略，自行阅读即可，StreamGraph由StreamNode和StreamEdge构成，StreamNode是StreamGraph中的节点，从Transformation转换而来，可以理解为一个算子。
        //StreamEdge表示边，连接两个StreamNode，包含了旁路输出、分区器、字段筛选输出等信息。StreamGraph的构建思想其实和上面的Plan构建类似，从Sink向前
        //追溯到SourceTransformation，针对某一类型的transformation，调用其transformXX方法，该方法会首先转换上游的Transformation进行递归转换，确保上游都已经完成
        //了转换，然后通过addOperator构造StreamNode，调用addEdge与上游transform连接，构造出StreamNode。
        StreamGraph streamGraph = getStreamGraphGenerator().setJobName(jobName).generate();
    }
    StreamExecutionEnvironment.executeAsync(sg) {
        //流式计算场景的executeAsync方法内容和上文批处理中一样，可以说是无差别，最终走到的execute方法也是一样，因为该方法接收的是一个pipeline，
        //而Plan和StreamGraph都继承于Pipeline接口。
        ......
    }
}
```
本文的重点不是去解释JobGraph是如何构建和提交后发生了什么。关于这些，有空细细分析一波。  
这篇文章写完后我觉得于我自己很有必要的是理一理Flink流批一体的实现。




