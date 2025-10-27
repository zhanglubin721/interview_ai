# **1) 目标与基本约束**

- **目标**：跨进程/跨主机互斥执行一段临界区代码，进程崩了也能自动释放。
- **场景**：**单实例 Redis**（无主从切换）。若有主从/哨兵切换，请看文末“注意事项”。





# **2) 获取锁（原子 + 带过期）**

**命令**（服务端原子执行）：

```
SET lock:user:123 <token> NX PX <ttl>
```

- NX：仅当 key 不存在时写入（互斥）
- PX ttl：一次性设置**过期时间**（毫秒）
- token：调用方生成的**随机唯一标识**（如 UUIDv4），代表“谁拿到锁”

**Java（Jedis/Lettuce 思路相同）**

```
String key = "lock:user:123";
String token = UUID.randomUUID().toString();
long ttlMs = 15000; // 15s，按业务P99设置

boolean acquired = "OK".equals(
    jedis.set(key, token, SetParams.setParams().nx().px(ttlMs))
);
```

> 选择 ttlMs：要**大于临界区 P99 耗时**（再留一点裕量），过短会“锁过期后原持有者还在执行”。





# **3) 释放锁（Lua 原子校验 + 删除）**

避免“把别人加的锁删掉”，释放时必须**检查 value（token）是否还是我**，**与删除同一条命令内原子完成**。

**Lua 脚本**

```
-- KEYS[1]=key, ARGV[1]=token
if redis.call("get", KEYS[1]) == ARGV[1] then
  return redis.call("del", KEYS[1])
else
  return 0
end
```

**Java 调用**

```
String lua =
  "if redis.call('get', KEYS[1]) == ARGV[1] then " +
  "  return redis.call('del', KEYS[1]) " +
  "else return 0 end";
Object ok = jedis.eval(lua, Collections.singletonList(key),
                       Collections.singletonList(token));
```

> 好处：**幂等**（多次释放也安全），并且规避“先 GET 再 DEL”之间的竞态。





# **4) 续期（看门狗 / watchdog）**

如果临界区可能超过初始 TTL，需要**后台定时续期**，否则锁会过期被别人抢走。

**策略**

- 单独线程定时（建议**每 ttl/3** 续期一次，留足抖动和 GC 暂停余量）。
- 续期也必须**先校验 token**再续期，原子执行。
- 如果续期失败（key 不存在或 token 不匹配），**立刻停止续期并中断临界区后续动作**（你已经丢锁）。

**Lua 续期脚本**

```
-- KEYS[1]=key, ARGV[1]=token, ARGV[2]=ttl_ms
if redis.call("get", KEYS[1]) == ARGV[1] then
  return redis.call("pexpire", KEYS[1], ARGV[2])
else
  return 0
end
```

**Java 简版**

```
ScheduledExecutorService ses = Executors.newSingleThreadScheduledExecutor();
AtomicBoolean renewing = new AtomicBoolean(true);
long ttlMs = 15000;
long period = ttlMs / 3;

ScheduledFuture<?> task = ses.scheduleAtFixedRate(() -> {
    if (!renewing.get()) return;
    Object r = jedis.eval(renewLua,
        Collections.singletonList(key),
        Arrays.asList(token, String.valueOf(ttlMs)));
    if (!(r instanceof Long) || ((Long) r) == 0L) {
        // 续期失败：锁已失效或被他人占用
        renewing.set(false);
        // 这里应尽快让业务停止/回滚（打断后续外部副作用）
    }
}, period, period, TimeUnit.MILLISECONDS);

// ...临界区逻辑...
// 结束后释放并停表
renewing.set(false);
jedis.eval(unlockLua, Collections.singletonList(key),
           Collections.singletonList(token));
task.cancel(true);
```

> 实战建议：把续期线程做成**守护线程池**，续期周期加入**5–10% 随机抖动**，降低齐步压力。





# **5) 获取失败的退避与超时**

- 自旋重试 + **指数退避 + 随机抖动**（如 50–200ms 之间随机），设置**总体等待上限**，超时直接失败返回，避免把线程都堵死。
- 需要**公平性**时可做排队（一般不强求，复杂度变高）。







# **6) 可选：可重入（单 JVM 线程内）**

Redis 的值可存成 "token:count" 或用 Hash 结构记录 (token, count)，先判断 token 一致再 INCR/DECR count。多 JVM/多线程时实现“跨进程可重入”较复杂，实际**一般不建议**（使用 Redisson 的可重入锁更容易）。





# **7) 常见风险与防护**

1. **锁过期后“原持有者”还在干活**

   - 续期失败后一定要**停止**对下游的写入；
   - 若下游资源支持，加入**“封签/栅栏（fencing token）”**：每次获取锁分配单调递增版本，下游只接受**更大版本**的写入，彻底规避“过期后双写”风险（需要资源方配合版本校验）。

   

2. **TTL 设得太短或 GC 长暂停**

   - 把 ttl 设成**业务 P99 + 裕量**，续期 ttl/3；
   - 监控续期失败次数。

   

3. **主从/哨兵切换场景不安全**

   - 单实例没问题；**有主从时**可能出现“主上写入未同步、主挂、从提升”导致锁丢（非线性化）。
   - 解决：使用 **WAIT**/强一致集群、Redisson 的实现、或更复杂的 RedLock/Quorum（各有取舍）。

   

4. **误删他人锁**

   - 坚持**Lua 检查 token 再删**；切勿 DEL 直接删。

   

5. **消息/线程异常**

   - 续期和释放要做**幂等**、**失败重试**（指数退避）；
   - 应用崩溃：锁会因 TTL 自动释放。

   





# **8) 何时用现成组件**

- **Redisson** 已内置：可重入锁、公平锁、看门狗（默认 30s，自动续期）、信号量、读写锁等；稳定省心。除非是非常轻量的场景或学习目的，不必重复造轮子。







## **速记（面试版）**

- **拿锁**：SET key token NX PX ttl（原子 + 过期 + 唯一 token）。
- **解锁**：Lua if get==token then del（幂等 + 原子）。
- **续期**：看门狗定时 PEXPIRE（Lua 校验 token），周期取 ttl/3。
- **失败处理**：续期失败立刻停手；必要时用**栅栏 token**保护下游。
- **主从切换风险**：单机 OK，主从要额外手段/用现成实现。

这套就是“单机 Redis 分布式锁”的标准做法；按上面的代码块，你可以很快在项目里拉起一个可靠的实现。