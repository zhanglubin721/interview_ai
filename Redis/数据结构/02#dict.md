# **1）数据结构总览（dict / dictht / dictEntry / dictType）**

```
typedef struct dictEntry {
    void *key;
    union { void *val; uint64_t u64; int64_t s64; double d; } v;
    struct dictEntry *next;   // 拉链法处理碰撞（单链表）
} dictEntry;

typedef struct dictht {
    dictEntry **table;        // 桶数组（大小为 2^n）
    unsigned long size;       // 桶个数
    unsigned long sizemask;   // size-1，用于快速定位桶下标
    unsigned long used;       // 已存放的键值对数量
} dictht;

typedef struct dict {
    dictType *type;           // 钩子：哈希/比较/析构等函数指针
    void *privdata;           // 传给钩子的私有数据
    dictht ht[2];             // 两张表：ht[0] 主表，ht[1] 仅在 rehash 时启用
    long rehashidx;           // 迁移进行到的桶下标；-1 表示未在 rehash
    unsigned long iterators;  // 正在运行的安全迭代器数量
} dict;
```

- **dictType** 定义了本字典如何计算 hash、如何比较/复制/释放 key 和 value。Redis 为不同场景（DB、Set、Zset 等）定制不同的 dictType。
- **哈希函数**：现代 Redis 用 **SipHash**（带随机种子）防止碰撞攻击。
- 桶个数始终是 **2 的幂**，因此 index = hash & (size-1)。





# **2）扩容/收缩的触发条件（负载因子与目标尺寸）**

- **负载因子** load = used / size。

- 默认允许 resize 时（dict_can_resize = 1）：

  - 当 load >= 1（≈满桶）会触发**扩容**；

  

- 当全局暂时**禁止**普通 resize（比如持久化阶段可能会设置），只有当 load >= 5 时才**强制扩容**（dict_force_resize_ratio = 5）。

- **目标尺寸**：扩容或收缩时，目标桶数取 **不小于需要值的最小 2^n**。扩容通常以 used * 2 作为需要值；收缩则以 used 作为需要值。

> 直觉：Redis 追求“**装载因子在 1 左右**”，既节省内存，又保证常数级性能。





# **3）为什么要“渐进式” rehash？**

如果像 Java HashMap 那样一次性把所有桶拷到新表，单线程事件循环会被长时间阻塞，导致 **抖动/卡顿**。

Redis 采取 **incremental rehash（渐进式）**：把一次重活拆成很多小步，在后续的**普通操作**和**后台周期任务**里一点点完成，从而把延迟摊平到很多次 O(1) 操作中。





# **4）渐进式 rehash 的启动与状态**

- 当决定扩容/收缩时：

  1. 申请一张新表 ht[1]（大小是前面说的目标 2^n）。
  2. 置 rehashidx = 0，表示从旧表 ht[0] 的第 0 号桶开始“搬家”。

  

- **进行中**：rehashidx >= 0。

- **完成**：当旧表 ht[0].used == 0，则：

  - 释放旧表数组；
  - 把 ht[1] 作为新的 ht[0]；
  - 清空 ht[1]，置 rehashidx = -1（结束）。

  





# **5）每一步具体搬什么？（“按桶搬迁”，链上逐个搬）**

一次 **rehash step** 的核心动作：

```
while (ht[0].table[rehashidx] 为空) {
    rehashidx++                   // 跳过空桶：空桶不需要搬
}
把 ht[0].table[rehashidx] 链上的所有 entry 逐个从旧表摘下，
根据 新表的 sizemask 重新计算桶位，插入到 ht[1] 对应的桶链表头。
ht[0].table[rehashidx] 置空；rehashidx++。
```

- **按桶**迁移（不是按 entry 计数），但一个桶可能有多条链。
- Redis 实现中一次 step 可能搬迁**一个非空桶**，也可能在时间片内搬多桶（取决于调用方）。





# **6）谁来“驱动”这些小步？（两条驱动力）**

1. **惰性驱动**：绝大多数与字典相关的命令（查找/新增/删除）入口都会尝试做 dictRehashStep()，迁移**一小步**。每次对该 dict 的查/增/删都会“顺手搬一桶”（与 CHM 类似）。
2. **主动驱动**：server 定时任务（serverCron）里调用 dictRehashMilliseconds()，在每次周期里花一个很小的时间片（比如 ~1ms）推进多个桶的迁移。

