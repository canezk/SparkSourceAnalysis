### RDD
#### RDD基本属性
RDD(Resilient Distributed DataSet) [RDD Paper](http://101.96.8.164/www-bcf.usc.edu/~minlanyu/teach/csci599-fall12/papers/nsdi_spark.pdf)是Spark对一个不可变数据集合的抽象，他包含5个基本属性
- A list of partitions
  ```
  protected def getPartitions: Array[Partition]
  ```
- A function for computing each split(a given partition)
  ```
  def compute(split: Partition, context: TaskContext): Iterator[T]
  ```
- A list of dependencies on other RDDs
  ```
  def getDependencies: Seq[Dependency[_]] = deps
  ```
- Optionally, a Partitioner for key-value RDDs
  ```
  @transient val partitioner: Option[Partitioner] = None
  ```
- Optionally, a list of preferred locations to compute each split on
  ```
  protected def getPreferredLocations(split: Partition): Seq[String] = Nil
  ```

#### RDD其他属性
- SparkContext(Required)
  ```
  可以理解为一个容器，每个RDD都和一个sparkcontext实例关联。是spark应用程序和spark集群之间的唯一通信口
  ```
- Name(Optionally)
  ```
  给rdd设置一个名字，好处是在web ui可以更好的区分不同job的rdd
  ```
- WithScope
  ```
  类似于java的aop，用于spark ui更好的展示job的运行过程,具体jira是Spark-6943
  https://issues.apache.org/jira/browse/SPARK-6943
  ```
- SparkContext.clean
  ```
  闭包清理，在rdd各种transformation和action中使用，目的是为了去除unused的引用以及不能serializable的引用，
  减少网络io，同时避免rpc发送给worker的时候序列化失败。
  可以参考下面链接的讨论：
  https://www.quora.com/What-does-Closure-cleaner-func-mean-in-Spark
  ```
- StorageLevel
  ```
  RDD可以被持久化，避免多次重复运算，因为有些rdd是经过多次transformation操作得到的，重复运算可能会比较耗时
  ```
- CheckpointRdd
  ```
  注意和cache的区别，论文提到宽依赖比较适合做checkpoint,因为checkpoint成功之后会去除dependency，可以避免对所有的parent rdd做compute。
  这块值得思考一下。
  ```
 
#### 典型的Spark Job
一个Spark作业就是一个RRD的有向无环图，从已知的RDD 计算出未知RDD, 最后把RDD数据写到存储系统

![Spark Job](spark-job.png)
### 典型的RDD
#### HadoopRDD
从HDFS上读取文件
- partitions: 根据inputFormat.getSplits分割Paritition, HDFS文件按照Block划分
- Computing function: 返回从HDFS文件读取记录
- Dependencies: None
- Partitioner: None
- Preferred locations: Parition对应block在HDFS上的datanode locations

#### MapPartitionsRDD
RDD每一个记录上执行func操作

- partitions: 与父RDD一致
- Computing function: 先执行父RDD compute操作，在执行用户给定Func，生成新的记录
- Dependencies: OneToOneDependency
- Partitioner: None
- Preferred locations: checkpointed location or None

#### ShuffledRDD
ShuffleRDD 负责从依赖的staging拉取数据

- partitions: 一个shuffle id 一个partition
- Computing function: 通过shuffleManager 从依赖RRD拉取数据
- Dependencies: ShuffleDependency(依赖的RRD/Partitioner/Aggregator/Combiner/)
- Partitioner: ShuffleDependency的Partitioner
- Preferred locations: 根据拉去数据所在位置计算一个location

#### ParallelCollectionRDD
将集合划分为指定数目的partitions
- partitions: 一个partition对应原集合部分区间的数据
- Computing function: 读取原始记录
- Dependencies: None
- Partitioner: None
- Preferred locations: None

### RDD operation
RRD上操作分为两类：转换(Transformations) vs 执行(Actions)

- 转换(Transformations) (如：map, filter, groupBy, join等), 返回一个新的RDD，转换操作是Lazy的，也就是说从一个RDD转换生成另一个RDD的操作不是马上执行，Spark在遇到Transformations操作时只会记录需要这样的操作，并不会去执行，需要等到有Actions操作的时候才会真正启动计算过程进行计算
- 执行(Actions) (如：count, collect, save等)，Actions操作会返回结果或把RDD数据写到存储系统中。Actions是触发Spark启动计算的动因。

### 任务分解
```
当RDD遇到Actions操作时候，启动一个Spark Job,比如reduce action:(建议在ide里面单步调试examples下面的例子)
--> RDD#reduce
--> SparkConext#runJob
--> DAGScheduler#runJob/submitJob/handleJobSubmitted DAGScheduler处理作业
--> DAGScheduler#newResultStage(ShuffleMapStage/ResultStage)/ 将RRD依赖关系将作业分解为不同Stage(ShuffleMapStage/ResultStage)
--> DAGScheduler#submitStage(stage) 提交前置依赖stage为空或者已经全部执行成功的stage
--> DAGScheduler#submitMissingTasks 根据Stage的依赖关系，调度各个Stage。每个Stage根据partitions个数分为不同Task
--> TaskScheduler#submitTasks 提交Task
--> SchedulerBackend#reviveOffers 
```
![Spark Stage](spark-stage.png)

### 数据流

```
Executor-> launchTask
--> TaskRunner#run
--> Task(ResultTask/ShuffleMapTask)#run/runTask
--> RRD(HadoopRDD/MapPartitionsRDD/ShuffledRDD)#iterator/computeOrReadCheckpoint/compute
--> ShuffleMapTask:ShuffleWriter#write / ResultTask: func
```
- ResultTask 数据通过Func最后处理后返回结果或把RDD数据写到存储系统任务
- ShuffleMapTask 数据输出到shuffle Service，为接下来的Stage服务Task

Executor与Driver维持心跳
- Executor通过心跳，不断更新执行中的Task的metrics
- Driver上HeartbeatReceiver 不断接受Executor 判断Executor是否存活