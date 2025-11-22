# 消息积压解决方案

## 一、先定位：搞清楚“谁在积压、从什么时候开始”

最基础的排查要搞清这几件事：

1. **哪个 Topic / 哪些分区 / 哪个消费组在积压**

   - Kafka 看：consumer lag（每个 group、每个 partition 的 lag）。
   - RocketMQ 看：消费进度 vs 队列最大 offset。

   

2. **积压从什么时候开始、积了多少**

   - 是最近瞬间飙升（突发流量），还是长期微积（消费能力整体不足）。
   - 算一下 **生产 TPS** 和 **消费 TPS** 大致差多少倍。

   

3. **消费端是不是出了“异常情况”**

   - 消费者实例挂了、线程池打满、频繁 full GC。
   - 消费逻辑里有：
     - 大量 IO（慢 SQL、调用第三方 HTTP、写文件）。
     - 锁、事务冲突、重试风暴（一直消费失败又重试）。

**很多时候“积压”根本不是吞吐不够**，而是消费逻辑卡死 / DB 被打挂 / 某个下游 500 导致疯狂重试。





## 二、先止血：不先控制入口，永远追不上

### 1. 对“生产端”做限流 / 降级

- 非核心消息可以：

  - 降采样（只发重要数据）。
  - 合并上报（批量聚合再发）。

- 针对造成积压的那几个业务接口：

  - 临时做**流量控制 / 熔断 / 降级**，挡在前面避免继续洪峰灌入 MQ。

- 这一步目的很简单：

  > **让“进入速度”不要继续远大于“消费速度”**。

  > 不然你加多少消费者都只是延缓死亡时间。



### **2. 分级处理：**不重要的消息直接丢弃**/超时失效**

- 有些场景本身就是“过时就没意义”的（比如页面埋点、监控数据）：
  - 对已经超过某个时间窗的消息，可以考虑标记为过期直接丢弃，记录日志即可。
- 避免为“已经没有价值的数据”浪费大量消费资源。

> 面试里可以说一句金句：

> 「对于时效性很强的 Topic，可以在积压时按时间阈值做**过期丢弃**，避免为无价值数据买单。」



## 三、加速消化：提高消费能力（短期 + 中期手段）

### 1. 横向扩容消费者实例

**Kafka：**

- 同一 Consumer Group 下，**消费者实例数 ≤ 分区数** 才有用：
  - 如果现在有 8 个 partition，只跑了 2 个实例 → 可以加到 8 个实例，让每个实例独占 1 个 partition。
- 如果实例已经接近 partition 数，再加实例没用，就要考虑**增加分区**（注意：增加分区只影响新数据的分配，历史数据仍在旧分区）。



**RocketMQ：**

- 集群消费（Clustering）模式下：

  - 同一个 group 加更多实例，SDK 内部会 Rebalance，把 Topic-Queue 平分给新实例，整体消费并发提升。

  

- 同时可以调大消费线程数：

  - 修改 consumeThreadMin / consumeThreadMax，提高每个实例里的消费并发。

> 要注意：盲目加实例和线程，如果后端 DB/缓存/下游服务扛不住，会把压力整体迁移到别的地方。



### 2. 提升单实例吞吐：批量消费 + 批处理下游

很多积压场景都是因为**消费逻辑里对 DB/下游的操作是逐条处理**，可以做几件事：

1. **MQ 批量拉取 / 批量回调**

   - Kafka：

     - 调大 max.poll.records（每次 poll 拉更多消息）。
     - 调整 fetch.min.bytes / fetch.max.wait.ms，更倾向批量传输。

     

   - RocketMQ：

     - 调大 pullBatchSize（一次拉多少条）。
     - 调大 consumeMessageBatchMaxSize（一次交给 listener 多少条）。

     

2. **消费逻辑改成批处理**

   - 原来是：来一条消息 → insert 一次 DB。
   - 改为：一批（如 100 条）消息 → 拼成一批 insert/upsert。
   - 或者缓存到内存队列里做“批量 flush”。

> 面试可以说：

> 「**一个大的批量写 DB >> N 次单条写 DB**，可以用批量插入/批量更新和中间缓冲队列来极大提高消费端吞吐。」