> 这就保证了即使业务暂时没有对这个 dict 做很多操作，后台也会一点点推进，**不会永远卡在半路**。





# **7）迁移期间各类操作如何工作？**

**查找（find）**

- 先在旧表 ht[0] 查（考虑 rehashidx 左侧桶可能已搬空，但查了也只是空），再在新表 ht[1] 查；
- 因为所有新插入都会直接进 ht[1]（见下），所以两边都要看。



**插入/更新（add/replace）**

- 如果在 rehash 中：**一律插到 ht[1]**；
- 如果未在 rehash：插到 ht[0]。
- 这样可避免“刚插入的数据又要再搬一次”。



**删除（delete）**

- 同时在两张表里尝试删一遍即可（最多命中一边）。



**遍历（迭代）**

- 有 **安全（safe）** 和 **不安全（unsafe）** 两种迭代器：

  - **safe** 允许遍历时修改/删除，内部用 dict->iterators 计数和一些策略保证安全（库内部在关键点会减少激进的 rehash 推进来降低风险）；
  - **unsafe** 迭代时不允许结构性更改（否则会触发校验失败），换来更快的开销。

  

- Redis 的 SCAN/HSCAN/SSCAN 使用了 **位反转游标** 的扫描算法（见 #10），在扩容/收缩/rehash 期间也能做到 **不丢不重**（或可接受的“**少量重复**但不遗漏**”）。







# **8）复杂度与延迟**

- **单次查找/插入/删除**：

  - 平均 **O(1)**；
  - 若在 rehash 期，多了一次对另一个表的 O(1) 查找；
  - 每次操作通常还会额外推进 **一个桶** 的迁移，常数级。

  

- **最坏情况**：链很长会退化到 O(k)（k=该桶链长）。Redis 使用 SipHash 并且保持装载因子≈1，极大减小最坏情况概率。

- **延迟**：一次 rehash step 只是移动一个桶的若干 entry，耗时受该桶链长影响，但总体把一次性大开销摊到了很多次微小开销里，适合单线程事件循环。







# **9）一个“全过程”示意（ASCII）**

假设旧表 size=8（mask=7），used≈8，触发扩容 → 新表 size=16（mask=15）：

```
启动：
ht[0]: [b0][b1][b2][b3][b4][b5][b6][b7]   rehashidx=0
ht[1]: [  ][  ][  ][  ][  ][  ][  ][  ][  ][  ][  ][  ][  ][  ][  ][  ]

Step1: 搬 b0 链到 ht[1]（用新 mask 15 重新散列）
ht[0]: [  ][b1][b2][b3][b4][b5][b6][b7]   rehashidx=1
ht[1]: [..根据新桶位分布..]

Step2: 搬 b1
ht[0]: [  ][  ][b2][b3][b4][b5][b6][b7]   rehashidx=2
ht[1]: [..增长..]

... 重复直到 b7 也搬完

完成：
ht[0] <- ht[1]; 清空 ht[1]; rehashidx = -1
```

迁移中查找 key：**先 ht[0] 后 ht[1]**；插入：**只进 ht[1]**。





# **10）**dictScan的“位反转”扫描（支撑 SCAN 的关键）

- 哈希表扩容从 2^n → 2^(n+1) 时，一个旧桶 i 会分裂成新桶 i 和 i + 2^n（高位新增 1 位）。
- **位反转（bit-reversed cursor）** 遍历顺序能保证：当 size 变化时，所有“相同低位前缀”的桶会被成组扫描，不会因为扩容/收缩而**漏扫**；在 rehash 期间，Redis 同时对两张表按各自的 mask 做这套位反转游标扫描。
- 代价是：**可能出现少量重复**（官方接受的性质），但不会遗漏。这也是 SCAN 文档里说“可能重复、需去重”的原因。







# **11）与 Java** **HashMap**的对比（你熟悉的视角）

