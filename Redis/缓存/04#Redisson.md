# **1) 它是什么（一句话）**

**Redisson** 是基于 Netty 的 Redis 客户端，内置了大量“分布式并发原语”（锁、读写锁、信号量、闭锁、限流器等）。

最常用的是它的 **RLock 分布式可重入锁**，帮你把：SET NX PX、唯一 token、Lua 原子解锁、**看门狗自动续期** 这整套细节都封装好了。





# **2) 用法速览（你日常最需要的那三招）**

## **2.1 基本可重入锁（看门狗自动续期）**

```
Config config = new Config();
config.useSingleServer().setAddress("redis://127.0.0.1:6379");
// 可选：设置看门狗默认续期窗口（默认 30s）
config.setLockWatchdogTimeout(30_000);

RedissonClient redisson = Redisson.create(config);

RLock lock = redisson.getLock("order:lock:123");
lock.lock();               // 不指定 leaseTime => 启用“看门狗”自动续期
try {
    // 临界区
} finally {
    lock.unlock();         // 只有持有者线程能解锁（Lua校验）
}
```

- **看门狗**：当你 lock() **不传 leaseTime** 时生效。默认把锁的过期时间设为 **30s**，并且每隔约 **10s** 自动续期回到 30s，直到你 unlock()。适合**你不确定业务耗时**的场景。



## **2.2 指定租期（禁用看门狗）**

```
if (lock.tryLock(200, 10, TimeUnit.SECONDS)) { // 最多等 200ms，拿到锁后10s自动到期
    try { ... } finally { lock.unlock(); }
}
```

- 一旦**显式指定了 leaseTime**（如 10s），**就不会自动续期**。适合你**能明确上限**的短临界区。



## **2.3 公平锁 / 读写锁**

```
RLock fair = redisson.getFairLock("inventory:lock");
// 公平队列，避免饥饿（基于队列+PubSub）
fair.lock();

RReadWriteLock rw = redisson.getReadWriteLock("sku:rw");
rw.readLock().lock();  // 多读并发，写互斥
rw.writeLock().lock(); // 独占写
```





# **3) 它底层到底做了什么（核心点别背错）**

1. **存储结构**：RLock 不是简单的 SET key token，而是用 **Hash 结构**记录：

   - **字段**：线程标识（clientId + threadId）
   - **值**：重入计数（reentrant count）
   - 锁 key 有 TTL（过期时间）
   - 解锁/续期/加锁的判断都在 **Lua 脚本里原子执行**。

   

2. **可重入**：同一个 JVM、同一线程再次 lock() 会把 Hash 的计数 +1；unlock() 计数减到 0 才真的 DEL。

3. **唯一标识**：线程维度的 id（Redisson 自动带上），Lua 解锁时会验证“是不是我加的锁”，避免误删他人锁。

4. **看门狗**：

   - 只有 **没传 leaseTime** 才启用；
   - 默认 lockWatchdogTimeout=30s，**每 ~10s** 给锁续期回到 30s；
   - 线程/进程挂了，续期停止，TTL 到期后锁自动释放。

   

5. **公平锁**：用一个等待队列（Redis 结构 + Pub/Sub 通知）控制获取顺序，避免“后来者插队”。代价是**吞吐更低**，一般热点场景不建议用。

6. **tryLock(waitTime, leaseTime)**：内部会阻塞订阅通知，直到超时或轮到你，拿到后设置一次租期（**不续期**）。





# **4) 配置建议（常用几条就够了）**

```
config.setLockWatchdogTimeout(30_000); // 看门狗窗口：默认 30s
config.setThreads(0);                  // 业务线程数（0=cpu*2），一般默认即可
config.setNettyThreads(0);             // Netty 线程数（默认=cpu*2）

// 单机 / 哨兵 / 集群择一
config.useSingleServer()
      .setAddress("redis://xxx:6379")
      .setDatabase(0)
      .setConnectionPoolSize(64)
      .setConnectionMinimumIdleSize(16);
// config.useSentinelServers()...
// config.useClusterServers()...
```

- **看门狗窗口如何选**：如果你常规临界区 P99 在 200ms~3s，30s 很稳。若临界区可能跑十几秒，增大到 60s/120s。
- **已知短任务**：优先用 tryLock(wait, lease) 指定租期，禁用续期，避免“忘记解锁导致长期占用”。





