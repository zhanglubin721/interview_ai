### #9.1.1

#### 持久化策略

配置项：

```
aof-use-rdb-preamble yes
```

这是 Redis 4.0 引入、在 Redis 7.0+ 里**默认开启**的一种“**混合持久化格式**”： 

> **AOF 文件的开头部分是一个 RDB 快照（preamble），后面再接普通 AOF 日志。**

整体工作方式可以这么理解：



1. 日常写入：

   - 仍然是**AOF 方式**：每条写命令按你配置的 fsync 策略追加到 AOF 缓冲 / 文件里。 

   

2. 需要 AOF 重写（BGREWRITEAOF）时：

   - 以前：重写出的新 AOF 全是“命令序列”的纯 AOF。
   - 现在：重写时先直接写一个 **RDB 快照** 到新文件头部（preamble），然后再写后续增量命令作为 AOF tail。 

   

3. 重启加载时：

   - Redis 把整个 appendonly.aof 先当成 RDB 读入前半段 snapshot，
   - 再按 AOF 方式回放后半段的增量命令。 

   

这样你同时拿到了：

- **RDB 的优点**：

  - 文件前半段是紧凑的 RDB 格式，**加载速度快、体积小**；

  

- **AOF 的优点**：

  - 快照之后的增量命令照样可以“秒级追加”，**丢数据窗口可以做到 1s 以内甚至更小**（看 fsync 策略）；

  

- 对你来说：

  - 运维只看一个持久化文件 appendonly.aof，但里面既有 snapshot 又有增量 log，
  - **不用再手动去管 “先从 dump.rdb 再从 appendonly.aof 回放” 这两套文件。**

  

Redis 官方和一些文章就把这叫做 **“RDB-AOF hybrid persistence / RDB preamble AOF”**，本质就是你说的“结合了 RDB 和 AOF 的优点的策略”。 



小结一句话（你可以直接贴文档）：

> 现在的 Redis 可以同时开启 RDB 和 AOF，两种各干各的；而从 4.0 起又提供了 aof-use-rdb-preamble yes 的混合持久化：AOF 文件以 RDB 快照作为前缀，后面跟 AOF 增量日志。这样既保留了 AOF 的高可靠性和小丢失窗口，又利用 RDB 的紧凑格式加快重启数据加载速度，是“RDB + AOF 优点结合”的推荐持久化策略（Redis 7 起默认开启）。



### #9.7.1

#### Sentinel 高可用

##### 1. 整体结构：一主多从 + 多个 Sentinel 进程

典型架构长这样：

```
        +----------------------+
        |    Sentinel A       |
        +----------------------+
        +----------------------+
        |    Sentinel B       |
        +----------------------+
        +----------------------+
        |    Sentinel C       |
        +----------------------+

              |           ^
     (监控 + 选主 + 通知)   |
              v           |

          [ master ]
         10.0.0.1:6379
             /    |   \
            /     |    \
     [ slave1 ] [ slave2 ] [ slave3 ]
    10.0.0.2   10.0.0.3   10.0.0.4
```

- **一主多从：**

  - Master 负责写，Slave 从主库复制数据，平时只读或读写分离。

  

- **多个 Sentinel：**

  - 是**独立的进程**（redis-sentinel），不存业务数据，只做：

    监控 + 故障判断 + 选主 + 通知客户端“新主是谁”。

> 核心点：**高可用的是“主从 + Sentinel 组合”，不是单靠主从。**

> **主从只是一份数据多份复制；Sentinel 才负责“主挂了之后自动切换”。**





##### 2. **Sentinel** 具体在监控什么？

每个 Sentinel 会周期性地：

1. 定期 PING master + 所有 slave + 其他 Sentinel：

   - 一段时间（down-after-milliseconds）收不到合法回复，就把这个实例标记为 **SDOWN（主观下线）**。

   

2. 跟 master、slave 上订阅特殊频道：__sentinel__:hello

   - 用来发现集群里还有哪些 Sentinel，交换一些元数据。

   

所以每个 Sentinel 自己有一个“视角”：

- “我觉得这个 master 死了吗？”
- “有哪些 slave 在正常复制？”
- “有几个兄弟 Sentinel 和我意见一致？”





##### 3. 从 SDOWN 到 ODOWN：谁说了算？

Sentinel 的故障判断分两层：

1. **主观下线（SDOWN）**：

   - 单个 Sentinel 自己认为“这个 master 掉线了”（PING 超时等）。

   

2. **客观下线（ODOWN）**：

   - Sentinel 会问其他 Sentinel：“你们是不是也觉得这货挂了？”
   - 当**超过配置的 quorum（法定票数）** Sentinel 都认为它挂了 → 进入 ODOWN。

   

例如配置：

```
sentinel monitor mymaster 10.0.0.1 6379 2
```

