# ES 数据存储架构

## 1. ES 到底是个啥？和 Lucene 什么关系？

一句话版本：

> **Elasticsearch = 分布式的、对 Lucene 做了封装的搜索+分析引擎。**

> Lucene 负责“单机 + 单索引”的查和存，ES 负责“**集群、多机器、水平扩展、REST 接口**”。



大致分工：

- **Lucene**：底层库

  - 倒排索引结构（term → 哪些文档包含它）
  - 文本分词、打分（BM25）
  - 段（segment）结构、合并策略

  

- **Elasticsearch**：分布式框架

  - 集群管理（节点加入/离开、分片调度）
  - Index / Shard 抽象
  - 写入/查询在多节点之间的路由和汇总
  - 副本、高可用、滚动升级
  - REST API、聚合、ILM、data tiers 等

你可以理解为：**ES = Lucene + 分布式外壳 + 各种周边机制。**





## 2. ES 的“世界观”：集群 / 节点 / 索引 / 分片

先记住下面这几个层级（从大到小）：

1. **Cluster（集群）**

   - 一堆节点（Node）组成一个逻辑整体，对外只看成一个“服务”。

   

2. **Node（节点）**

   - 一台机器上的 ES 进程。
   - 节点上会存放多个 shard。

   

3. **Index（索引）**

   - 在 ES 里面，基本等价于“**一张大表**”或“一个业务数据集”，比如：orders-2025.11.14。
   - 查询基本都是针对 index 来写的。

   

4. **Shard（分片）**

   - Index 被切成多个 **Primary Shard**（主分片）+ 对应的 **Replica Shard**（副本分片）。
   - 每个 shard 底层就是一个 **Lucene 索引**，真正的倒排结构都在这里。

   

你可以脑补为：

> **Cluster** = 一堆机器

> 每台机器上跑一个 **Node**

> 每个 Node 上放了几个 **Shard**（Lucene 索引）

> 若干 Shard 一起组成一个 **Index**。





## 3. 底层存储：倒排索引 & segment

再往下看一层：**一个 shard（一个 Lucene 索引）里面是什么？**

核心概念只有两个：

1. **倒排索引（inverted index）**
   - 针对文本字段（title, content 等）：
   - 先分词成 term，然后建立结构：

```
term1 -> [docId3, docId7, docId100, ...]
term2 -> [docId1, docId3, ...]
```

- 搜索 “X AND Y” 就变成：
  - 取出 term X 的文档集合
  - 取出 term Y 的文档集合
  - 取交集，再按 BM25 等算法打分排序



1. **segment（段）**

   - Lucene 把索引数据分成多个**不可变**的小段：
     - 每次写入、刷新，会生成新的 segment
     - 老的 segment 只读，不会改
   - 后台有个 **merge 线程**，把小 segment 合并成大 segment，顺便清理删除标记。
   - 所以：**写入很快（append + 新 segment），删除/更新是“标记删除”，真正回收靠后续 merge。**

   

你可以这样理解一个 shard 的内部：

> shard = 一堆 segment（只读小“文件包”）

> 每个 segment 里有倒排索引、doc_values 等结构

> 搜索时要同时在所有 segment 上查，然后把结果 merge。





## 4. 写入一条文档，ES 到底做了哪些事？

从客户端视角，你就是：

```
POST /my_index/_doc
{
  "user": "zhang",
  "msg": "hello elasticsearch",
  "age": 27
}
```

**内部大致流程：**



1. **路由到某个 shard**
   - 协调节点（你打的那台节点）收到请求
   - 根据 _id or routing：

```
shardId = hash(routing) % number_of_shards
```

1. 找到这个 shard 的 **primary** 在哪个节点，把请求转过去

   

2. **写 translog + 写内存索引**

   

   - 在目标节点上：
     - 把这个写操作追加到 **translog（预写日志）** → 防止宕机丢数据
     - 同时把文档加入 Lucene 的内存缓冲区，准备将来刷新成 segment

   

3. **复制到副本（replica）**

   - primary 写成功后，会根据写一致性策略，将操作复制到相应的 replica shard 上
   - 副本也做同样的操作：translog + 内存 buffer

   

4. **refresh（刷新）**

   - 默认每隔 1s 左右（refresh_interval），ES 会触发一次 refresh：
     - 把内存 buffer 里的文档刷成一个新的 segment（文件）
     - 新 segment 打开后，这批文档就变成**可搜索**的
   - 所以 ES 是 **“近实时（NRT）搜索”**：写完不是立刻可见，一般延迟 ~1s 左右。

   

5. **flush（刷盘）**

   - 定期或 translog 太大时，触发 flush：
     - 把内存中的 segment 写实到磁盘
     - translog 做 checkpoint，可以裁剪旧的 translog
   - flush 是更“重”的操作，保证崩溃恢复时只需重放较少的 translog。

你现在可以先记住：

> 写入流程 = 路由 → primary 写 + translog → 复制到 replica → 定期刷新成 segment → 定期 flush 清理日志。



## 5. 搜索一条数据，在集群里的路径

当你发一个搜索：

