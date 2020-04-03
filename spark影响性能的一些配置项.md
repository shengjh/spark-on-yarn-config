# 影响spark性能的部分配置项 

由于影响spark性能的设置有很多很多，涉及到各个方面，不同的部署方式和不同的任务需要的配置是不一样的，这里列出部分影响saprk性能的配置项，集群模式为spark-yarn。

## spark-yarn

总体来说，影响spark任务性能主要在于内存(memory)的分配和cpu核心(executor)的设置。

### yarn的配置
避免将100％的资源分配给YARN容器，因为主机需要一些资源来运行操作系统和Hadoop守护程序，一般留1~2个核给操作系统
1. yarn.nodemanager.resource.memory-mb控制每个主机上container使用的最大内存总和
2. yarn.nodemanager.resource.cpu-vcores控制每个主机上container使用的最大内核总数
3. yarn.scheduler.increment-allocation-mb yarn会在内存分配的时候进行向上舍入，这是舍入的单位大小


### spark的配置
#### yarn-client模式
在yarn client模式中，driver是运行在资源分配master节点，不需要分配核，但是yarn里面存在AppMaster(AM),可以为AM分配内存和核数

1. spark.executor.memoryOverhead属性值将添加到executor内存中，以确定每个executor对YARN的完整内存请求,堆外内存,executorMemory * 0.10, with minimum of 384
2. spark.yarn.am.memoryOverhead 

3. spark.yarn.am.memory (default 1G)

4. spark.yarn.am.cores (default 1)

5. spark.executor.cores/--num-executors (default 1)

6. spark.executor.memory/--executor-memory

7. hdfs在线程过大时存在问题，尽量设置executor.cores <=5

8. spark.default.parallelism 对于没有父rdd的操作，默认的分区数(处理rdd)

9. spark.memory.offHeap.size

10. spark.sql.shuffle.partitions(处理spqrk sql)

##### yarn-cluster模式
在yarn cluster模式中，driver时是运行在AppMaster的容器中

1. spark.executor.memoryOverhead属性值将添加到executor内存中，以确定每个executor对YARN的完整内存请求,堆外内存,executorMemory * 0.10, with minimum of 384
2. spark.driver.memoryOverhead

3. spark.driver.memory (default 1G)

4. spark.yarn.driver.cores (default 1)

5. spark.executor.cores/--num-executors (default 1)

6. spark.executor.memory/--executor-memory

7. hdfs在线程过大时存在问题，尽量设置executor.cores <=5

8. spark.default.parallelism 对于没有父rdd的操作，默认的分区数(处理rdd)

9. spark.memory.offHeap.size

10. spark.sql.shuffle.partitions(处理spqrk sql)


### 举例
#### yarn-client
物理上，2个节点，128GB内存，32核，搭建yarn-client mode
1. 每个节点预留给操作系统2个cpu，18GB内存，可以分配给yarn管理的则有30 core，110GB memory
2. yarn.scheduler.increment-allocation-mb = default 512M   
3. spark.yarn.am.memory = default 1G,
4. spark.yarn.am.memoryOverhead = max(384M, 1G * 0.1) = 512M(向上取整)
5. spark.yarn.am.cores = 2
6. --executor-cores = 3, 则每个节点有10个executor
7. --executor-memory 110 / 10 = 10GB，由于还要分配overHead memory，取10GB
8. spark.executor.memoryOverhead = max(384M, 7GB * 0.1)= 1GB
9. spark.default.parallelism = 57
因此，每个节点10个executor，2个节点共19个executor(AM占用一个executor)，则设置为
```shell
spark-submit --master yarn --deploy-mode client --num-executors 19 --executor-cores 3 --executor-memory 10g 
```



参考文档：
* https://blog.cloudera.com/how-to-tune-your-apache-spark-jobs-part-2/
* http://spark.apache.org/docs/latest/configuration.html
* http://spark.apache.org/docs/latest/running-on-yarn.html
* https://blog.csdn.net/agent_x/article/details/78668762