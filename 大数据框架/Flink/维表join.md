[DataStream模式做维表join四种方式](https://jishuin.proginn.com/p/763bfbd58048)   
[阿里云flink维表join用法](https://help.aliyun.com/document_detail/62506.html?spm=a2c4g.11186623.6.816.2630c2d7abW0mI)  
阿里云版支持period for system_time这样的语法标识维表，开源版没有，开源使用temporal join来实现维表join，FOR SYSTEM_TIME AS OF lefttable.proctime（左表的time字段比如ts as PROCTIME()）
这样的语法。就是借助lookupjoin的思想实现。   
