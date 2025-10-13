# **1) CAS 的语义（抽象定义 & 线性化点）**

对某个内存位置 addr，给定**期望值** expected 和**新值** update，执行：

- 如果 *addr == expected：**原子地**把 *addr 改为 update，返回 **true**；
- 否则：不修改，返回 **false**。

> **线性化点（linearization point）**：一次 CAS 成功的那一刻。并发正确性证明通常以该瞬间为“原子事件”。



**Java API 对应**

- AtomicXXX.compareAndSet(expected, update)
- JDK 9+ VarHandle.compareAndSet / compareAndExchange（返回旧值）







# **2) JMM（Java 内存模型）下的内存语义**

CAS 不只是“值比较”，还携带**内存可见性/有序性（happens-before）**。以常用操作为例（精简版）：

| **操作**                | **读/写** | **内存序（JMM语义）**                         | **说明**                 |
| ----------------------- | --------- | --------------------------------------------- | ------------------------ |
| get()                   | 读        | **volatile 读（acquire）**                    | 读到之前发布的值         |
| set(v)                  | 写        | **volatile 写（release）**                    | 释放前序写               |
| lazySet(v)              | 写        | **release**（弱于 volatile 写）               | 允许“稍后”对外可见       |
| compareAndSet(e,u) 成功 | 读+写     | **相当于一次 volatile 读 + 一次 volatile 写** | 建立 hb                  |
| weakCompareAndSet…      | 读+写     | 与变体相符（Plain/Acquire/Release/Volatile）  | 可能**偶发失败**，须循环 |
| VarHandle 族            | 读/写/CAS | Acquire/Release/Volatile/Plain 精细可选       | 性能/语义平衡            |

> 直觉：CAS 成功就像“先读确认旧值→再做一次发布写”，把**之前的普通写**一并**发布**出去；CAS 失败至少做了**一次 volatile 读**。







# **3) 硬件层：CAS 与 LL/SC、围栏与一致性**

**两大实现范式**

- **CAS 指令**（x86）：lock cmpxchg、lock xadd 等，带**总线锁/缓存一致性**，通常等价**全栅栏**（至少强于 acquire/release）。
- **LL/SC（Load-Linked/Store-Conditional）**（ARM/RISC-V）：ldxr/ldaxr + stxr/stlxr 循环配合 dmb 或 A/R 语义，避免“伪共享写放大”。

**缓存一致性（MESI/…）保证了：当 CAS 成功写入更新时，该缓存行的所有副本在其他核上失效/更新**，从而被**看见**。







# **4) CAS 循环（典型用法 & 代码范式）**

绝大多数“读-改-写”都要写成 **CAS 循环**（失败则重试）：

```java
AtomicLong counter = new AtomicLong();

long incrementAndGet() {
    long prev, next;
    do {
        prev = counter.get();      // volatile 读
        next = prev + 1;           // 纯计算
    } while (!counter.compareAndSet(prev, next)); // 失败重试
    return next;                   // 线性化点 = CAS 成功
}
```

**函数式更新**（JDK8+）会把用户函数包在 CAS 循环里，**可能被多次调用**，所以函数必须**无副作用**：

```java
long updateAndGet(LongUnaryOperator op) {
    long p, n;
    do {
        p = counter.get();
        n = op.applyAsLong(p);     // 可能重复执行
    } while (!counter.compareAndSet(p, n));
    return n;
}
```







# **5) 进展性（progress）与正确性**

- **无锁（lock-free）**：系统整体在有限步内有**某个**线程能前进（CAS 常见保证）。
- **等待自由（wait-free）**：每个线程在有限步内完成（更强、实现难）。
- **阻塞自由（obstruction-free）**：若不被竞争打断则能完成。

> CAS 算法通常是 **lock-free**：有竞争时个别线程会反复失败，但总体在前进。







# **6) ABA 问题（什么时候致命、如何处理）**

**现象**：位置从 A → B → A，CAS 仅比较“是否等于 A”，会**误以为没变**而成功。

- 对**数值累加/计数**这类只关心“现在值”的逻辑，**ABA 无害**。
- 对**无锁链表/栈/队列**等需要确认“期间未被更改”的结构，**ABA 致命**（可能丢节点/错链接）。



