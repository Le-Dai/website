---
slug: apache-flink-important-concepts-summary
title: Apache Flink 重要知识点总结
authors: [ledai]
tags: [Flink]
date: 2020-04-17
---

## Apache Flink 重要知识点总结
<!-- truncate -->

## Flink Data Source

### SourceFunction

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.addSource(new SourceFunction<Long>() { 
    private long count = 0L;
    private volatile boolean isRunning = true;
    public void run(SourceContext<Long> ctx) {
        while (isRunning && count < 1000) {
            // 通过collect将输入发送出去 
            ctx.collect(count);
            count++;
        }
    }
    public void cancel() {
        isRunning = false;
    }
}).print();
env.execute();
```



### ParallelSourceFunction 和 RichParallelSourceFunction

通过 SourceFunction 实现的数据源是不具有并行度的，即不支持在得到的 DataStream 上调用 `setParallelism(n)` 方法 如果需要实现并行输入流 则需要实现 ParallelSourceFunction 或 RichParallelSourceFunction 接口

![SourceFunction](https://github.com/wangzhiwubigdata/God-Of-BigData/raw/master/pictures/flink-RichParallelSourceFunction.png)

ParallelSourceFunction 直接继承自 ParallelSourceFunction，具有并行度的功能。RichParallelSourceFunction 则继承自 AbstractRichFunction，同时实现了 ParallelSourceFunction 接口，所以其除了具有并行度的功能外，还提供了额外的与生命周期相关的方法，如 open() ，closen() 。  



## Flink 物理分区

### Random partitioning [DataStream → DataStream]

随机分区 (Random partitioning) 用于随机的将数据分布到所有下游分区中，通过 shuffle 方法来进行实现：

```
dataStream.shuffle();
```

### Rebalancing [DataStream → DataStream]

Rebalancing 采用轮询的方式将数据进行分区，其适合于存在数据倾斜的场景下，通过 rebalance 方法进行实现：

```
dataStream.rebalance();
```

### Rescaling [DataStream → DataStream]

当采用 Rebalancing 进行分区平衡时，其实现的是全局性的负载均衡，数据会通过网络传输到其他节点上并完成分区数据的均衡。 而 Rescaling 则是低配版本的 rebalance，它不需要额外的网络开销，它只会对上下游的算子之间进行重新均衡，通过 rescale 方法进行实现：

```
dataStream.rescale();
```

ReScale 这个单词具有重新缩放的意义，其对应的操作也是如此，具体如下：如果上游 operation 并行度为 2，而下游的 operation 并行度为 6，则其中 1 个上游的 operation 会将元素分发到 3 个下游 operation，另 1 个上游 operation 则会将元素分发到另外 3 个下游 operation。反之亦然，如果上游的 operation 并行度为 6，而下游 operation 并行度为 2，则其中 3 个上游 operation 会将元素分发到 1 个下游 operation，另 3 个上游 operation 会将元素分发到另外 1 个下游operation：

### Broadcasting [DataStream → DataStream]

将数据分发到所有分区上。通常用于小数据集与大数据集进行关联的情况下，此时可以将小数据集广播到所有分区上，避免频繁的跨分区关联，通过 broadcast 方法进行实现：

```
dataStream.broadcast();
```

###Custom partitioning [DataStream → DataStream]

```java
DataStreamSource<Tuple2<String, Integer>> streamSource = env.fromElements(new Tuple2<>("Hadoop", 1),
                new Tuple2<>("Spark", 1),
                new Tuple2<>("Flink-streaming", 2),
                new Tuple2<>("Flink-batch", 4),
                new Tuple2<>("Storm", 4),
                new Tuple2<>("HBase", 3));