```
GET /my_index/_search
{
  "query": { "match": { "msg": "hello" } }
}
```

内部大致是这样：

1. **找到所有相关的 shard**

   - 协调节点先根据索引的 shard 分布，列出这个 index 的所有 primary/replica shard。
   - 对每个逻辑 shard，会选一个实际 shard（primary 或 replica）执行查询。

   

2. **分发查询（fan-out）**

   - 协调节点把同一条 query 分发到多个节点上的 shard。

   - 每个 shard 在本地用 Lucene 的倒排索引做搜索：

     - 分词、查 term 的 posting list
     - 计算打分（BM25）
     - 选出本 shard 的 top N 结果

     

3. **汇总结果（merge）**

   - 各个 shard 把自己的 top N 回给协调节点。
   - 协调节点把所有 shard 的结果 merge 排序，取全局 top N。
   - 再根据需要执行高亮、聚合、排序等。

简化理解：

> 一次搜索 = 对所有 shard 做“**并行小搜索**”，然后再在协调节点做一次“小合并”。

> 分片越多、节点越多 → 并行度越大，但 fan-out 的开销也越大。





## 6. 副本、容错和一致性（大致感知就行）

ES 使用的是 **主从复制模型**：

- 每个分片有一个 **Primary**，若干 **Replica**

- 写请求必须先落到 primary，再复制到 replica

- 如果 primary 所在节点挂了：

  - 集群会在剩余节点中 **把一个 replica 提升为新的 primary**
  - 重新分配 shard，保证每个分片又有对应的副本数

  

一致性上：

- ES 默认是 **“最终一致” + “单分片上强一致（primary→replica）”**

- 你可以通过写参数控制：

  - replication / wait_for_active_shards 等

- 大致观念：

  - 写 OK = 至少 primary 成功写 + 一部分 replica 也写成功（取决于策略）
  - 读时从任意 shard（primary 或 replica）读，所以短时间内可能看到旧数据（除非你只从 primary 读）

  





## 7. 聚合、排序与 doc_values（知道有这么个结构即可）

ES 的 **聚合（aggregation）**、排序等操作主要依赖一种 **列式存储结构：doc_values**：

- 倒排索引适合“从 term 找 doc”
- 但聚合需要“**按字段取所有 doc 的值**”
  - 比如：按 age 分桶、按 price 求 avg
- doc_values 就是为字段建立的一种列式结构（可以理解成“字段 → 一个按 docId 排序的值数组”）

这样聚合时就不需要扫描原始 _source，能加速统计和排序。





## 8. 把这一大坨捋成一个简单的“心智模型”

你现在可以先只记住下面这几个关键点（大面）：

1. **整体架构**

   - ES 是基于 Lucene 的分布式搜索引擎。
   - 集群由多个节点组成，Index 被切成多个分片（shard），分布到节点上。

   

2. **存储结构**

   - shard 底层就是一个 Lucene 索引 → 里面是多段 segment（只读）、倒排索引、doc_values 等。
   - 写入新数据 → 生成新的 segment；删除/更新 → 标记删除，merge 时真正回收。

   

3. **写入流程**

   - 客户端 → 协调节点 → 路由到 primary shard → 写入 translog + 内存 → 复制到 replica
   - 周期性 refresh → 新写入变为可搜索；
   - 周期性 flush → translog 截断，数据更“稳”。

   **Flush（Lucene/ES 层）（物理存储）**：

   - 条件：RAM buffer 到一定大小 / 文档数阈值 / 手动 / 一些内部策略
   - 动作：把内存中的索引数据写成一个新的 segment（真正的磁盘文件），并做一次 commit
   - 作用：持久化、方便崩溃恢复，减少 translog

   **Refresh（ES 层）（对外可见）**：

   - 条件：refresh_interval 时间到了（默认 1s）或你手动调用 _refresh
   - 动作：打开一个新的 searcher，使“最近写入但还在内存里的数据/新 segment”对搜索可见
   - 作用：**控制“写完多久之后能被搜索看到”**

   

   

4. **查询流程**

   - 协调节点 fan-out 到所有相关 shard → 每个 shard 用 Lucene 搜索 → 返回局部 topN → 汇总、排序、聚合 → 返回结果。

   

5. **高可用 & 扩展**

   - 通过 primary/replica 分片和多节点，做到：
     - 数据冗余、节点故障可用
     - 水平扩展（加机器 → 分片分散出去，吞吐变大）
   - 分片数决定并行度与上限，主分片数创建后基本定死，副本数可调。





# 图解

## **1. 集群 → 节点 → 索引 → 分片 总体结构**

```
+==========================================================+
|                 Elasticsearch Cluster                    |
|                      (my-es-cluster)                     |
+==========================================================+
|                                                          |
|  +----------------+   +----------------+   +------------+ |
|  |    Node 1      |   |    Node 2      |   |   Node 3   | |
|  |  (data node)   |   | (data+coord)   |   | (data node)| |
|  +----------------+   +----------------+   +------------+ |
|  | Shards:        |   | Shards:        |   | Shards:    | |
|  |  logs_P0       |   |  logs_P1       |   |  logs_P2   | |
|  |  logs_R1       |   |  logs_R2       |   |  logs_R0   | |
|  |  ...           |   |  ...           |   |  ...       | |
|  +----------------+   +----------------+   +------------+ |
|                                                          |
+==========================================================+


Legend:
- Cluster：整个 ES 集群
- Node：集群中的一个节点（一个 ES 进程）
- logs_P0：索引 logs-2025.11.14 的 primary shard 0
- logs_R0：索引 logs-2025.11.14 的 replica shard 0
```

