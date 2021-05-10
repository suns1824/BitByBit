jmap -heap/-histo/-dump    内存   
jstack    线程   
jstat -gc        
jconsole/arthas  



top/iostat   



cms：以获取最短回收停顿时间为目标、基于标记清除、并发收集，缺点：对CPU资源敏感、无法处理浮动垃圾、有空间碎片。    
g1：空间整合、可预测停顿。