streamSource.partitionCustom(new Partitioner<String>() {
    @Override
    public int partition(String key, int numPartitions) {
        // 将第一个字段包含flink的Tuple2分配到同一个分区
        return key.toLowerCase().contains("flink") ? 0 : 1;
    }
}, 0).print();
```



## 任务链和资源组

Flink 提供的底层 API 用户自己控制任务链 与 资源  

### startNewChain

startNewChain 用于基于当前 operation 开启一个新的任务链。如下所示，基于第一个 map 开启一个新的任务链，此时前一个 map 和 后一个 map 将处于同一个新的任务链中，但它们与 filter 操作则分别处于不同的任务链中：

```
someStream.filter(...).map(...).startNewChain().map(...);
```

### disableChaining

disableChaining 操作用于禁止将其他操作与当前操作放置于同一个任务链中，示例如下：

```
someStream.map(...).disableChaining();
```

### slotSharingGroup

slot 是任务管理器 (TaskManager) 所拥有资源的固定子集，每个操作 (operation) 的子任务 (sub task) 都需要获取 slot 来执行计算，但每个操作所需要资源的大小都是不相同的，为了更好地利用资源，Flink 允许不同操作的子任务被部署到同一 slot 中。slotSharingGroup 用于设置操作的 slot 共享组 (slot sharing group) ，Flink 会将具有相同 slot 共享组的操作放到同一个 slot 中 。示例如下：

```
someStream.filter(...).slotSharingGroup("slotSharingGroupName");
```



## Flink Windows

### Time Windows

滚动窗口 (Tumbling Windows) 是指彼此之间没有重叠的窗口。例如：每隔1小时统计过去1小时内的商品点击量，那么 1 天就只能分为 24 个窗口，每个窗口彼此之间是不存在重叠的，具体如下：  

![Tumbling](https://github.com/wangzhiwubigdata/God-Of-BigData/raw/master/pictures/flink-tumbling-windows.png)

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 接收socket上的数据输入
DataStreamSource<String> streamSource = env.socketTextStream("hadoop001", 9999, "\n", 3);
streamSource.flatMap(new FlatMapFunction<String, Tuple2<String, Long>>() {
    @Override
    public void flatMap(String value, Collector<Tuple2<String, Long>> out) throws Exception {
        String[] words = value.split("\t");
        for (String word : words) {
            out.collect(new Tuple2<>(word, 1L));
        }
    }
}).keyBy(0).timeWindow(Time.seconds(3)).sum(1).print(); //每隔3秒统计一次每个单词出现的数量
env.execute("Flink Streaming");
```

### Sliding Windows

滑动窗口用于滚动进行聚合分析，例如：每隔 6 分钟统计一次过去一小时内所有商品的点击量，那么统计窗口彼此之间就是存在重叠的，即 1天可以分为 240 个窗口。图示如下：