再把 **Index → Shards** 单独放大一下：

```
Index: logs-2025.11.14
(number_of_shards = 3, number_of_replicas = 1)

+---------------------------------------------------------+
|                 Index: logs-2025.11.14                  |
+---------------------------------------------------------+
|  Logical Shards:                                        |
|                                                         |
|   Shard 0  ---->  Primary P0   +--- Replica R0          |
|   Shard 1  ---->  Primary P1   +--- Replica R1          |
|   Shard 2  ---->  Primary P2   +--- Replica R2          |
|                                                         |
+---------------------------------------------------------+

部署到节点上（示例）：

  Node1:  logs_P0 (primary), logs_R1 (replica)
  Node2:  logs_P1 (primary), logs_R2 (replica)
  Node3:  logs_P2 (primary), logs_R0 (replica)
```





## 2. 单个节点内部：多个 shard + 文件

```
+=================== Node 1 (data) =======================+
|                                                         |
|  /var/lib/elasticsearch/nodes/0/indices/                |
|                                                         |
|   logs-2025.11.14/                                      |
|      shard_0/   --->  Primary P0                        |
|      shard_1/   --->  (maybe nothing on this node)      |
|      shard_2/   --->  (maybe nothing on this node)      |
|                                                         |
|   logs-2025.11.15/                                      |
|      shard_0/   --->  Replica R0                        |
|      ...                                                |
|                                                         |
|  每个 shard_* 目录里：                                    |
|   +------------------------------+                      |
|   | Lucene index segments        |  (seg_1, seg_2...)   |
|   | Translog (operation log)     |  (translog-N)        |
|   | Shard-level metadata         |                      |
|   +------------------------------+                      |
|                                                         |
+=========================================================+
```





## 3. 单个 Shard 内部：Lucene 索引结构

```
+================ Shard: logs-2025.11.14_P0 ===============+
|  (One primary shard; one Lucene index)                  |
+---------------------------------------------------------+
|                                                         |
|   +------------------+       +----------------------+   |
|   |  Segments        |       |   Translog           |   |
|   |  (immutable)     |       |  (operations log)    |   |
|   +------------------+       +----------------------+   |
|   |  seg_1           |       | translog-1           |   |
|   |  seg_2           |       | translog-2           |   |
|   |  seg_3           |       | ...                  |   |
|   |  ...             |       +----------------------+   |
|   +------------------+                                  |
|                                                         |
+=========================================================+
```

把一个 **segment** 再拆一下：

```
+-------------------- Segment: seg_3 ---------------------+
|                                                         |
|  +---------------- Inverted Index --------------------+ |
|  |  term: "hello"  -> [docId 1, docId 7, docId 42...] | |
|  |  term: "world"  -> [docId 2, docId 7, ...]         | |
|  |  term: "es"     -> [docId 3, docId 9, ...]         | |
|  +----------------------------------------------------+ |
|                                                         |
|  +----------------- Doc Values (column) --------------+ |
|  |  field "age":                                      | |
|  |    docId 1 -> 27                                   | |
|  |    docId 2 -> 35                                   | |
|  |    docId 3 -> 29                                   | |
|  |    ...                                             | |
|  +----------------------------------------------------+ |
|                                                         |
|  +---------------- Stored Fields / _source -----------+ |
|  |  原始 JSON 文档 / 部分字段存储                         | |
|  +----------------------------------------------------+ |
|                                                         |
+---------------------------------------------------------+
```

可以粗暴理解：

- 一个 **Shard** = 一个 Lucene index = 一堆 **Segment**

- 每个 **Segment**：

  - 倒排索引：做全文检索的（term → docId list）
  - doc_values：做聚合/排序的（列式结构）
  - stored fields / _source：存原始文档（用来返回给用户）

  



## 4. 写入路径（从客户端到 shard 内部）

```
Client
  |
  |  HTTP: POST /logs-2025.11.14/_doc
  v
+------------------ 任意节点 (协调节点) -------------------+
|  1) 解析请求                                            |
|  2) 根据 routing 计算 shardId = hash(_id) % 3           |
|  3) 查 cluster state 找到 shardId=0 的 primary 在哪      |
+-------------------------|------------------------------+
                          |
                          v
                 +--------+--------------------------+
                 |  Node1: Shard logs_P0 (primary)   |
                 +----------------------------------+
                 |                                   |
                 |  a) 写入 translog（append）        |
                 |  b) 写入内存 buffer                |
                 |  c) 返回成功（或等待 replica）       |
                 +----------------------------------+
                          |
                          |  复制写入
                          v
         +----------------+------------------+
         |  Node3: Shard logs_R0 (replica)   |
         +-----------------------------------+
         |  同步 translog + buffer            |
         +-----------------------------------+

后续：
- 每隔 refresh_interval (~1s)：把内存 buffer flush 成新的 segment
- 定期 flush：持久化、清理旧 translog
```





