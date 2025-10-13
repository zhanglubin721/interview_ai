# **0) Condition 是什么？**

- Condition 绑定在某把锁（通常是 ReentrantLock）上，是**在持有锁的前提下**做“等待/通知”的机制，类似 Object.wait/notify 的“可分组、可多路”版本（一把锁可有多条 Condition）。

- **两条队列**同时存在：

  1. **同步队列**（Sync Q）：AQS 的 CLH 变体队列，等锁的线程都在这儿。
  2. **条件队列**（Cond Q）：仅用于 await 等待某条件的线程；是单向链 firstWaiter → ... → lastWaiter。

  

**节点状态**

- 在 Cond Q 里的节点：waitStatus = CONDITION (-2)
- 在 Sync Q 里的节点：waitStatus ∈ {0, SIGNAL(-1), CANCELLED(1)…}

> 重要：**signal 不会直接“把锁交给等待者”**，它只把节点**从 Cond Q 搬运到 Sync Q**，让等待者去**重新竞争锁**。真正的“顺序”由锁的获取规则决定（公平/非公平）。



# **1)** **await()**的完整路径（可中断版本）

### **（A）调用方前置条件**

- 必须**已持有**这把锁；否则抛 IllegalMonitorStateException。
- 典型用法：**while**(条件不满足) **await()**；用 while 是为了应对**伪唤醒/错误通知**。



### **（B）关键步骤（按真实 AQS 顺序）**

```
await() {
  // 1) 把自己加入“条件队列”（节点状态 = CONDITION）
  Node node = addConditionWaiter();     // 串到 firstWaiter/lastWaiter

  // 2) 彻底释放当前持有的锁（含重入计数），记录释放的“层数”
  int saved = fullyRelease(node);       // 对应 unlock() 多次直到 state=0

  int interruptMode = 0;
  // 3) 在被 signal 前，我“不在同步队列上”，一直阻塞
  while (!isOnSyncQueue(node)) {        // 还没被搬到 Sync Q
    LockSupport.park(this);             // 进入等待（可被中断/超时唤醒）
    if (Thread.interrupted()) {         // 被中断了
      interruptMode = checkInterruptWhileWaiting(node); 
      // 判定中断发生在 signal 之前还是之后（见下）
      break;
    }
  }

  // 4) 一旦被搬进 Sync Q（或因为中断/超时），开始“重新获取锁”
  if (acquireQueued(node, saved) && interruptMode != THROW_IE)
      interruptMode = REINTERRUPT;      // 自旋获取锁，返回时已重新持有 saved 层次

  // 5) 清理 Cond Q 中的垃圾节点（已取消/已转移者）
  if (node.nextWaiter != null) 
      unlinkCancelledWaiters();

  // 6) 中断处理：抛出或“补中断位”
  if (interruptMode == THROW_IE) throw new InterruptedException();
  if (interruptMode == REINTERRUPT) selfInterrupt();
}
```



### **（C）ASCII：**await()**时的队列迁移**

```
(调用 await 前)
[Lock held by T]  SyncQ: head ...             CondQ: (empty)

1) addConditionWaiter(T 节点: ws=CONDITION)
[Lock held by T]  SyncQ: head ...             CondQ:  T(CONDITION)

2) fullyRelease → T 释放锁
[Lock free]       SyncQ: head ...             CondQ:  T(CONDITION)

3) park 等待 signal/中断/超时
```

**要点**

- await() 会**完全释放锁**（含重入层数），回头**重新获取**到同样层数。
- 在被 signal 前，节点只在 **Cond Q**，**不在 Sync Q**，因此**不会参与锁竞争**。



# **2)** **signal()**/ **signalAll()**的完整路径

### **（A）调用方前置条件**

- 必须**已持有**这把锁；否则抛 IllegalMonitorStateException。
- 通常在**修改了共享条件**（状态）之后调用 signal，并且在同一把锁的保护下（防丢通知的套路）。





### **（B）关键步骤**

```
signal() {
  Node first = firstWaiter;
  if (first != null) doSignal(first);
}

doSignal(Node first) {
  // 1) 从 Cond Q 的头取出一个节点 n
  do {
    if ((firstWaiter = first.nextWaiter) == null) lastWaiter = null;
    first.nextWaiter = null;  // 断开条件队列中的链接
  } while (!transferForSignal(first) && (first = firstWaiter) != null);
}

boolean transferForSignal(Node node) {
  // 2) 把节点从 CONDITION → 0，并“入队”到 Sync Q 尾部
  if (!CAS(node.waitStatus, CONDITION, 0))
      return false;                        // 节点已被取消，换下一个
  Node p = enq(node);                      // 入 Sync Q 尾
  // 3) 设前驱为 SIGNAL，保证释放时能唤醒它
  int ws = p.waitStatus;
  if (ws > 0 || !CAS(p.waitStatus, ws, SIGNAL))
      LockSupport.unpark(node.thread);     // 兜底：直接唤醒它
  return true;
}
```