### **3. 临时开“**专用消费集群**”做快速消费 / 异步落地**

针对已经积压的那一大坨老消息，可以采用：

1. **新建一个临时 Consumer Group**

   - 只负责快速把积压消息拉出来，写到：
     - 另一条备份 Topic；
     - 或一个中间表 / 数据湖（HDFS、对象存储）。
   - 消费逻辑尽量简单：**不做复杂业务校验，只做“转存/归档”**。

   

2. 后面再用离线任务慢慢跑业务逻辑：

   - 比如用 Spark / Flink / 定时 Job 从中间表里扫数据做补偿。



这样做的本质是：

> **把“在线消费链路”从大量历史积压中解放出来，先保证新消息能及时处理**，

> 积压那一坨通过“旁路”慢慢消化。





### 4. 增加分区 / 队列数（结构性扩容）

如果确认是**长期业务量上涨**导致消费能力不足，不只是短期峰值，那么：

- Kafka：

  - 增加 Topic 的 partition 数量；
  - 对应增加 Consumer Group 实例；
  - 必要时做 partition reassign，把分区均匀分布到更多 broker 上。

  

- RocketMQ：

  - 为 Topic 分配更多 queue（每个 broker 上 queue 数）。
  - 扩 broker 实例，配合新的 queue 做路由。

注意：**增加分区/队列是结构性变更**，要评估：

- 消费顺序语义是否受影响（按 key 保持局部顺序）。
- 下游是否能接受更高并发（DB、缓存的吞吐）。





## **四、修业务：**从根上减少积压概率**（长期治理）**

很多公司线上大积压，最后定位都是消费逻辑写得有坑：

### 1. 优化消费逻辑

- 拆分耗时操作：不要在消费线程里做：
  - 非必要的远程调用；
  - 超重的计算；
  - 大量日志。
- IO 操作前加缓存 / 降级：
  - 比如 DB 查不到就落缓存、打标重试，而不是疯狂 reject 导致消息重试风暴。
- 避免**事务过大 / 锁粒度过粗 / 死锁**，导致消费线程长期阻塞。



### 2. 消息分层：核心 vs 非核心

- 核心链路（订单、支付）：保证及时消费，Topic 独立、资源优先分配。
- 非核心链路（埋点、报表）：可以有更长的延迟要求，积压时可降级甚至丢弃。



### 3. 监控 & 预警体系

- 必配监控指标：

  - 各 Topic 各消费组的 lag。
  - 消费端 TPS、失败率、重试次数。
  - 消费端线程池使用率、耗时分布。

  

- 在“积压到一定数量/时间”之前就报警：

  - 例如：某个 group lag > 10w 且持续 5 分钟。

  



## **五、**你可以对面试官这样总结**（背诵版）**

你要简短说，可以这样说一段（你可以按自己习惯改下）：

> 生产环境出现消息积压，我一般分四步处理：

> **第一步是定位**：看是哪个 Topic/消费组积压、从什么时候开始，是消费实例挂了、下游变慢，还是最近流量突增。

> **第二步是先止血**：对上游做限流/降级，对时效性较强、已经过期的消息可以直接丢弃，避免继续积压。

> **第三步是加速消化**：

- > 横向扩容消费实例、增加消费线程，必要时增加分区/队列；

- > 调整批量参数（Kafka 的 max.poll.records，RocketMQ 的 pullBatchSize 等），把消费逻辑改成批量写 DB；

- > 对于已经积累的大量历史消息，可以开一个临时消费集群快速转存到中间存储，主消费链路只处理新消息。

  > **第四步是长期治理**：优化消费逻辑，避免重试风暴和大事务，按照业务重要性拆 Topic 并配置不同的资源/策略，同时把 lag、消费耗时、失败率纳入监控和告警，避免下次再“突然爆雷”。





## 一、消息发送：同步 vs 异步、单条 vs 批量

先统一几个概念：

- **同步发送**：你的业务线程调用 send() 后，会**一直等结果**（成功 or 异常）再往下跑。
- **异步发送**：你调用 send() 后**马上返回**，结果通过回调、Future 异步拿；业务线程不会卡在这里。
- **单条 vs 批量**：
  - “单条”：你的代码一条条调用 send(msg)。
  - “批量”：你的代码一次传一批消息给 SDK；或者 SDK 在内部自动把多条消息攒成批量一起发给 Broker（这是常见优化点）。