**对策**

- AtomicStampedReference<T>：引用 + **版本号**（每次成功 stamp+1）
- AtomicMarkableReference<T>：引用 + **布尔标记**（常用于逻辑删除）
- 在结构中并入“**世代/版本位**”一并 CAS





# **7) 与** **volatile**、锁 的关系（各司其职）

- **volatile**：给**可见性/有序性**，**不**给“读改写原子性” → x++ 用 volatile 仍会丢更新。
- **CAS**：给**单内存位置**的原子读改写；配合 JMM 具备发布/获取语义。
- **锁**：可把**多个位置**的更新变成一个**大原子区**；还能提供**互斥**与**条件等待**（wait/notify），语义更强但可能阻塞。

> 选择：单点 RMW → **CAS/原子类**；多位置一致性/复合事务 → **锁**；高并发计数 → **LongAdder**（分段 CAS，sum() 近似快照）。







# **8) 典型模式与实战要点**

## **8.1 计数/统计（CAS 循环 vs LongAdder）**

- AtomicLong：精确线性化值；高争用下 CAS 失败多、乒乓重。
- LongAdder：把计数拆到 base + cells[]，不同段上**并行 CAS**，吞吐高；sum() 是**近似快照**。





## **8.2 无锁栈（Treiber Stack）骨架**

```
class Node { Node next; int v; }
AtomicReference<Node> head = new AtomicReference<>();

void push(Node x) {
    Node h;
    do { h = head.get(); x.next = h; }
    while (!head.compareAndSet(h, x));      // 线性化点
}

Node pop() {
    Node h, n;
    do {
        h = head.get();
        if (h == null) return null;
        n = h.next;
    } while (!head.compareAndSet(h, n));
    return h;
}
```

> 需要关注 **ABA**（可用 AtomicStampedReference 修补）。





## **8.3 Michael–Scott 队列（无界无锁队列）**

- 基于 **双指针（head/tail）** 的两处 CAS；JUC 的 ConcurrentLinkedQueue 类似思想（另有细节优化与 GC 考量）。





## **8.4 回退与抖动控制**

- **指数回退（exponential backoff）**：CAS 失败后短暂 parkNanos 或自旋 delay，缓和热点；
- **分段/消除**：比如 **elimination backoff stack** 在高争用下通过配对消除相反操作；
- **伪共享**：对齐到 cache line，使用 @Contended 或手工填充。







# **9) 常见坑 & FAQ**

1. **“volatile + i++ 还会丢更新？”**

   会。i++ 是 **读→算→写** 三步；需要 **原子 RMW**（CAS 或锁）。

2. **“CAS 一定快速吗？”**

   低争用下快；高争用下可能**失败-自旋**导致抖动，且有**缓存线乒乓**。这种场景用 **LongAdder** 或**加锁合批**更合适。

3. **“weakCompareAndSet 为何偶发失败？”**

   为了给 JVM/CPU 更多优化空间（例如更弱的围栏/失败路径），**必须在循环中使用**。JDK 9 的 VarHandle 系列明确内存序变体。

4. **“CAS 能跨多个变量吗？”**

   不行。CAS 只对**一个位置**原子；跨多个位置必须用锁或两阶段协议。

5. **“CAS 成功的内存效果有多强？”**

   至少达到 **volatile 写** 的发布语义；配合之前的读，建立了 hb 链。x86 的 lock 指令通常更强（近似全栅栏），ARM 用 acq/rel 变体。







# **10) 一页速记（面试/白板）**

- **定义**：if (*p == exp) *p = upd（原子）；成功处是**线性化点**

- **JMM**：CAS 成功 ≈ **volatile 读+写**；失败 ≥ **volatile 读**

- **CPU**：x86 lock cmpxchg；ARM ldxr/stxr 循环 + ldar/stlr/dmb

- **优点**：无阻塞、线程切换少、lock-free 进展

- **缺点**：高争用失败多、缓存线乒乓、**ABA**

- **对策**：回退/分段（LongAdder）、AtomicStampedReference 防 ABA

- **选型**：

  - 单点精确 RMW → AtomicLong/Reference（CAS）
  - 高并发计数 → LongAdder（分段）
  - 跨变量一致性/条件等待 → synchronized/Lock

  