## 5. 查询路径（从 index → 多分片 → merge）

```
Client
  |
  |  HTTP: GET /logs-2025.11.14/_search
  v
+------------------- 任意节点 (协调节点) ------------------+
|  1) 根据 Index 找到所有 logical shards: 0,1,2            |
|  2) 对每个 shard 选一个实际副本 (primary 或 replica)  			|
|     Shard0 -> Node1.logs_P0 (or Node3.logs_R0)         |
|     Shard1 -> Node2.logs_P1 (or Node1.logs_R1)         |
|     Shard2 -> Node3.logs_P2 (or Node2.logs_R2)         |
+-------------------------|------------------------------+
          |               |                |
          v               v                v
   +-----------+   +-----------+    +-----------+
   |  Shard 0  |   |  Shard 1  |    |  Shard 2  |
   |  (Lucene) |   |  (Lucene) |    |  (Lucene) |
   +-----------+   +-----------+    +-----------+
   | 本地：倒排索引搜索、       ...                |
   |       计算 topN、聚合等                      |
   +-----------+   +-----------+    +-----------+
          \             |                /
           \            |               /
            \           |              /
             v          v             v
           +--------------------------------------+
           |    协调节点 merge 所有 shard 结果       |
           |    排序 / 聚合 / 分页 / 高亮            |
           +--------------------------------------+
                           |
                           v
                        返回给 Client
```



你可以先把这几张图当成 ES 存储架构的“骨架”：

- **集群层**：Cluster → Node → Index → Shard（primary/replica）
- **Shard 层**：Lucene index → Segments → 倒排索引 / doc_values / _source
- **写入/查询路径**：协调节点 fan-out / merge





- 若有新数据（新 segment、或者内存里的结构），就生成一个新的 IndexReader
- 这个 reader 就是一个新的“快照视图”

- ES 再把这个 reader 封装成一个新的 **Searcher**，替换掉旧的 searcher
- 从这一刻起，所有新的 _search 请求，用的都是这个新 searcher，所以能看到最近一次 refresh 前写入的所有文档。



# IndexReader 快照视图

**“一份被固定住的、只读的 segments 列表 + 每个 segment 的活跃文档视图”**。

## 1. 先用一句话定性：

> **快照视图 = 某一时刻，IndexWriter 里所有 segment + 删除标记 的“定格照片”。**

特点：

- **只读**：你不能通过这个 reader 改任何东西。
- **一致**：在这份快照里，你看到的是某个时刻的完整状态，不会读到“一半刷进去的数据”。
- **不随时间自动变化**：后面再写新文档、再 refresh，这个老 reader **看不到**，必须重新开一个新的 reader。

有点类似 MySQL 里的 **MVCC 快照读**：

在你执行 SELECT 的那一刻，拿了一份视图，之后别的事务 insert 的数据，你这个视图是看不到的（除非再发一次新的 SELECT）。



## 2. 从 “长什么样” 的角度：可以抽象成这样一个结构

从 Lucene 的角度看，一个 NRT IndexReader（具体是 DirectoryReader）大致是下面这样的：

```
SnapshotReader
  |
  +-- SegmentReader[0]  (对应 seg_1 + 它的 liveDocs)
  |
  +-- SegmentReader[1]  (对应 seg_2 + 它的 liveDocs)
  |
  +-- SegmentReader[2]  (对应 seg_3 + 它的 liveDocs)
  |
  +-- ...  (当前这一代所有 segment)
```

每个 SegmentReader 里，又包含几类东西（都是只读视图）：

```
SegmentReader (for seg_3)
  |
  +-- PostingLists（倒排索引：term -> docId 列表）
  |
  +-- DocValues（列式：field -> [docId -> value]）
  |
  +-- StoredFields / _source（原始字段，用于返回）
  |
  +-- liveDocs 位图（bitset: 这个 docId 是否“未删除”）
```

所以你可以直接这样想：

> **快照视图 = 一个“SegmentReader 数组”，每个 SegmentReader=“这个段里面所有的数据 + 删除标记”的只读句柄。**

它不是一块新的大内存复制，而是：

- 持有一组 **segment 文件的引用**（底层是文件句柄 + mmap + some buffers）
- 持有这一代的 **liveDocs 位图**（表示哪些 doc 是“活的”）
- 这些引用都是带**引用计数**的：只要还有 reader 用它，就不会删掉对应的 segment 文件。



## 3. 为什么叫“快照（snapshot）”？

因为它满足几个典型的“快照”特征：

### 3.1 固定的一组 segments

假设某个时刻，你的 IndexWriter 里有：

```
Segment set at time T:
  { seg_1, seg_2, seg_3 }

liveDocs：
  比如 seg_1 里 doc0/1/2 还活着，doc3 已删除...
```

这时 refresh 创建 reader：