### 1. Kafka 生产端

#### 1）Kafka：API 角度的同步 / 异步

Kafka 的 KafkaProducer 核心方法是：

```
Future<RecordMetadata> future = producer.send(record);
```

这个 send() 做的事本质是：

- **立即把 record 放进本地缓冲队列，返回一个 Future**；
- 真正的网络发送由 IO 线程在后台批量完成。

所以：

- **同步发送**（你自己让它同步）：

```
producer.send(record).get(); // 阻塞等待 ack，成功 / 抛异常
```

- **异步发送**（更常见）：

```
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        // 异步失败处理
    } else {
        // 发送成功
    }
});
```

- **fire-and-forget**（完全不管结果）：

```
producer.send(record); // 不 get，不 callback，只管丢出去
```

> 结论：

> Kafka 的发送永远是**内部异步**的，

> “同步 / 异步”是**你自己的线程要不要等 Future** 的区别。



#### 2）Kafka：单条 vs 批量

**对你写代码的人来说：**

- API 只有按条 send(record)，没“官方 send(List)”。
- 批量是你自己写循环 OR 在业务层先聚合。



**但 Producer 内部一定是在做批量**：

- 对每个 topic-partition 有一个**缓冲区**，攒一批 Record 成为一个 RecordBatch。
- 批量大小由配置控制：

```
batch.size=16384      # 每个 partition 的 batch 缓冲区大小（字节）
linger.ms=5           # 最多等 5ms，攒不到 batch.size 也会发出去
compression.type=snappy/gzip/lz4 # 批量一起压缩
```



> 直观理解：

> 你**一条条**调用 send()，

> 内部会按 partition 自动**攒批**，

> 最终是**一批批**发给 Broker，Broker 也一批批落 log。



### 2. RocketMQ 生产端

RocketMQ 的发送模式说白了就三种：

1. **同步 send**
2. **异步 send**
3. **oneway（单向，不等结果）**



#### **1）同步发送（**最常用**）**

```
SendResult result = producer.send(msg); // 直到 Broker 返回才继续
```

这是**默认**最常用的方式，尤其是订单、支付、库存这类对可靠性比较敏感的业务。



#### 2）异步发送

```
producer.send(msg, new SendCallback() {
    @Override
    public void onSuccess(SendResult sendResult) {
        // 成功回调
    }

    @Override
    public void onException(Throwable e) {
        // 失败回调，做补偿
    }
});
```

适合高吞吐、对单条时延要求比较高的链路，比如：大批量埋点、日志、非关键通知等。



#### 3）单向发送（oneway）

```
producer.sendOneway(msg); // 不等结果，不保证到没到
```

- 极端场景用，比如：日志打点、监控上报，对丢一部分消息能接受。



#### 4）RocketMQ 的批量发送

RocketMQ 生产者实际上有**真正的批量 API**：

```
List<Message> msgs = ...; // 同一 topic，不能是延迟消息/事务消息
SendResult result = producer.send(msgs);
```

限制：

- 这批消息必须是**同一个 topic**；
- **不支持事务消息 / 定时消息等特殊类型**；
- 总大小有限制（默认 4MB 左右，需要拆分）。

内部发送时还是会打成一批一起写 CommitLog。





## 二、消息消费：批量 vs 单条、拉模式 vs 推模式

这里一定要分清几个层面：

- **MQ 客户端 SDK 与 Broker 的协议**：几乎都是“批量拉取”。
- **交到你业务代码的接口**：可能是 “一条一条” 给你，或者一批 List 给你。



### 1. Kafka 消费端

Kafka 的消费是**明确的拉模式**（poll）：

```
ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
for (ConsumerRecord<String, String> record : records) {
    // 按条处理
}
```



#### 1）MQ 层面：一定是批量拉取

- 每次 poll()，底层发 FetchRequest，Broker 返回一个批次；

- 返回的批次里可以有：

  - 多个 partition；
  - 每个 partition 多条消息。

  

#### 2）你代码层面：可单条，也可批量处理

- 通常写法是：

```
for (ConsumerRecord<?, ?> record : records) {
    // 一条一条处理
}
```