**signalAll()**：反复把 Cond Q 的所有节点做上述搬运。





### **（C）ASCII：**signal()把节点搬到同步队列

```
(调用 signal 时)
SyncQ: head ... tail                   CondQ: T1(CONDITION) -> T2(CONDITION) ...

搬运 T1：
  1) T1.waitStatus: CONDITION → 0
  2) enq(T1)  →  SyncQ: head ... tail -> T1
  3) 设 T1.prev.waitStatus = SIGNAL（或兜底 unpark）

此时：
SyncQ: head ... T1(waiting to reacquire)   CondQ: T2(CONDITION) ...
```

**要点**

- signal() **不释放锁**、也不直接让被唤醒者拿到锁；它只把等待者**转运**到 **Sync Q**，等待者要**重新竞争锁**。
- 如果调用者随后 unlock()，释放路径会照常从 head 开始唤醒**最近的有效后继**（即刚运来的等待者有机会被唤醒并最终拿到锁）。



# **3)** **await()**的中断/超时语义（与“何时检查中断”）

- **可中断的 await()**：

  - 若**在被移动到 Sync Q 之前**发生中断：checkInterruptWhileWaiting 返回 **THROW_IE**，最终**抛 InterruptedException**，并确保自己也会被**转运进 Sync Q**（避免挂死），随后**获取/释放一次锁**使状态一致，再抛出。
  - 若**在转运到 Sync Q 之后**才中断：不抛，而是**在成功重新获得锁后 selfInterrupt()**（把中断位补回来），交给上层处理。

  

- **awaitUninterruptibly()**：忽略中断，不抛；但中断位会在返回前被**重新设置**。

- **超时**：awaitNanos/awaitUntil/await(long,TimeUnit) 都会在超时后走**取消路径**，把节点标为取消并确保能**回到 Sync Q** 抢一次锁后返回（返回值指示是否在超时前被唤醒）。

> 这套设计保证：**不会丢失中断**、**不会把线程永远遗留在 Cond Q**。



# **4) 为何不会“丢通知”？为何还要** **while****判条件？**

- **不丢通知**：

  - 修改条件 + signal 都在**同一把锁**保护下；

  - signal 把等待者搬到 **Sync Q**；

  - 等待者**只有在重新获取了锁**后才从 await 返回。

    这条**锁获取的 happens-before** 链保证了**信号顺序与可见性**。

  

- **仍需 while 验证条件**：

  - 可能出现**伪唤醒**；

  - 也可能被 **signalAll** 或别的等待者先一步消费了条件；

  - 还有**超时/中断后**返回但条件仍不成立。

    所以标准写法必须是：

```
lock.lock();
try {
    while (!conditionHolds()) {
        cond.await();
    }
    // 条件成立，干正事
} finally { lock.unlock(); }
```



# **5) 与** **Object.wait/notify**的差别（速记）

- **多路条件**：一把锁可挂多条 Condition，每条独立的“等待集”；wait/notify 只有一条。
- **必须持锁**：await/signal 强制在持锁时调用；错误直接抛异常（更安全）。
- **队列分离**：Cond Q 与 Sync Q 明确分隔，signal 是“**搬运**”不是“直接交接锁”。





# **6) 常见坑 & 实战要点**

1. **一定用 while** 检查谓词，不要用 if。
2. signal/signalAll 要放在**修改共享状态之后**再调用，且**都在持锁期间**。
3. 一般建议：**唤醒最小集合**（先 signal），只有在**需要广播**时用 signalAll。
4. Condition **不保证 FIFO**；顺序由锁的获取策略（公平/非公平）+ 竞争时机决定。
5. 若等待者较多，signalAll 后的一次性“惊群”会引起**再竞争风暴**；评估成本。





## **一页总览（时序速记）**

```
await():
  addConditionWaiter(CONDITION) → fullyRelease(lock) → park 循环
  ←(signal/timeout/interrupt)→ 被转运到 SyncQ → acquireQueued(saved) → 重新持锁返回
  （视中断发生点：抛 IE 或 selfInterrupt）

signal():
  从 CondQ 取 first → transferForSignal：
    CONDITION→0 → enq 到 SyncQ 尾 → 设前驱 SIGNAL（或直接 unpark）
  （随后 unlock() 时按 AQS 规则从 head 唤醒最近的有效后继）
```

把这张图和上面的伪代码对上，你再回到 JDK 源码（ConditionObject.await/signal、transferForSignal、checkInterruptWhileWaiting、fullyRelease、isOnSyncQueue、acquireQueued），就能顺着“**加入条件队列 → 释放锁 → 等待 → 被搬运 → 重新抢锁 → 返回**”这条主线把每个细节定位清楚了。