```
SnapshotReader(T):
  segments = [ seg_1, seg_2, seg_3 ]
  liveDocs = 当时所有删除标记的那一版
```

接下来，如果有新的写入：

- 产生了新 segment seg_4
- 做了 merge，把 seg_1 + seg_2 合并成 seg_5，标记 seg_1/seg_2 为“待回收”

**这个过程中，SnapshotReader(T) 看起来完全不变**：

- 它仍然指向 [ seg_1, seg_2, seg_3 ]
- 在它的“世界里”，seg_1/seg_2 永远存在，不会突然消失或变成 seg_5

直到：

- 这个 reader 被所有正在跑的查询用完，ref-count 归零
- Lucene 才会真正删除这些旧 segment 文件（如果已经被合并掉了）

这就是典型的快照特征：**在拿快照的那一刻，“包含哪些段、每段哪些 doc 是活的”就被固定住了**。





### 3.2 固定的一组 liveDocs（删除可见性）

再看删除：

- 写线程在 T0 时刻 delete _id = 123（其实是某个 segment 上的 docId X）：
  - 在当前 writer 状态的 liveDocs 里，给它打上删除标记
- 如果这一次 delete 发生在你快照之后，那这条删除，对 **旧的 SnapshotReader(T)** 是不可见的：
  - 它的 liveDocs 仍然是 T 时刻那一版
  - 在它眼里，这个 doc 依然是“活的”

反之，如果在 T0 时刻：

- 先 delete
- 再 refresh 拿 reader

那新 fast-snapshot 就包含了这条删除标记。

**总结一下：**

> 每个 reader 都有自己的一份 *“liveDocs 快照”*。

> 在这个 reader 里，“谁是已删除的”在它的生命周期内是不会变的。

这跟你熟悉的“MVCC 下的可见性”是一个味道：

**读的是某个时间点的版本，而不是随时变化的最新版本。**





## 4. 回到 ES 的世界：一个 shard 上多个 reader 怎么共存？

再把这个概念搬回 ES 的 shard：

```
Shard (primary or replica)
  |
  +-- 当前活跃 Searcher (Reader_A，时间点 TA)
  |
  +-- 某些正在执行中的查询还在用 Reader_A
```

当下一次 refresh 到来：

1. ES 通过 NRT 接口从 IndexWriter 打开新一代 reader：Reader_B（时间点 TB）
2. 以后新的 _search 请求全部使用 Reader_B
3. 旧的 Reader_A 还会保留一段时间，直到所有用它的查询都结束：

```
时间线：

   查询1开始 (用 Reader_A)
   查询2开始 (用 Reader_A)
   refresh() 产生 Reader_B
   查询3开始 (用 Reader_B)
   查询1结束 （Reader_A 引用计数 -1）
   查询2结束 （Reader_A 引用计数 -1 → 0）
   -> Reader_A 关闭，底层 segments 的 ref-count 也减一，可能触发删除
```

**关键点：**

- **查询永远在“自己的那一代快照”上跑完**，不会跑到一半突然看到新段或不同的删除标记；
- 更新/写入线程可以一直往前推进，生成新 segments、做 merge，不会影响正在跑的 query 的一致性；
- 这就是 Lucene/ES 的“读写分离 + 近实时”的核心机制。





## 5. 你可以怎么把它讲给别人听（面试/交流用）

如果面试官问到：

> “你说 ES 有 NRT（近实时搜索），那个 reader 的快照视图具体是什么含义？”

你可以这样说（用简化版语言）：

> 在 Lucene 里，底层是很多 segment 文件，每次写入/删除都会改变当前这一代的 segment 集合和删除标记。

> 每次 ES 做 refresh 时，会从当前 IndexWriter 的状态里拿一份 **只读快照**：

- > 这份快照包含当时所有的 segment，以及当时的删除位图（liveDocs）。

- > 这份快照在整个查询过程中保持不变，后续的新写入和删除不会影响它。

> 我们可以把这个快照视图理解成“**某一时刻索引的完整、只读版本**”，

> 查询就跑在这个版本上，所以能做到：

- > 写线程不断前进

- > 已经开始的 query 在固定视图上稳定执行

- > 每次 refresh 再打开一个新视图，让新数据对后续 query 可见。



如果对方再追问：

> “那旧的 segment 是什么时候被删除的？”

你可以接着说：

> 旧快照上引用着旧 segment 文件，它们有引用计数：

- > 只要还有 reader 在用，segment 文件不会真的删

- > 当所有 reader 都不用某个 segment 了（ref-count 变 0），Lucene 才会物理删除它或者在 merge 之后清理掉

> 所以 NRT reader 其实是一个 **“由一组 segment + liveDocs 组成的只读快照”**，是非常典型的 copy-on-write / MVCC 思路。



# DSL 例子

1. 货物信息分词查询，参与相关性评分。
2. 出发地 / 目的地（省市区）**必须命中，但不参与评分**。
3. 有长宽高信息的货源，在原有 _score 基础上**额外加分**。

假设字段大概是这样（你可以按自己实际字段名替换）：

