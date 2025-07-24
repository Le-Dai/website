---
slug: elasticsearch-implementation-principles-summary
title: Elasticsearch 实现原理以及知识点小结
authors: [ledai]
tags: [学习历程, Elasticsearch]
date: 2018-07-30
---

<h1>Elasticsearch</h1>

<!-- truncate -->

<h2>ES的核心机制</h2>

<h3>主备机制</h3>
（1）index包含多个shard

（2）每个shard都是一个最小工作单元，承载部分数据，lucene实例，完整的建立索引和处理请求的能力

（3）增减节点时，shard会自动在nodes中负载均衡

（4）primary shard和replica shard，每个document肯定只存在于某一个primary shard以及其对应的replica shard中，不可能存在于多个primary shard

（5）replica shard是primary shard的副本，负责容错，以及承担读请求负载

（6）primary shard的数量在创建索引的时候就固定了，replica shard的数量可以随时修改

（7）primary shard的默认数量是5，replica默认是1，默认有10个shard，5个primary shard，5个replica shard

（8）primary shard不能和自己的replica shard放在同一个节点上（否则节点宕机，primary shard和副本都丢失，起不到容错的作用），但是可以和其他primary shard的replica shard放在同一个节点上

<h3>容错机制</h3>

Elasticsearch容错机制：master选举，replica容错，数据恢复。
（1）9 shard，3 node

（2）master node宕机，自动master选举，red

（3）replica容错：新master将replica提升为primary shard，yellow

（4）重启宕机node，master copy replica到该node，使用原有的shard并同步宕机后的修改，green

容错的步骤为：

a、原来node1是一个master，假如因为某种原因node1宕机了，node中是R0，R1和R2全部丢失。

b、进行master选举，自动选举另外一个node成为新的master，承担起master的责任来，例如现在全部选举了node2为master。

c、新的master，将丢失的primary shard的某个replica shard提升为primary  shard。此时集群的状态就会变为yellow，原因是虽然目前所有的primary  shard全部变成active，但是replice shard少了，也就并不是所有的replica  shard都是active。

d、重启故障的node1，新的master(node2)会将缺失的副本都copy一份到这个node1上面去，而且这个node1会使用之前已经有的shard数据，只是同步一下宕机之后发生的修改，此时，集群是status再次变回green。

<h2>Elasticsearch是如何实现Master选举的</h2>

1.Elasticsearch的选主是ZenDiscovery模块负责的，主要包含Ping（节点之间通过这个RPC来发现彼此）和Unicast（单播模块包含一个主机列表以控制哪些节点需要ping通）这两部分；

2.对所有可以成为master的节点（node.master: true）根据nodeId字典排序，每次选举每个节点都把自己所知道节点排一次序，然后选出第一个（第0位）节点，暂且认为它是master节点。

3.如果对某个节点的投票数达到一定的值（可以成为master节点数n/2+1）并且该节点自己也选举自己，那这个节点就是master。否则重新选举一直到满足上述条件。

<h2>Elasticsearch是如何避免脑裂现象的</h2>

当集群中master候选的个数不小于3个（node.master: true）。可以通过discovery.zen.minimum_master_nodes 
这个参数的设置来避免脑裂，设置为(N/2)+1。

这里node.master : true 是说明你是有资格成为master，并不是指你就是master。是皇子，不是皇帝。假如有10个皇子，这里应该设置为（10/2）+1=6，这6个皇子合谋做决策，选出新的皇帝。另外的4个皇子，即使他们全聚一起也才四个人，不足合谋的最低人数限制，他们不能选出新皇帝。

假如discovery.zen.minimum_master_nodes 设置的个数为5，有恰好有10个master备选节点，会出现什么情况呢？5个皇子组成一波，选一个皇帝出来，另外5个皇子也够了人数限制，他们也能选出一个皇帝来。此时一个天下两个皇帝，在es中就是脑裂。

假如集群master候选节点为2的时候，这种情况是不合理的，最好把另外一个node.master改成false。如果我们不改节点设置，还是套上面的(N/2)+1公式，此时discovery.zen.minimum_master_nodes应该设置为2。这就出现一个问题，两个master备选节点，只要有一个挂，就选不出master了。