# **5) 常见坑 & 最佳实践**

**（1）一定 finally{ unlock(); }**

- unlock() 只会在“**是我加的锁**”时生效，否则抛异常（或返回 false）；别吞异常。



**（2）别把超长逻辑塞在锁里**

- 锁住“必要的临界区”就放；I/O、RPC、慢 SQL 能外移就外移。



**（3）watchdog 不是“容错神器”**

- **网络长时间抖动/GC STW** 可能让续期失败 → 锁过期被别人拿走。对下游有副作用的写，建议引入 **“栅栏号（fencing token）”**：

  - 获取锁时 INCR 一个版本号，调用下游写接口时带上版本；下游只接受“更大版本”的写，旧持有者就算继续跑也写不进去（需要下游配合校验版本）。

  

**（4）主从切换的语义**

- 这是分布式锁的老问题：如果用了主从/哨兵，**主节点未把写同步到从节点就挂了**，从节点晋升后可能**“丢锁”**。
- 解决路径：
  - 使用 **可靠存储**（例如多个主节点 + **RedLock** 组合），或
  - 对关键操作使用 **栅栏号** 做幂等校验，或
  - 业务上允许“极低概率的并发”，用补偿兜底。
- Redisson 支持 RedissonRedLock（多主投票），但其可靠性在学界有争议；要用需评估网络模型与延迟。



**（5）避免“锁键名”冲突**

- 用业务命名空间和资源 ID：order:lock:{orderId}。
- 一类资源一个 key，别把不相关的互斥放到同一把锁。



**（6）选择合适的原语**

- “读多写少” → RReadWriteLock（多读共享）。
- “一次只能 N 个进入” → RSemaphore；需要过期许可可用 RPermitExpirableSemaphore。
- “等待所有前置任务完成” → RCountDownLatch。
- “多个锁要么都拿到要么都失败” → RMultiLock（组合）。







# **6) 监控与排障**

- 打点 **加锁等待时间**、**持有时长**、**失败/超时次数**；
- Redis 侧看 **键空间**、**慢日志**、**CPU/QPS**；
- 关注 **看门狗续期失败**（可在应用侧打 warn）。







# **7) 何时用 Redisson、何时自己手写**

- 99% 情况直接用 Redisson：**可靠、可重入、续期、丰富原语**都给你包了。
- 自己手写 SET NX PX + Lua 适合**极简依赖**和**专项学习**，但要自己扛续期、重入、公平、容灾等细节。







## **结尾小抄（背诵版）**

- **lock() 不传租期 ⇒ 启用“看门狗”：默认 30s，约每 10s 续期。
- **tryLock(wait, lease) 指定租期 ⇒ 不续期。
- **Lua 原子释放** + **线程ID校验** 确保“只解自己的锁”。
- 主从切换可能丢锁：关键路径引入 **栅栏号**，或考虑多主 **RedLock**（谨慎评估）。
- **原则**：锁短、解快、少持有；监控等待/失败；键名分域，避免误伤。





# 可重入、写独占、公平锁实现原理

**数据形态**

- **Redis 键**：lock:{yourLockName}（类型：**Hash**，整把锁就是这个键）。
- **Hash 字段**：
  - r:{clientId}:{threadId} → 该线程持有**读锁**的重入计数。
  - w:{clientId}:{threadId} → 该线程持有**写锁**的重入计数。
  - 同时可能存在多个 r:*（多线程并发读）；w:*通常至多一个（同线程可重入）。
- **TTL**：挂在**整个 Hash 键**上；看门狗定期 PEXPIRE 把 TTL 重置到窗口（默认 30s）。





## **核心数据结构（Redis 里怎么存）**

- **一个 Hash 键**（名字就是你的锁名，例如 doc:123），里面用**字段 = 持有者标识、值 = 计数**来做重入：
  - 读计数：r:{clientId}:{threadId} -> count
  - 写计数：w:{clientId}:{threadId} -> count
- **键 TTL**：给整个 Hash 设过期时间；看门狗会定期 PEXPIRE 续期（你用 lock() 不传租期时）。
- **Pub/Sub 通知通道**：当锁完全释放（读=0、写=0）或状态改变时，发布一条消息，**唤醒等待方**（Redisson 所有锁都会这么做）。 

