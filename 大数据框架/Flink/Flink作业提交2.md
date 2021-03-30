上文有个被一笔带过的地方：
```text
clusterClient.submitJob(jobGraph)
```
个人觉得这里有必要梳理一波，两点：
>* RestClusterClient是如何submit的？
>* submitJob后发生了什么(集群状态的变化、Flink执行层面的逻辑)？

从这一篇开始，我决定写得宏观一点，风格自我一点。
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
                        //序列化stateHandle，然后将其写到zk上，不是state本身，因为state可能会比较大，zk一般都是寸kb级别的对象。
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
                //创建jobmaster，todo
                createJobManagerRunner(jobGraph, initializationTimestamp) {
                    
                }
            }
        }
    }
}
```