我还是用皇子的例子来说明。假如先皇在位的时候规定，必须他的两个皇子都在的时候，才能从中2选1 继承皇位。万一有个皇子出意外挂掉了，就剩下一个皇子，天下不就没有新皇帝了么。


<h2>客户端在和集群连接时，如何选择特定的节点执行请求的？</h2>

TransportClient利用transport模块远程连接一个elasticsearch集群。它并不加入到集群中，只是简单的获得一个或者多个初始化的transport地址，并以 轮询 的方式与这些地址进行通信


<h2>Elasticsearch 文档索引过程描述</h2>

1.协调节点默认使用文档ID参与计算（也支持通过routing），以便为路由提供合适的分片。

`shard = hash(document_id) % (num_of_primary_shards)`

2.当分片所在的节点接收到来自协调节点的请求后，会将请求写入到Memory Buffer，然后定时（默认是每隔1秒）写入到Filesystem Cache，这个从Momery Buffer到Filesystem Cache的过程就叫做refresh；

3.当然在某些情况下，存在Momery Buffer和Filesystem Cache的数据可能会丢失，ES是通过translog的机制来保证数据的可靠性的。其实现机制是接收到请求后，同时也会写入到translog中，当Filesystem cache中的数据写入到磁盘中时，才会清除掉，这个过程叫做flush。

4.在flush过程中，内存中的缓冲将被清除，内容被写入一个新段，段的fsync将创建一个新的提交点，并将内容刷新到磁盘，旧的translog将被删除并开始一个新的translog。

5.flush触发的时机是定时触发（默认30分钟）或者translog变得太大（默认为512M）时。

<h3>关于Lucene的segement（也就是上文中所说的段）的补充：</h3>
- Lucene索引是由多个段组成，段本身是一个功能齐全的倒排索引。
- 段是不可变的，允许Lucene将新的文档增量地添加到索引中，而不用从头重建索引。
- 对于每一个搜索请求而言，索引中的所有段都会被搜索，并且每个段会消耗CPU的时钟周、文件句柄和内存。这意味着段的数量越多，搜索性能会越低。
- 为了解决这个问题，Elasticsearch会合并小段到一个较大的段，提交新的合并段到磁盘，并删除那些旧的小段


<h2>Elasticsearch 文档更新和删除过程描述</h2>

- 删除和更新也都是写操作，但是Elasticsearch中的文档是不可变的，因此不能被删除或者改动以展示其变更；
- 磁盘上的每个段都有一个相应的.del文件。当删除请求发送后，文档并没有真的被删除，而是在.del文件中被标记为删除。该文档依然能匹配查询，但是会在结果中被过滤掉。当段合并时，在.del文件中被标记为删除的文档将不会被写入新段。
- 在新的文档被创建时，Elasticsearch会为该文档指定一个版本号，当执行更新时，旧版本的文档在.del文件中被标记为删除，新版本的文档被索引到一个新段。旧版本的文档依然能匹配查询，但是会在结果中被过滤掉。

<h2>Elasticsearch搜索的过程描述</h2>

- 搜索被执行成一个两阶段过程，我们称之为 Query Then Fetch
- 在初始查询阶段时，查询会广播到索引中每一个分片拷贝（主分片或者副本分片）。 每个分片在本地执行搜索并构建一个匹配文档的大小为 from + size 的优先队列。PS：在搜索的时候是会查询Filesystem Cache的，但是有部分数据还在Memory Buffer，所以搜索是近实时的。
- 每个分片返回各自优先队列中 所有文档的 ID 和排序值 给协调节点，它合并这些值到自己的优先队列中来产生一个全局排序后的结果列表。
- 接下来就是 取回阶段，协调节点辨别出哪些文档需要被取回并向相关的分片提交多个 GET 请求。每个分片加载并丰富文档，如果有需要的话，接着返回文档给协调节点。一旦所有的文档都被取回了，协调节点返回结果给客户端。
 补充：Query Then Fetch的搜索类型在文档相关性打分的时候参考的是本分片的数据，这样在文档数量较少的时候可能不够准确，DFS Query Then Fetch增加了一个预查询的处理，询问Term和Document frequency，这个评分更准确，但是性能会变差。
 
 <h2>在并发情况下，Elasticsearch如果保证读写一致？</h2>
 - 可以通过版本号使用乐观并发控制，以确保新版本不会被旧版本覆盖，由应用层来处理具体的冲突；
 - 另外对于写操作，一致性级别支持quorum/one/all，默认为quorum，即只有当大多数分片可用时才允许写操作。但即使大多数可用，也可能存在因为网络等原因导致写入副本失败，这样该副本被认为故障，分片将会在一个不同的节点上重建。
 - 对于读操作，可以设置replication为sync(默认)，这使得操作在主分片和副本分片都完成后才会返回；如果设置replication为async时，也可以通过设置搜索请求参数_preference为primary来查询主分片，确保文档是最新版本。
 
 <h2>Elasticsearch在部署时，对Linux的设置有哪些优化方法？</h2>
 