> 说明：字段名的前缀是抽象表示；真实实现同样是“**按线程维度统计计数**”，并用 Lua 保证原子检查 + 修改。华为云的兼容文档也明确了 Redisson 锁的 Lua 会用到 HEXISTS/HINCRBY/PEXPIRE/... 这一套哈希与 TTL 指令。 





## **“读可重入”怎么做（拿/放读锁的原子流程）**

**获取读锁（Lua 里的原子逻辑）**（伪代码）：

```
-- 前置：keys[1] = 锁hash，argv[1] = r:client:thread，argv[2] = w:client:thread，argv[3] = ttl
-- 1) 若存在“写持有者”且不是我自己 → 失败
if hasAnyWriteOwnerOtherThanMe() then return 0 end
-- 2) 给我自己的读计数 +1；首次进入则创建该field
hincrby(keys[1], argv[1], 1)
pexpire(keys[1], argv[3])  -- 续期
return 1
```

**释放读锁：**

```
-- 读计数 -1；若为0删除该field
local c = hincrby(keys[1], argv[1], -1)
if c == 0 then hdel(keys[1], argv[1]) end
-- 若此时既无任何读者也无写者 → publish 解阻
if noReadersAndWriters(keys[1]) then publish(unlockChannel) end
return 1
```

**为什么可重入？**

同一线程再次 readLock() 只是把自己的 r:{client}:{thread} 计数 **+1**；unlock() 再 **-1**，到 0 才真正“放掉”。





## **“写独占 + 可重入”怎么做**

**获取写锁：**

```
-- 1) 若存在“写持有者且不是我” → 失败
if hasWriteOwnerOtherThanMe() then return 0 end
-- 2) 若存在任何“读持有者且不是我” → 失败
if hasAnyReaderOtherThanMe() then return 0 end
-- 3) 到这里：要么没人持有，要么我自己在重入
hincrby(keys[1], argv[2], 1)  -- w:client:thread 计数 +1（可重入）
pexpire(keys[1], argv[3])
return 1
```

**释放写锁：**

```
local c = hincrby(keys[1], argv[2], -1)
if c == 0 then hdel(keys[1], argv[2]) end
-- 写释放后如果没有读写持有者 → publish 唤醒
if noReadersAndWriters(keys[1]) then publish(unlockChannel) end
return 1
```

> 这就实现了：**写是独占的**（有任何“别人”的读/写都不让过），**写也可重入**（同线程 HINCRBY 叠加计数）。





## **公平性：有没有“公平读写锁”？**

- **Redisson 的 RReadWriteLock 明确是**“**非公平**”**模式**（文档与 javadoc 都写了 *Works in non-fair mode. Therefore order of read and write locking is unspecified.*）。即：**不保证先来先得**，新来的读在很多情况下可以“插队”。 

- **为什么不做公平？** 做严格公平需要**排队结构**（入队顺序、公平唤醒、阻止新读突入），吞吐会显著下降，还会放大 Redis 往返与队列维护成本。Redisson只在**互斥锁**上提供了**公平版本** RFairLock；**读写锁没有公平版**。 

- **要“写不饥饿”怎么权衡？** 实战里常用两种办法：

  1. **写入口套一把 RFairLock**（把“写请求”先公平排队），进入后再用 writeLock() 独占修改；
  2. **应用层削峰/合并**热点读，或对读路径做 singleflight，降低“读洪峰永远压着写”的概率。

  





## **等待/唤醒怎么做（避免自旋）**

当获取失败时，客户端会**订阅该锁的通道**并阻塞等待；对端释放时 PUBLISH 一条通知，等待方被唤醒后再执行一次“Lua 检查 + 获取”。这避免了热自旋，同时减少无效请求；这是 Redisson 锁体系的通用机制。 





## **小结（对你要点逐条对应）**

- **读可重入**：Hash 里给**本线程**的读字段计数 +1/-1，到 0 才真正释放。
- **写独占 + 可重入**：有“别人”的读/写就拒绝；同线程写 +1/-1 做重入。
- **公平性**：RReadWriteLock **非公平**；若需要“写不饥饿”，在写入口**叠 RFairLock** 或做应用层排队。
- **原子性与看门狗**：所有“检查+修改+续期/发布”都在 **Lua** 内原子完成；不传租期时由看门狗定期 PEXPIRE 保持 TTL。 