![Sliding](https://github.com/wangzhiwubigdata/God-Of-BigData/raw/master/pictures/flink-sliding-windows.png)

可以看到 window 1 - 4 这四个窗口彼此之间都存在着时间相等的重叠部分。想要实现滑动窗口，只需要在使用 timeWindow 方法时额外传递第二个参数作为滚动时间即可，具体如下：

```java
// 每隔3秒统计一次过去1分钟内的数据
timeWindow(Time.minutes(1),Time.seconds(3))
```



### Session Windows

当用户在进行持续浏览时，可能每时每刻都会有点击数据，例如在活动区间内，用户可能频繁的将某类商品加入和移除购物车，而你只想知道用户本次浏览最终的购物车情况，此时就可以在用户持有的会话结束后再进行统计。想要实现这类统计，可以通过 Session Windows 来进行实现。

![Session](https://github.com/wangzhiwubigdata/God-Of-BigData/raw/master/pictures/flink-session-windows.png)

具体的实现代码如下：

```java
// 以处理时间为衡量标准，如果10秒内没有任何数据输入，就认为会话已经关闭，此时触发统计
window(ProcessingTimeSessionWindows.withGap(Time.seconds(10)))
// 以事件时间为衡量标准    
window(EventTimeSessionWindows.withGap(Time.seconds(10)))
```



### Global Windows

最后一个窗口是全局窗口， 全局窗口会将所有 key 相同的元素分配到同一个窗口中，其通常配合触发器 (trigger) 进行使用。如果没有相应触发器，则计算将不会被执行。

![Global](https://github.com/wangzhiwubigdata/God-Of-BigData/raw/master/pictures/flink-non-windowed.png)

这里继续以上面词频统计的案例为例，示例代码如下：

```java
// 当单词累计出现的次数每达到10次时，则触发计算，计算整个窗口内该单词出现的总数
window(GlobalWindows.create()).trigger(CountTrigger.of(10)).sum(1).print();
```

### Count Windows

Count Windows 用于以数量为维度来进行数据聚合，同样也分为滚动窗口和滑动窗口，实现方式也和时间窗口完全一致，只是调用的 API 不同，具体如下：

```java
// 滚动计数窗口，每1000次点击则计算一次
countWindow(1000)
// 滑动计数窗口，每10次点击发生后，则计算过去1000次点击的情况
countWindow(1000,10)
```

```java
public WindowedStream<T, KEY, GlobalWindow> countWindow(long size) {
    return window(GlobalWindows.create()).trigger(PurgingTrigger.of(CountTrigger.of(size)));
}
public WindowedStream<T, KEY, GlobalWindow> countWindow(long size, long slide) {
    return window(GlobalWindows.create())
        .evictor(CountEvictor.of(size))
        .trigger(CountTrigger.of(slide));
}
```

  

## Flink 状态管理

### Keyed State

键控状态 (Keyed State) ：是一种特殊的算子状态，即状态是根据 key 值进行区分的，Flink 会为每类键值维护一个状态实例。如下图所示，每个颜色代表不同 key 值，对应四个不同的状态实例。需要注意的是键控状态只能在 `KeyedStream` 上进行使用，我们可以通过 `stream.keyBy(...)` 来得到 `KeyedStream` 。

![Global](https://github.com/wangzhiwubigdata/God-Of-BigData/raw/master/pictures/flink-operator-state.png)

- **ValueState**：存储单值类型的状态。可以使用 `update(T)` 进行更新，并通过 `T value()` 进行检索。

- **ListState**：存储列表类型的状态。可以使用 `add(T)` 或 `addAll(List)` 添加元素；并通过 `get()` 获得整个列表。

- **ReducingState**：用于存储经过 ReduceFunction 计算后的结果，使用 `add(T)` 增加元素。

- **AggregatingState**：用于存储经过 AggregatingState 计算后的结果，使用 `add(IN)` 添加元素。

- **FoldingState**：已被标识为废弃，会在未来版本中移除，官方推荐使用 `AggregatingState` 代替。

- **MapState**：维护 Map 类型的状态。

```java
public class ThresholdWarning extends 
    RichFlatMapFunction<Tuple2<String, Long>, Tuple2<String, List<Long>>> {

    // 通过ListState来存储非正常数据的状态
    private transient ListState<Long> abnormalData;
    // 需要监控的阈值
    private Long threshold;
    // 触发报警的次数
    private Integer numberOfTimes;

    ThresholdWarning(Long threshold, Integer numberOfTimes) {
        this.threshold = threshold;
        this.numberOfTimes = numberOfTimes;
    }

    @Override
    public void open(Configuration parameters) {
        // 通过状态名称(句柄)获取状态实例，如果不存在则会自动创建
        abnormalData = getRuntimeContext().getListState(
            new ListStateDescriptor<>("abnormalData", Long.class));
    }

    @Override
    public void flatMap(Tuple2<String, Long> value, Collector<Tuple2<String, List<Long>>> out)
        throws Exception {
        Long inputValue = value.f1;
        // 如果输入值超过阈值，则记录该次不正常的数据信息
        if (inputValue >= threshold) {
            abnormalData.add(inputValue);
        }
        ArrayList<Long> list = Lists.newArrayList(abnormalData.get().iterator());
        // 如果不正常的数据出现达到一定次数，则输出报警信息
        if (list.size() >= numberOfTimes) {
            out.collect(Tuple2.of(value.f0 + " 超过指定阈值 ", list));
            // 报警信息输出后，清空状态
            abnormalData.clear();
        }
    }
}
```

### 状态有效期

以上任何类型的 keyed state 都支持配置有效期 (TTL) ，示例如下：

```java
StateTtlConfig ttlConfig = StateTtlConfig
    // 设置有效期为 10 秒
    .newBuilder(Time.seconds(10))  
    // 设置有效期更新规则，这里设置为当创建和写入时，都重置其有效期到规定的10秒
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite) 
    /*设置只要值过期就不可见，另外一个可选值是ReturnExpiredIfNotCleanedUp，
     代表即使值过期了，但如果还没有被物理删除，就是可见的*/
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();
ListStateDescriptor<Long> descriptor = new ListStateDescriptor<>("abnormalData", Long.class);
descriptor.enableTimeToLive(ttlConfig);
```



### Operator State

算子状态 (Operator State)：状态是和算子进行绑定的，一个算子的状态不能被其他算子所访问到。官方文档上对 Operator State 的解释是：*each operator state is bound to one parallel operator instance*，所以更为确切的说一个算子状态是与一个并发的算子实例所绑定的，即假设算子的并行度是 2，那么其应有两个对应的算子状态：

![Global](https://github.com/wangzhiwubigdata/God-Of-BigData/raw/master/pictures/flink-operator-state.png)

- **ListState**：存储列表类型的状态。
- **UnionListState**：存储列表类型的状态，与 ListState 的区别在于：如果并行度发生变化，ListState 会将该算子的所有并发的状态实例进行汇总，然后均分给新的 Task；而 UnionListState 只是将所有并发的状态实例汇总起来，具体的划分行为则由用户进行定义。
- **BroadcastState**：用于广播的算子状态。

```java
public class ThresholdWarning extends RichFlatMapFunction<Tuple2<String, Long>, 
Tuple2<String, List<Tuple2<String, Long>>>> implements CheckpointedFunction {

    // 非正常数据
    private List<Tuple2<String, Long>> bufferedData;
    // checkPointedState
    private transient ListState<Tuple2<String, Long>> checkPointedState;
    // 需要监控的阈值
    private Long threshold;
    // 次数
    private Integer numberOfTimes;

    ThresholdWarning(Long threshold, Integer numberOfTimes) {
        this.threshold = threshold;
        this.numberOfTimes = numberOfTimes;
        this.bufferedData = new ArrayList<>();
    }

    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        // 注意这里获取的是OperatorStateStore
        checkPointedState = context.getOperatorStateStore().
            getListState(new ListStateDescriptor<>("abnormalData",
                TypeInformation.of(new TypeHint<Tuple2<String, Long>>() {
                })));
        // 如果发生重启，则需要从快照中将状态进行恢复
        if (context.isRestored()) {
            for (Tuple2<String, Long> element : checkPointedState.get()) {
                bufferedData.add(element);
            }
        }
    }

    @Override
    public void flatMap(Tuple2<String, Long> value, 
                        Collector<Tuple2<String, List<Tuple2<String, Long>>>> out) {
        Long inputValue = value.f1;
        // 超过阈值则进行记录
        if (inputValue >= threshold) {
            bufferedData.add(value);
        }
        // 超过指定次数则输出报警信息
        if (bufferedData.size() >= numberOfTimes) {
             // 顺便输出状态实例的hashcode
             out.collect(Tuple2.of(checkPointedState.hashCode() + "阈值警报！", bufferedData));
            bufferedData.clear();
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        // 在进行快照时，将数据存储到checkPointedState
        checkPointedState.clear();
        for (Tuple2<String, Long> element : bufferedData) {
            checkPointedState.add(element);
        }
    }
}
```



### CheckPoints

为了使 Flink 的状态具有良好的容错性，Flink 提供了检查点机制 (CheckPoints) 。通过检查点机制，Flink 定期在数据流上生成 checkpoint barrier ，当某个算子收到 barrier 时，即会基于当前状态生成一份快照，然后再将该 barrier 传递到下游算子，下游算子接收到该 barrier 后，也基于当前状态生成一份快照，依次传递直至到最后的 Sink 算子上。当出现异常后，Flink 就可以根据最近的一次的快照数据将所有算子恢复到先前的状态。

![CheckPoints](https://github.com/wangzhiwubigdata/God-Of-BigData/raw/master/pictures/flink-stream-barriers.png)

```java
// 开启检查点机制，并指定状态检查点之间的时间间隔
env.enableCheckpointing(1000); 

// 其他可选配置如下：
// 设置语义
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
// 设置两个检查点之间的最小时间间隔
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);
// 设置执行Checkpoint操作时的超时时间
env.getCheckpointConfig().setCheckpointTimeout(60000);
// 设置最大并发执行的检查点的数量
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
// 将检查点持久化到外部存储
env.getCheckpointConfig().enableExternalizedCheckpoints(
    ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
// 如果有更近的保存点时，是否将作业回退到该检查点
env.getCheckpointConfig().setPreferCheckpointForRecovery(true);
```

保存点机制 (Savepoints) 是检查点机制的一种特殊的实现，它允许你通过手工的方式来触发 Checkpoint，并将结果持久化存储到指定路径中，主要用于避免 Flink 集群在重启或升级时导致状态丢失。示例如下：

```java
# 触发指定id的作业的Savepoint，并将结果存储到指定目录下
bin/flink savepoint :jobId [:targetDirectory]
```



### 状态管理器分类

#### 1. MemoryStateBackend

默认的方式，即基于 JVM 的堆内存进行存储，主要适用于本地开发和调试。

#### 2. FsStateBackend

基于文件系统进行存储，可以是本地文件系统，也可以是 HDFS 等分布式文件系统。 需要注意而是虽然选择使用了 FsStateBackend ，但正在进行的数据仍然是存储在 TaskManager 的内存中的，只有在 checkpoint 时，才会将状态快照写入到指定文件系统上。

#### 3. RocksDBStateBackend

RocksDBStateBackend 是 Flink 内置的第三方状态管理器，采用嵌入式的 key-value 型数据库 RocksDB 来存储正在进行的数据。等到 checkpoint 时，再将其中的数据持久化到指定的文件系统中，所以采用 RocksDBStateBackend 时也需要配置持久化存储的文件系统。之所以这样做是因为 RocksDB 作为嵌入式数据库安全性比较低，但比起全文件系统的方式，其读取速率更快；比起全内存的方式，其存储空间更大，因此它是一种比较均衡的方案。



## Apache Flink的时间类型

- ProcessingTime

是数据流入到具体某个算子时候相应的系统时间。ProcessingTime 有最好的性能和最低的延迟。但在分布式计算环境中ProcessingTime具有不确定性，相同数据流多次运行有可能产生不同的计算结果。

- IngestionTime

IngestionTime是数据进入Apache Flink框架的时间，是在Source Operator中设置的。与ProcessingTime相比可以提供更可预测的结果，因为IngestionTime的时间戳比较稳定(在源处只记录一次)，同一数据在流经不同窗口操作时将使用相同的时间戳，而对于ProcessingTime同一数据在流经不同窗口算子会有不同的处理时间戳。

- EventTime

EventTime是事件在设备上产生时候携带的。在进入Apache Flink框架之前EventTime通常要嵌入到记录中，并且EventTime也可以从记录中提取出来。在实际的网上购物订单等业务场景中，大多会使用EventTime来进行数据计算。



## Watermark

### Watermark的产生方式

目前Apache Flink 有两种生产Watermark的方式，如下：

- Punctuated - 数据流中每一个递增的EventTime都会产生一个Watermark。

在实际的生产中Punctuated方式在TPS很高的场景下会产生大量的Watermark在一定程度上对下游算子造成压力，所以只有在实时性要求非常高的场景才会选择Punctuated的方式进行Watermark的生成。

- Periodic - 周期性的（一定时间间隔或者达到一定的记录条数）产生一个Watermark。在实际的生产中Periodic的方式必须结合时间和积累条数两个维度继续周期性产生Watermark，否则在极端情况下会有很大的延时。

所以Watermark的生成方式需要根据业务场景的不同进行不同的选择。



### Watermark的接口定义	

对应Apache Flink Watermark两种不同的生成方式

Periodic Watermarks - AssignerWithPeriodicWatermarks

Punctuated Watermarks - AssignerWithPunctuatedWatermarks

AssignerWithPunctuatedWatermarks 继承了TimestampAssigner接口 -TimestampAssigner



### Watermark解决什么问题

Watermark生成接口和Apache Flink内部对Periodic Watermark的实现来看，Watermark的时间戳可以和Event中的EventTime 一致，也可以自己定义任何合理的逻辑使得Watermark的时间戳不等于Event中的EventTime，Event中的EventTime自产生那一刻起就不可以改变了，不受Apache Flink框架控制，而Watermark的产生是在Apache Flink的Source节点或实现的Watermark生成器计算产生(如上Apache Flink内置的 Periodic Watermark实现), Apache Flink内部对单流或多流的场景有统一的Watermark处理。

如何将迟来的EventTime 位11的元素正确处理。要解决这个问题我们还需要先了解一下EventTime window是如何触发的？ EventTime window 计算条件是当Window计算的Timer时间戳 小于等于 当前系统的Watermak的时间戳时候进行计算。



## 分布式缓存

Flink提供了一个分布式缓存 可以使用户在并行函数中很方便的读取本地文件，并把它放在taskmanager节点中，防止task重复拉取。 此缓存的工作机制如下：程序注册一个文件或者目录(本地或者远程文件系统，例如hdfs或者s3)，通过ExecutionEnvironment注册缓存文件并为它起一个名称。 当程序执行，Flink自动将文件或者目录复制到所有taskmanager节点的本地文件系统，仅会执行一次。用户可以通过这个指定的名称查找文件或者目录，然后从taskmanager节点的本地文件系统访问它。

```java
//获取运行环境
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();


//1：注册一个文件,可以使用hdfs上的文件 也可以是本地文件进行测试
env.registerCachedFile("/Users/wangzhiwu/WorkSpace/quickstart/text","a.txt");
```

```java
DataSet<String> result = data.map(new RichMapFunction<String, String>() {
            private ArrayList<String> dataList = new ArrayList<String>();

            @Override
            public void open(Configuration parameters) throws Exception {
                super.open(parameters);
                //2：使用文件
                File myFile = getRuntimeContext().getDistributedCache().getFile("a.txt");
                List<String> lines = FileUtils.readLines(myFile);
                for (String line : lines) {
                    this.dataList.add(line);
                    System.err.println("分布式缓存为:" + line);
                }
            }

            @Override
            public String map(String value) throws Exception {
                //在这里就可以使用dataList
                System.err.println("使用datalist：" + dataList + "------------" +value);
                //业务逻辑
                return dataList +"：" +  value;
            }
        });

        result.printToErr();
    }
```



## 广播变量

在Flink中，同一个算子可能存在若干个不同的并行实例，计算过程可能不在同一个Slot中进行，不同算子之间更是如此，因此不同算子的计算数据之间不能像Java数组之间一样互相访问，而广播变量Broadcast便是解决这种情况的。

我们可以把广播变量理解为是一个公共的共享变量，我们可以把一个dataset 数据集广播出去，然后不同的task在节点上都能够获取到，这个数据在每个节点上只会存在一份



```java
1：初始化数据
  DataSet<Integer> num = env.fromElements(1, 2, 3)
  2：广播数据
  .withBroadcastSet(toBroadcast, "num");
  3：获取数据
  Collection<Integer> broadcastSet = getRuntimeContext().getBroadcastVariable("num");
  
  注意：
  1：广播出去的变量存在于每个节点的内存中，所以这个数据集不能太大。因为广播出去的数据，会常驻内存，除非程序执行结束
  2：广播变量在初始化广播出去以后不支持修改，这样才能保证每个节点的数据都是一致的。
```

### 注意事项

### 使用广播状态，task 之间不会相互通信

只有广播的一边可以修改广播状态的内容。用户必须保证所有 operator 并发实例上对广播状态的 修改行为都是一致的。或者说，如果不同的并发实例拥有不同的广播状态内容，将导致不一致的结果。

### 广播状态中事件的顺序在各个并发实例中可能不尽相同

广播流的元素保证了将所有元素（最终）都发给下游所有的并发实例，但是元素的到达的顺序可能在并发实例之间并不相同。因此，对广播状态的修改不能依赖于输入数据的顺序。

### 所有operator task都会快照下他们的广播状态

在checkpoint时，所有的 task 都会 checkpoint 下他们的广播状态，随着并发度的增加，checkpoint 的大小也会随之增加

### 广播变量存在内存中

广播出去的变量存在于每个节点的内存中，所以这个数据集不能太大，百兆左右可以接受，Gb不能接受



```java
public class BroadCastTest {

    public static void main(String[] args) throws Exception{
        ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
        //1.封装一个DataSet
        DataSet<Integer> broadcast = env.fromElements(1, 2, 3);
        DataSet<String> data = env.fromElements("a", "b");
        data.map(new RichMapFunction<String, String>() {
            private List list = new ArrayList();
            @Override
            public void open(Configuration parameters) throws Exception {
                // 3. 获取广播的DataSet数据 作为一个Collection
                Collection<Integer> broadcastSet = getRuntimeContext().getBroadcastVariable("number");
                list.addAll(broadcastSet);
            }

            @Override
            public String map(String value) throws Exception {
                return value + ": "+ list;
            }
        }).withBroadcastSet(broadcast, "number") 
            // 2. 广播的broadcast
          .printToErr();//打印到err方便查看
    }
}
```

