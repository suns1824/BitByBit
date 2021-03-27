上文有个被一笔带过的地方：
```text
clusterClient.submitJob(jobGraph)
```
个人觉得这里有必要梳理一波，两点：
>* RestClusterClient是如何submit的？
>* submitJob后发生了什么(集群状态的变化、Flink执行层面的逻辑)？

todo：2021.3.28