- 表示：名字叫 mymaster 的 master，在 10.0.0.1:6379；
- 2 是 **quorum**：至少 2 个 Sentinel 同意 master 挂了，才视为 ODOWN。

> **只有 ODOWN 以后，Sentinel 集群才会启动真正的“故障转移（failover）”流程。**





##### 4. 选主：从多个 slave 里选一个新的 master

进入 ODOWN 后，Sentinel 会在它们之中选出一个“**Leader Sentinel**”，由它来主导这次 failover（通过一个简单的 Raft 风格投票/epoch 机制选出）。Leader 负责：

1. **在所有 slave 中挑一个最合适的**：

   - 优先级（slave-priority）最高的；
   - 复制偏移量最新（和旧 master 最接近）的；
   - 网络状态良好、延迟小的；

   

2. 对选中的 slave：

   - 发命令：SLAVEOF NO ONE → 把它升为新 master；

   

3. 对其它 slave：

   - 让它们 SLAVEOF new-master → 改成从新 master 复制。

整个过程就是：

> 主挂了 → Sentinel（Leader）从一堆 slave 里择优 → 把它扶正为 master → 其余全部挪过去当从库。





##### 5. 客户端怎么知道新 master 是谁？

这是 Sentinel 体系里**经常被误解**的点：

> **Sentinel 不会自动帮你改客户端的连接地址。**

> **你必须让客户端“通过 Sentinel 去查当前的 master 地址”。**

典型做法是：

- Java / Spring 里，用 RedisSentinelConfiguration / Jedis/Lettuce 的 Sentinel 模式：

  - 配置：**Sentinel 地址列表 + master 名称**（mymaster）；

  - 客户端启动时，会去询问任一 Sentinel：

    SENTINEL get-master-addr-by-name mymaster

    Sentinel 返回当前 master 的 IP:Port；

  - 如果 master 发生变更，客户端会收到 Sentinel 发的通知（Pub/Sub 或定期轮询），再**自动重连到新 master**。

也可以手写脚本调用 Sentinel：

```
SENTINEL get-master-addr-by-name mymaster
# -> 10.0.0.2 6379    （原 master 10.0.0.1 挂后，slave1 被扶正）
```

> 所以在“基于 Sentinel 的高可用”里，**客户端必须支持 Sentinel 协议**，要么用中间代理（如 Twemproxy/Proxy），否则你自己代码里得处理“主从切换”的逻辑。





##### 6. 故障恢复完整时序（简版）

你可以脑补一下“主机炸了”的完整链路：

1. 主机（10.0.0.1:6379）机器宕机：

   - 所有 Sentinel 对这个实例 PING 超时 → 各自标记 SDOWN。

   

2. 一段时间后：

   - 有足够多 Sentinel 互相交流发现“大家都觉得它挂了” → 标记 ODOWN；

   

3. 选出一个 Leader Sentinel：

   - 通过 epoch/vote 简单选举一次，让某个 Sentinel 负责这轮 Failover。

   

4. Leader Sentinel 执行故障转移：

   - 选出最适合的 slave（比如 10.0.0.2）→ SLAVEOF NO ONE；
   - 通知其他 slave 改为 SLAVEOF 10.0.0.2 6379；

   

5. Sentinel 更新元数据：

   - 内部认为 mymaster 对应的新地址是 10.0.0.2:6379；

   

6. 客户端侧：

   - 使用 Sentinel 模式的客户端会收到变更通知，或在下一次请求之前重新向 Sentinel 拉 master 地址；
   - 后续所有写请求就打到 10.0.0.2 这台新 master。

   

原来挂掉的那台机器若之后重启：

- Sentinel 会发现它又活了，但它的角色不能再是 master 了；
- 会把它配置成新 master 的从库：SLAVEOF 10.0.0.2 6379，重新同步数据。





总结

> **Redis Sentinel 提供的是“主从架构上的自动故障转移与发现服务”。**

> 在一主多从架构下，多个 Sentinel 进程持续通过 PING 监控 master 和各个 slave，当某个 Sentinel 认为 master 超时未响应时会先标记为“主观下线（SDOWN）”，再通过与其他 Sentinel 交互，如果有足够多 Sentinel（达到 quorum）都认为其不可用，则将 master 标记为“客观下线（ODOWN）”，并发起一次故障转移（failover）。

> 故障转移过程中，由选举出的 Leader Sentinel 在所有从库中选出一个最合适的 slave（根据优先级、复制偏移量、延迟等）升级为新的 master，并让其余 slave 重新复制新 master；随后 Sentinel 更新该 master 名称对应的实际地址。客户端若以 Sentinel 模式接入（配置 Sentinel 地址 + master 名称），便可通过 SENTINEL get-master-addr-by-name 等命令获知当前主节点，在主从切换时自动重连新 master，从而实现整个 Redis 集群的高可用。