- 货物信息全文：goods_text（text）
- 出发地：from_province / from_city / from_district（keyword）
- 目的地：to_province / to_city / to_district（keyword）
- 长宽高：length / width / height（double 或 integer）

```json
GET goods_index/_search
{
  "from": 0,
  "size": 20,
  "query": {
    "function_score": {
      // 1. 基础查询：全文 + 精确过滤（过滤不参与评分）
      "query": {
        "bool": {
          "must": [
            {
              "multi_match": {
                "query": "冷链 钢卷 运输",        // 用户输入的搜索词
                "fields": [
                  "goods_text^3",             // 货物信息权重更高
                  "remark"                    // 备注等其它说明字段
                ]
              }
            }
          ],
          "filter": [
            // 出发地
            { "term": { "from_province": "北京市" } },
            { "term": { "from_city": "北京市" } },
            { "term": { "from_district": "朝阳区" } },

            // 目的地
            { "term": { "to_province": "河北省" } },
            { "term": { "to_city": "廊坊市" } },
            { "term": { "to_district": "广阳区" } }
          ]
        }
      },

      // 2. 自定义加分逻辑：有长宽高就加分
      "functions": [
        {
          // 有 length 字段的货源，加 1 分
          "filter": {
            "exists": { "field": "length" }
          },
          "weight": 1.0
        },
        {
          // 有 width 字段的货源，再加 0.5 分
          "filter": {
            "exists": { "field": "width" }
          },
          "weight": 0.5
        },
        {
          // 有 height 字段的货源，再加 0.5 分
          "filter": {
            "exists": { "field": "height" }
          },
          "weight": 0.5
        }
      ],

      // 3. 组合方式：原始相关度 + 额外加分
      "score_mode": "sum",   // 多个 function 的分数相加：1 + 0.5 + 0.5 = 2
      "boost_mode": "sum"    // 新分数 = 原始 _score + 上面算出来的加分
    }
  },
  "sort": [
    // 先按 score 排，如果你希望再结合时间，也可以追加一个字段排序
    { "_score": "desc" },
    { "publish_time": "desc" }
  ]
}
```

这条 DSL 的语义拆解

- bool.must 中的 multi_match

  → 对 goods_text（+ 其它字段）做分词搜索，算 BM25 相关度，**是基础** **_score** **的来源**。

- bool.filter 里的出发地 / 目的地条件

  → 全部是**硬过滤**，只决定“是否保留这条货源”，不影响 _score，而且会被 ES 缓存，加速。

- function_score.functions

  - 每个函数都有自己的 filter，满足条件的文档额外得到一个 weight 分数。
  - 比如一条货源三个字段都有：length+width+height → 额外加分 1 + 0.5 + 0.5 = 2。

- score_mode = "sum"：

  → 把所有 function 的分数加起来，得到一个“额外加分值”。

- boost_mode = "sum"：

  → 新的 _score = 原始相关度 _score + 额外加分值。

  所以**本身相关度高 + 维度信息完整**的货源会更靠前。



如果你想再细一点，比如：

- 三个字段全都有 → 加 2 分
- 有任意两个 → 加 1 分
- 只有一个 → 加 0.5 分

```json
GET goods_index/_search
{
  "from": 0,
  "size": 20,
  "query": {
    "script_score": {
      "query": {
        "bool": {
          "must": [
            {
              "multi_match": {
                "query": "冷链 钢卷 运输",
                "fields": ["goods_text^3", "remark"]
              }
            }
          ],
          "filter": [
            { "term": { "from_province": "北京市" } },
            { "term": { "from_city": "北京市" } },
            { "term": { "from_district": "朝阳区" } },
            { "term": { "to_province": "河北省" } },
            { "term": { "to_city": "廊坊市" } },
            { "term": { "to_district": "广阳区" } }
          ]
        }
      },
      "script": {
        "source": """
          double base = _score;     // 原始 BM25 相关度
          int cnt = 0;
          if (!doc['length'].empty) cnt++;
          if (!doc['width'].empty) cnt++;
          if (!doc['height'].empty) cnt++;

          double extra = 0.0;
          if (cnt == 3) {
            extra = 2.0;
          } else if (cnt == 2) {
            extra = 1.0;
          } else if (cnt == 1) {
            extra = 0.5;
          }
          return base + extra;
        """
      }
    }
  }
}
```

# BM25

> **BM25 = 稀有词更值钱 + 词多有用但收益递减 + 长文要打折。**

稍微展开就是：

- 每个 term 先算一个 **idf（逆文档频率）**，语料里越少见，idf 越大，说明这个词越“有区分度”；
- 对每条文档，在指定字段里算这个 term 的 **tf（出现次数）**，tf 越大得分越高，但用一个含 k1、b 的分式让它 **递减增长**，防止单词刷满屏直接爆表；
- 同时用文档长度和平均长度做一个 **长度归一化**：同样的 tf，在特别长的文档里得分要被压一压，避免“废话多”的长文因为词多而天然占优势；
- 对一个查询里的多个 term，就是把这些 term 在这个文档上的 BM25 分数 **加和**，再结合字段 / 查询的 boost，得到最终 _score。