- 64 GB 内存的机器是非常理想的， 但是32 GB 和16 GB 机器也是很常见的。少于8 GB 会适得其反。
- 如果你要在更快的 CPUs 和更多的核心之间选择，选择更多的核心更好。多个内核提供的额外并发远胜过稍微快一点点的时钟频率。
- 如果你负担得起 SSD，它将远远超出任何旋转介质。 基于 SSD 的节点，查询和索引性能都有提升。如果你负担得起，SSD 是一个好的选择。
- 即使数据中心们近在咫尺，也要避免集群跨越多个数据中心。绝对要避免集群跨越大的地理距离。
- 请确保运行你应用程序的 JVM 和服务器的 JVM 是完全一样的。 在 Elasticsearch 的几个地方，使用 Java 的本地序列化。
- 通过设置gateway.recover_after_nodes、gateway.expected_nodes、gateway.recover_after_time可以在集群重启的时候避免过多的分片交换，这可能会让数据恢复从数个小时缩短为几秒钟。
- Elasticsearch 默认被配置为使用单播发现，以防止节点无意中加入集群。只有在同一台机器上运行的节点才会自动组成集群。最好使用单播代替组播。
- 不要随意修改垃圾回收器（CMS）和各个线程池的大小。
- 把你的内存的（少于）一半给 Lucene（但不要超过 32 GB！），通过ES_HEAP_SIZE 环境变量设置。
- 内存交换到磁盘对服务器性能来说是致命的。如果内存交换到磁盘上，一个 100 微秒的操作可能变成 10 毫秒。 再想想那么多 10 微秒的操作时延累加起来。 不难看出 swapping 对于性能是多么可怕。
- Lucene 使用了大量的文件。同时，Elasticsearch 在节点和 HTTP 客户端之间进行通信也使用了大量的套接字。 所有这一切都需要足够的文件描述符。你应该增加你的文件描述符，设置一个很大的值，如 64,000。

补充：索引阶段性能提升方法
- 使用批量请求并调整其大小：每次批量数据 5–15 MB 大是个不错的起始点。
- 段和段合并：Elasticsearch 默认值是 20 MB/s，对机械磁盘应该是个不错的设置。如果你用的是 SSD，可以考虑提高到 100–200 MB/s。如果你在做批量导入，完全不在意搜索，你可以彻底关掉合并限流。另外还可以增加 index.translog.flush_threshold_size 设置，从默认的 512 MB 到更大一些的值，比如 1 GB，这可以在一次清空触发的时候在事务日志里积累出更大的段。
- 如果你的搜索结果不需要近实时的准确度，考虑把每个索引的index.refresh_interval 改到30s。
- 如果你在做大批量导入，考虑通过设置index.number_of_replicas: 0 关闭副本。

<h2>对于GC方面，在使用Elasticsearch时要注意什么？</h2>
- SEE：https://elasticsearch.cn/article/32
- 倒排词典的索引需要常驻内存，无法GC，需要监控data node上segment memory增长趋势。
- 各类缓存，field cache, filter cache, indexing cache, bulk queue等等，要设置合理的大小，并且要应该根据最坏的情况来看heap是否够用，也就是各类缓存全部占满的时候，还有heap空间可以分配给其他任务吗？避免采用clear cache等“自欺欺人”的方式来释放内存。
- 避免返回大量结果集的搜索与聚合。确实需要大量拉取数据的场景，可以采用scan & scroll api来实现。
- cluster stats驻留内存并无法水平扩展，超大规模集群可以考虑分拆成多个集群通过tribe node连接。
- 想知道heap够不够，必须结合实际应用场景，并对集群的heap使用情况做持续的监控。