### 主从同步（PSYNC）

#### 老版本 SYNC的问题

Redis 2.8 之前，主从复制主要靠一个命令：SYNC：

- 从库连到主库之后，发送 SYNC
- 主库：
  1. 直接做一次 BGSAVE 生成 RDB
  2. 把 RDB 整个发给从库
  3. RDB 传输期间的新写命令，一边执行一边缓冲起来，等 RDB 传完再把这段缓冲命令一起发给从库

**问题在于**：

- 只要从库断线重连（哪怕只断了几秒），**就必须重新跑一遍全量复制**（重新发 RDB）
- 对大实例来说，BGSAVE + 传输 RDB 非常重：CPU/磁盘/网络都吃紧

→ 所以老版本在“网络不太稳定、有抖动”的环境里，**经常在全量复制，系统抖得厉害**。



#### 2. PSYNC 的核心目标：尽量“部分重同步”

PSYNC 全称“Partial Resynchronization”（部分重同步），从 Redis 2.8 开始引入，后面逐步成为标准复制协议的基础。

它想解决的问题就是：

> **从库断开再连上来时，如果只落后了一点点数据，能不能只补这点“尾巴”，不要每次都全量？**

这就引出两个概念：

- **master runid**：主库的唯一身份标识（重启会变）
- **复制偏移量 offset**：
  - 主从之间的“字节序列的进度条”
  - 主库每发给从库 N 字节复制数据，就把自己 offset +N
  - 从库也维护自己的 offset（自己已应用到哪里了）



以及一个关键结构：

- **replication backlog（复制 backlog 环形缓冲区）**：
  - 主库保留最近一段复制数据的字节流（按字节存命令内容）
  - 容量通过配置 repl-backlog-size 控制（比如 64MB、128MB）



有了 runid + offset + backlog，就可以做到：

- 如果从库重连时说：

  “我是之前那个主的从库，runid = X，offset = 123456”

- 主库看：

  - runid 一样 → 同一主库
  - offset 落在 backlog 范围内 → 说明“从上次断开后缺的那点数据，我这里还能找到”

→ 那只需要把“offset 之后的那部分 backlog 数据”发给它即可，叫 **部分重同步**（非常便宜）。



#### 3. PSYNC 协议：两种模式

##### 3.1 第一次挂从库：PSYNC ? -1

- 从库之前没有复制历史，不知道主的 runid 和 offset

- 会发送：PSYNC ? -1

- 主库收到之后会回复：+FULLRESYNC <runid> <offset>，然后：

  - 跑一次 BGSAVE
  - 发整份 RDB 给从库
  - RDB 之后再发增量命令

  

**这一步其实就是“新版协议下的全量复制”，只是 handshake 用 PSYNC 开始。**



##### 3.2 已经复制过、断线重连：正常的部分重同步流程

从库保存了之前保存的主库 runid 和自己的 offset，例如：

主库收到之后判断：

- 如果 runid 不匹配（主重启过或者切主了）

  → 无法部分同步，回复 +FULLRESYNC，走全量

- runid 一致，则检查 offset：

  - 如果 offset 还在 backlog 区间内：

    → 回复 +CONTINUE，进入 **部分重同步**

    → 直接从 backlog 中把 offset 之后的数据发给从库

  - 如果 offset 太早，已经被 backlog 覆盖掉了：

    → 退化为 +FULLRESYNC，重新发 RDB

  



#### 4. backlog 环形缓冲区是怎么用的？

你可以把 backlog 想象成主库上的“最近一段时间的复制命令滚动日志”：

- 类型是一个固定大小的环形缓冲：

  - repl-backlog-size 配多少，就有多少空间
  - 写满了就从头覆盖（丢最早的部分）

  

- 主库对于每一条写命令：

  1. 自己执行
  2. 把命令的序列化字节写入 backlog 环形缓冲
  3. 把同样的字节流通过网络发给所有从库

  

所以 backlog 有两个用途：

1. 新的从库全量同步时，BGSAVE/RDB 期间的增量命令都缓在 backlog 里；
2. 已有从库断线重连时，用它来补差（只要 offset 还在范围内）。



**存活时间**：

- backlog 实际上保证的是一个“可回放窗口”：

  - 如果你的 repl-backlog-size = 256MB，
  - 在你当前写流量下，可能能“覆盖过去 30 秒/1 分钟/5 分钟的写命令”

- 只要从库在这个窗口内完成重连，就可以用 PSYNC 做部分同步；

  超出就只能全量。



#### 5. 在哪些情况下会退回全量同步？

即使有 PSYNC，也不可能保证永远不全量：

1. **从库第一次接入**：

   - 不知道主的 runid/offset，只能全量（PSYNC ? -1）

   