# nested 总结

## 1.1 nested 是什么？

- nested 是 ES 里**特殊的 object 数组类型**：

  > “一个文档里嵌了一组**子文档**，子文档之间互相独立，但还是跟着父文档一起存、一起删。”

- 和普通 object 不同：

  - 普通 object 数组会被**扁平化**到同一层字段上；
  - nested 数组里的每个元素会被当成一个**独立的隐藏子文档**建索引。

  



## 1.2 解决什么问题？

典型问题：**多行条件要落在“同一条记录”上。**

例子：订单 + 多条明细 items：

```
"items": [
  { "product_id": "P1", "qty": 5 },
  { "product_id": "P2", "qty": 10 }
]
```

需求：查“至少有一条明细满足：product_id = P1 AND qty >= 10”。

- 如果 items 是普通 object：

  - ES 只看字段级别：

    - items.product_id 里有 P1
    - items.qty 里有 10

    

  - 它会误以为命中 ✅，但其实：

    - P1 的 qty = 5

    - P2 的 qty = 10

      → **典型 false positive。**

- 如果 items 是 nested：

  - 每个 item 是独立子文档；
  - 查询时可以要求“这些条件必须同时满足在同一个 nested 子文档上”，不会乱拼。

  



## 1.3 mapping 写法

```
"mappings": {
  "properties": {
    "items": {
      "type": "nested",
      "properties": {
        "product_id": { "type": "keyword" },
        "qty":        { "type": "integer" },
        "price":      { "type": "double" }
      }
    }
  }
}
```





## 1.4 查询写法（关键点：nested查询包一层 path）

```
{
  "query": {
    "nested": {
      "path": "items",
      "query": {
        "bool": {
          "must": [
            { "term":  { "items.product_id": "P1" } },
            { "range": { "items.qty": { "gte": 10 } } }
          ]
        }
      }
    }
  }
}
```

含义：

> “在 items 这个 nested 数组里，至少有一个元素同时满足 product_id = P1 且 qty ≥ 10。”





## 1.5 使用场景 vs 不适用场景

**适合用 nested 的场景：**

- 一条主记录下面有**多条“行内明细”**，查询时条件要落在同一行上：

  - 订单 + 明细
  - 商品 + 多段规格/价格
  - 货源 + 多段线路（segments）

  

**不适合 nested 的场景：**

- 子记录非常多、更新非常频繁：
  - nested 在底层依然是“跟着父文档一起存”的：
    - 更新一条 nested 子记录，本质是重写整个文档（所有子记录一起重建索引）；
    - 子记录量大 + 更新频繁，会有明显写放大。
  - 这种场景更适合考虑 parent-child 或者拆成两个索引 + 应用层 join。





## 1.6 优缺点总结

**优点：**

- 完整保留“数组里每一行”的语义；
- 查询时天然支持“多条件落在同一行”的约束；
- 性能比 parent-child 好（在同一 doc 内 join）。



**缺点：**

- 写入成本高：更新一个 nested 元素，需要重写整个父文档；
- 文档过大、nested 数组过长时，单个 doc 变得很重，影响索引和查询性能；
- 仅适合嵌套层级比较浅、子元素数量有限的场景。





# 第二天 0 点下架改为 24 小时下架

## **背景前提（两种方案共同点）**

- 索引按天建：goods_YYYY_MM_DD

- 货源有字段：

  - id：全局递增（用于增量分页：id > lastId）
  - create_time：创建时间

- 新需求：**货源有效期 = 创建后 24 小时**

- 查列表（找货大厅）：

  - 条件：出发地/目的地 精确、货物信息分词
  - 分页：每次查 id > lastId 的 20 条

  



## 方案一：单写（推荐）

> 每条货**只写当天索引**，查询时查“最近两天”+用时间过滤控制 24 小时。

### **写入**

- 11 月 14 日 10:00 发布的货 A：
  - **只写入**索引：goods_2025_11_14



### **查询（找货大厅）**

- 当前日期：11 月 14 日

  → 查询索引集合：goods_2025_11_13 + goods_2025_11_14

  （可以直接写两个 index，也可以用 alias 封装成 goods_current_2d）

- 查询 DSL 核心条件（伪例）：

```json
GET goods_2025_11_13,goods_2025_11_14/_search
{
  "size": 20,
  "sort": [{ "id": "asc" }],
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "冷链 钢卷",
            "fields": ["goods_text^3", "remark"]
          }
        }
      ],
      "filter": [
        { "term": { "from_city": "北京" } },
        { "term": { "to_city": "廊坊" } },
        {
          "range": {
            "create_time": { "gte": "now-24h" }  // 严格 24 小时 TTL
          }
        },
        {
          "range": {
            "id": { "gt": 1234567890 }           // 上一批 maxId
          }
        }
      ]
    }
  }
}
```

- 第二天（11 月 15 日）：
  - 查询索引集合变成：goods_2025_11_14 + goods_2025_11_15
  - 仍然使用 create_time >= now-24h + id > lastId 实现 TTL + 增量分页。