- 也可以自己按批处理，比如一次 poll 回来 500 条，组装成一批 SQL 做批量 insert：

```
List<DomainEvent> batch = new ArrayList<>();
for (ConsumerRecord<?, ?> record : records) {
    batch.add(convert(record));
}
saveBatch(batch); // 批量落库
```



#### 3）批量相关配置（常用这几个）

```
max.poll.records=500        # 每次 poll 最多给你多少条
fetch.min.bytes=1           # 单次 fetch 最小字节数（攒不够就再等一会）
fetch.max.wait.ms=500       # 等攒批的最长时间
```

> 小结：

> Kafka 消费协议是“**批量拉取**”，

> 你代码里**怎么用这批**（逐条 or 批量）完全由你自己决定。





### 2. RocketMQ 消费端

以最常用的 “**PushConsumer（其实内部还是 long-poll 拉取）**” 为例。

#### 1）Push 模式：回调里拿到 List<MessageExt>

并发消费监听器：

```
consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                                                    ConsumeConcurrentlyContext context) {
        // msgs 是一批消息
        for (MessageExt msg : msgs) {
            // 一条条处理也行
        }
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
```

顺序消费监听器类似，只是接口不同：

```
MessageListenerOrderly {
    ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs,
                                        ConsumeOrderlyContext context)
}
```

> 注意：**回调参数就是一批 List<MessageExt>**，

> 你可以选择：

- > 在循环里单条处理；

- > 或者合并成批，一次性写 DB。



#### 2）批量消费相关参数

RocketMQ 有两个重要参数：

```
consumeMessageBatchMaxSize=1   # 一次回调最多给多少条，默认一般是 1
pullBatchSize=32               # 内部一次从 Broker 拉多少条
```

- pullBatchSize：**SDK 从 Broker 拉的批量大小**；
- consumeMessageBatchMaxSize：**一次回调给你多少条**。

你要真正批量用，一般会把 consumeMessageBatchMaxSize 调大一些，比如 10、20、32，然后在回调里做批处理。



#### 3）PullConsumer（了解一下）

PullConsumer（手动拉）也是一次返回一批 List<MessageExt>，你自己决定怎么处理。





## 三、两者对比 + 实战建议（你可以直接记住这一段）

### Kafka

- **生产发送**

  - API 上是**单条 send**，内部自动按 partition **攒批**；
  - 你的线程要不要等结果 = 同步/异步的区别：
    - 等 future.get() → 同步；
    - 用回调 / 不管 future → 异步。

  

- **消费**

  - poll() 一次拿一批 ConsumerRecords，其实就是一堆消息；

  - 典型用法：

    - 对**关键链路**：一条条处理 + 手动 commit；
    - 对**批处理场景**：一批组装好，用批量 SQL / 批量写 ES 等。

    

  

### RocketMQ

- **生产发送**

  - **三种模式**：同步、异步、单向（oneway）；
  - 有真正的“**批量发送 API**”：producer.send(List<Message>)（要求同 topic、限制 4MB）。

  

- **消费**

  - Push 模式的回调就是 List<MessageExt>，天然支持**批量消费**；
  - 配合 consumeMessageBatchMaxSize、pullBatchSize 调整批量大小。

  



## 怎么回答面试官类似问题（给你一段可以背的）

> Kafka 这边，Producer API 形式上是按条 send，内部会按照 batch.size 和 linger.ms 自动在每个 partition 上攒批，最终批量发送给 Broker。我们在业务侧可以选择 send().get() 做同步发送，或者通过 callback 做异步发送。

> Consumer 端通过 poll() 一次拿一批 ConsumerRecord，可以逐条处理，也可以把一批数据拼成批量 SQL 或批量写 ES 来提升吞吐。

> 

> RocketMQ 这边，Producer 原生支持同步、异步和 oneway 三种发送模式，同时还提供了批量发送 API，可以一次发一批消息到同一个 Topic。

> Consumer 端的 Push 模式实际上是长轮询 Pull，一次回调就会给一个 List<MessageExt>，配合 consumeMessageBatchMaxSize、pullBatchSize 等参数可以控制批量消费大小，业务可以在回调里按条处理或者按批处理。