2. **主库重启（runid 变了）**：

   - 从库手上保留的是旧 runid；
   - reconnect 之后发的 runid 和当前主不一致 → FULLRESYNC

   

3. **从库落后太多，offset 被 backlog 覆盖**：

   - 主库 backlog 中最旧的 offset > 从库上报的 offset

     → backlog 已经 “抹掉” 从库需要的那部分历史 → FULLRESYNC

   

4. **参数/配置导致不能部分同步**：

   - 比如某些极端场景里 repl-backlog-size 太小，或者被关闭。

> 所以 PSYNC 不是“消灭全量”，而是 **“尽量减少全量，优先尝试部分重同步”**。

总结

> **PSYNC 是 Redis 自 2.8 引入的“部分重同步复制协议”，用来避免从库断线后每次都做全量复制。**

> 每个主从连接都维护一个主库 runid 和复制偏移量 offset，主库同时维护一个固定大小的复制 backlog 环形缓冲区，用于保存最近一段时间的写命令字节流。当从库重连时，它会发送 PSYNC <runid> <offset> 给主库；若 runid 一致且 offset 仍在 backlog 范围内，主库回复 +CONTINUE，仅从 backlog 中补发缺失部分（部分重同步）；否则回复 +FULLRESYNC，触发一次 RDB 全量复制。

> backlog 的大小由 repl-backlog-size 控制，会形成一个“可重放窗口”：从库只要在这个时间窗口内完成重连，就可以利用 PSYNC 避免全量。复制延迟可以通过 INFO REPLICATION 中主从 *_repl_offset 的差值来监控。整体来看，PSYNC 将 Redis 复制从“常常全量”优化为“优先增量，必要时再全量”，显著降低了网络抖动环境下的主从同步成本。



### 内存碎片

#### jemalloc 内存管理器预防内存碎片的措施

多级size class：没有了大小不同导致的间隙问题，不易产生外部碎片

线程级别arena：多arena 相互干扰小，碎片集中在局部，不会全局失控

延迟归还、延迟合并：方便再次利用

大对象独立空间：用完就还



内存结构

- **size class** 就是“户型（两居、三居、四居）”
- **bin** 就是“这个户型的房源池”

不过有个细节：

> 不是“全城所有两居都在一个小区”

> 而是“**每个城区（arena）里**都有一批两居小区（三居小区、四居小区）”。

> **jemalloc 会为进程建很多个 arena，每个线程主要用自己的那个 arena**，这样就不和其他线程抢同一套 bin/run 了，锁竞争就小很多。

```
arena
  ├─ bin(8B)
  │    ├─ run #1 (全是 8B slot)
  │    └─ run #2 (全是 8B slot)
  ├─ bin(16B)
  │    ├─ run #3 (全是 16B slot)
  │    └─ ...
  └─ bin(32B)
       ├─ ...
```

- **run（有时也叫 slab）**：

  - 是一大块**连续的内存区域**，一般是好多页（4KB × N）
  - **一个 run 只服务一个 bin（即一个 size class）**
  - 比如“16 字节 bin”的一个 run 可能是 4KB 或 8KB 大小

  

- **slot / region**：

  - 是 run 里切出来的**一个个固定大小的小格子**
  - 对 16 字节的 bin 来说，每个 slot 就是 16 字节
  - 对 32 字节的 bin 来说，每个 slot 就是 32 字节



#### 为什么要搞这么复杂的一套？

几个关键原因

你可以反过来想：

**如果不用这么复杂，只用一个最朴素的 malloc/free，会发生什么？**



##### Redis 这种「长期运行 + 高 QPS + 大量小对象」的服务

Redis 的特点：

- 一跑就是几个月不重启；
- 每秒成千上万次 malloc/free；
- **绝大部分是几十字节、几百字节的小对象**（SDS、dict 节点、listpack buffer 等）；
- 多线程：主线程 + I/O 线程 + 后台线程等。

对于这样的服务，一个分配器必须满足：

1. **分配/释放非常快**（否则 CPU 大量浪费在内存管理上）
2. **多线程下锁竞争要小**（否则线程都卡在 allocator 的全局锁上）
3. **内存碎片可控**（否则跑一阵子 RSS 飙上去，物理内存被榨干）
4. **行为稳定、可预期**（延迟抖动不能太大）
5. **不需要应用配合移动对象**（不像 JVM 那种可以做压缩 GC，C 的指针不能随便搬）

> 朴素的 allocator（比如“维护一个大链表，来回找最合适的空洞”那种）在这种场景，**会死得非常惨**：

> 慢、锁多、碎片爆炸、延迟抖动大。

所以 jemalloc 要解决的本质就是：

> 在**不能移动对象**的前提下，尽可能在「速度」和「碎片」之间取得一个很好的平衡，而且还能在多线程下扩展。

这就逼出来了这些结构：arena / bin / run / slot。