> 可以通过 **alias** 固定一个名字，比如 goods_current_2d，每天 0 点把“最老的一天”踢出 alias、“新的一天”加进 alias，代码里永远查同一个 alias。

### **特点**

- ✅ **TTL 逻辑清晰**：

  完全由 create_time >= now-24h 控制，和“写在哪些 index”解耦。

- ✅ **存储成本最低**：

  每条货源只存一份。

- ✅ **状态更新简单**：

  更新/下架只改一个 index。

- ✅ **增量分页兼容**：

  跨索引也可以统一 sort id asc + range id > lastId，ES 会全局归并排序。

- ✅ **配合 alias 更灵活**：

  想从“2 天窗口”改成“3 天窗口”，只改 alias 配置，不动代码。

- ⚠️ 需要在查询时**永远带上时间过滤**，不能只靠“索引日期”来做 TTL。





## 方案二：双写（当天 + 次日）

> 每条货**写两份：当天索引 + 次日索引**，查询时只查“当天索引”。

### **写入**

- 11 月 14 日 10:00 发布的货 A：

  - 写入索引：goods_2025_11_14
  - 同时写入索引：goods_2025_11_15

  

### **查询（找货大厅）**

- 11 月 14 日：只查 goods_2025_11_14
- 11 月 15 日：只查 goods_2025_11_15
- DSL 类似：

```json
GET goods_2025_11_14/_search
{
  "size": 20,
  "sort": [{ "id": "asc" }],
  "query": {
    "bool": {
      "must": [ ... 分词条件 ... ],
      "filter": [
        ... 出发地目的地 ...,
        {
          "range": {
            "id": { "gt": 1234567890 }          // 上一批 maxId
          }
        }
        // 如果要严格 24 小时 TTL，这里仍然要：
        // { "range": { "create_time": { "gte": "now-24h" } } }
      ]
    }
  }
}
```

> **关键点：**

> 如果不加 create_time >= now-24h 的条件，A 会从 11 月 14 日 10:00 一直可见到 11 月 15 日 23:59，**不是 24 小时，而是接近 38 小时** → 真正 TTL 还是得靠时间过滤。



### **特点**

- ✅ 查询索引名简单：

  每天只查“当天 index”，代码只按“今天日期”拼一个索引名。

- ✅ 你原本的“单 index + id>lastId 分页”逻辑可以不改。

- ❌ **TTL 不再只靠索引粒度**：

  要严格 24 小时，还得配合 create_time >= now-24h。

- ❌ **存储翻倍**：

  每条货源保存在两个 index 里。

- ❌ **状态更新复杂**：

  下架 / 修改时，要保证两个 index 都更新成功（否则出现今天列表已下架、明天列表还在的脏数据）。

- ❌ **清理逻辑更绕**：

  

  - 今天的 index 到期删除时，要保证里面的货在“次日 index”里也已经过了 TTL；
  - 否则可能删错 / 漏删。



## 两种方案对比小结

| **维度**          | **单写（只写当天 index，查两天 + 时间过滤）**     | **双写（写当天+次日，只查当天 index）**                      |
| ----------------- | ------------------------------------------------- | ------------------------------------------------------------ |
| 写入              | 每条货只写 1 个 index                             | 每条货写 2 个 index（今天 + 明天）                           |
| 查询索引集合      | 昨天 + 今天（可通过 alias 封装）                  | 只查当天索引                                                 |
| TTL 实现方式      | 依赖 create_time >= now-24h，逻辑清晰             | 如果不加时间过滤，会变成 ~38h；严格 TTL 仍需时间过滤         |
| 存储空间          | 较省，只存一份                                    | 翻倍                                                         |
| 状态更新          | 只更新一个 index，简单                            | 必须保证两索引状态一致，需要幂等 + 重试 + 校验               |
| 分页（id>lastId） | 直接支持：跨索引统一 sort + range                 | 直接在当天索引上支持                                         |
| 运维复杂度        | 中等（需要维护 alias 或“昨天+今天”索引集合）      | 中等偏高（要管理双写、一致性、索引删除时机）                 |
| 适用倾向          | 数据量大、TTL 要求严、喜欢逻辑干净的场景（⭐推荐） | 更在乎“查询只查当天索引”这件事，能接受双写与复杂一致性管理的场景 |





我的建议（给你一个一句话结论）

- 如果你**能接受查询时带一个固定的时间 filter（****create_time >= now-24h****）**，那：

  - **单写 + 查询昨天+今天（或 alias 管理最近两天）是更干净、长期可维护的方案。**

- 如果你们内部**非常执着于“所有查询只查当天 index，不想处理多索引场景”**，而且能接受：

  - 存储翻倍、

  - 状态双写 + 一致性校验 & 补偿，

    那双写方案也不是不行，但一定要在文档里标清楚这些 tradeoff。



## 多种 mapping 兼容问题

因为我们是后端先上，app 或者融合将在后端上线完之后的几天内择机上线与灰度，而多 mapping 兼容问题最长只会持续 24 小时，所以我们只需要在代码层面做兼容，比如如果增加了字段，但是从昨天的 index 里没有取到的话，就使用默认值等。