| **方面**   | **Redis dict**                                        | **Java HashMap（JDK 8）**                                    |
| ---------- | ----------------------------------------------------- | ------------------------------------------------------------ |
| 搬迁策略   | **渐进式**：两张表并存，按桶逐步迁移                  | **一次性 resize**：阈值触发后整表迁移（单线程时可能有较大停顿） |
| 桶结构     | 单链表（新版本仍以链表为主；极端碰撞靠 SipHash 避免） | 链表 + 树化（链长 ≥ 8 且容量 ≥ 64 时树化为红黑树）           |
| 触发策略   | 负载因子 ≈ 1 扩容；禁止期强制阈值=5                   | 默认负载因子 0.75（capacity×0.75 达到阈值）                  |
| 并发语义   | Redis 单线程模型；dict 本身不并发安全                 | HashMap 非线程安全；并发用 ConcurrentHashMap                 |
| 遍历与变化 | dictScan 位反转游标，rehash 期可迭代；可能重复        | 迭代期间结构改变会 ConcurrentModificationException（除非弱一致性的并发容器） |

核心差异：Redis **必须控制延迟**，所以牺牲了实现简单性，换来了 **无停顿的重散列**。





# **12）常见问答 & 易错点**

**Q1：rehash 期间新元素插到哪里？**

A：**只插到 ht[1]**。旧表只出不进。



**Q2：查找为什么要查两张表？**

A：因为旧表上的键在逐步搬、而新键直接进新表，只有查两张表才完整。



**Q3：什么时候结束？**

A：当 ht[0].used == 0，把 ht[1] 变成新的 ht[0]，rehashidx=-1。



**Q4：rehash 会不会被“暂停”？**

A：不会彻底停，但**推进速度**会受运行态影响：

- 每次字典操作的“顺手一步”；

- 后台周期任务给的时间片；

- 存在 **安全迭代器** 时，库会更保守地推进（以保证迭代安全）；

  总之它是“**尽可能快但不阻塞**”地前进。



**Q5：最坏碰撞怎么办？**

A：使用 **SipHash+随机种子** 大幅降低对抗性碰撞风险；装载因子≈1 也限制了平均链长。



**Q6：为什么装载因子是 1 而不是 0.75？**

A：Redis 更重视 **读写常数因子 + 内存效率** 的折中。链表查找在平均 1 左右的装载因子下仍是常数期望；而装载因子再低内存浪费明显。



**Q7：收缩什么时候发生？**

A：当数据量显著下降时调用 dictResize，把目标尺寸降到不小于 used 的最小 2^n；同样通过渐进式 rehash 完成收缩。







# **13）伪代码梳理（把关键动作串起来）**

**插入（add）**（忽略细节校验）：

```
if (rehashidx != -1) dictRehashStep();          // 顺手搬一步
ht = (rehashidx == -1) ? &ht[0] : &ht[1];       // 迁移期只往新表插
idx = hash(key) & ht->sizemask;
if (key 已存在) return REPLACE/ERROR;
插入链表头；ht->used++;
如果需要（load 因子超阈值 && 非迁移期）启动 rehash（构造 ht[1]，rehashidx=0）
```

**查找（find）**：

```
if (rehashidx != -1) dictRehashStep();
for t in {0,1}:
  h = hash(key) & ht[t].sizemask;
  在 ht[t].table[h] 链上找；找到即返回
return NULL;
```

**迁移一步（rehash step）**：

```
if (rehashidx == -1) return;
搬运从 ht[0].table[rehashidx] 开始的下一个“非空桶”；
rehashidx++；
如果 ht[0].used == 0 => 结束流程（swap 表，rehashidx=-1）
```







# **14）你可以怎么在面试里高效作答（要点版）**

1. **两张表 + rehashidx**：ht[0] 出、ht[1] 进；rehashidx 标记“搬到哪了”。
2. **触发条件**：负载因子≈1 扩容；禁止期≥5 强制扩容；收缩把目标定到 used。
3. **推进机制**：每次操作顺手搬一桶 + 后台定时器；**无停顿**。
4. **迁移期行为**：查两表；只往新表插；删除两表都试。
5. **SCAN 原理**：位反转游标，在 rehash 期间 **不漏，可能少量重复**。
6. **为什么不用树化**：Redis 通过 SipHash 限制碰撞；链表 + 低负载因子即可；实现更简单、常数好。
7. **与 Java HashMap 对比**：Java 一次性 resize 可能卡顿；Redis 渐进式迁移以避免长尾延迟。