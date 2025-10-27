# **1) 抽象与排序规则**

- 每个 **member**（唯一）对应一个 **score**（浮点 double）。
- 全序定义：先按 **score 升序** 排；score 相等时再按 **member 的字典序**（二进制安全的 *lex*，通常是 SDS 的字节序）排。
- 这保证了「稳定且确定」的次序：(score, member) 的**词典序**。







# **2) 物理实现：两套结构并存（查快 + 排快）**

3) 典型（非小型）zset 使用 **「dict + skiplist」** 双结构：

```
typedef struct zset {
    dict      *dict;    // member -> score   （O(1) 直查）
    zskiplist *zsl;     // (score,member) 有序索引（O(logN) 排序/范围）
} zset;
```



## **2.1 dict（哈希表）**

- key = member（SDS），value = score（double）。
- 支撑 **ZSCORE / ZADD-XX / NX / LT / GT** 等需要 O(1) 知道“是否存在/原分数”的场景。





## **2.2 跳表 zskiplist（按 (score,member) 排序的有序索引）**

关键结构体（易化版）：

```
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward; // 便于从尾部逆向
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 这一层的“前进指针”
        unsigned long span;            // 跨越了多少个节点（用于求 rank）
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    zskiplistNode *header, *tail;
    unsigned long length;
    int level; // 当前最高层数（<= MAXLEVEL，比如 32/64）
} zskiplist;
```

- **多层前进指针**：像“多级高速公路”。查找用上层快速越过大片元素，最后在最底层（level 0）落到精确位置。
- **span（跨度）**：每层的指针还记录“这一步跨过了多少个底层节点”。它让 **ZRANK** 这类“求排名”在 O(logN) 完成（沿路把跨过的 span 累加即可）。

**随机层数生成**：插入时用几何分布决定新节点层高，常用参数 p=1/4，MAXLEVEL≈32/64。期望高度 O(logN)。

> 有了 p=1/4，每升一层的概率是 1/4；层高分布稀疏，保证平均查找链路长度是 O(logN)。





# **3) 核心操作与复杂度（典型路径）**

> 记住一个总规律：**定位 O(logN)**，之后顺着 level 0 走 **k** 个元素 ⇒ **O(logN + k)**。

### **3.1 ZADD（插入/更新）**

- 先在 **dict** 查 member 是否存在（O(1)）：
  - **不存在**：直接在 **skiplist** 里按 (score,member) 定位插入（O(logN)），在 **dict** 写入映射。
  - **已存在**：
    - 若新 score 与旧 score 不同：**从跳表删除旧节点**，再按新分数重新插入（O(logN)）；dict 里更新分数。
    - 若相同分数但 member 相同：跳表**位置不变**（已满足排序），视选项（XX/NX/LT/GT/INCR）决定是否更新。
- 复杂度：O(logN)。



### **3.2 ZREM（删除）**

- dict 找到旧 score，**跳表删节点**（需要沿着各层修指针+span，O(logN)），dict 中删条目。
- 复杂度：O(logN)。



### **3.3 ZSCORE**

- 直接 dict：O(1)。



### **3.4 ZRANGE / ZRANGE BYSCORE / BYLEX**

- **BYSCORE**：跳表按 min 定位到首个命中节点（O(logN)），然后在 level 0 线性走 **k** 个（O(k)）。
- **BYLEX**：限定 *同一分数* 的成员按字典序范围取；跳表中同分数段是连续的，定位起点后向前走。
- **普通 ZRANGE（按 rank）**：借助 span 快速定位第 *rank* 个元素：O(logN + k)。



### **3.5 ZRANK / ZREVRANK**

- 沿多层前进指针搜索目标，同时把途经边的 span 加总得到“前面有多少个节点” ⇒ O(logN)。



### **3.6 ZPOPMIN / ZPOPMAX**

- 取 **表头/表尾**（header->level[0].forward / tail），删除需要修各层指针 ⇒ O(logN)。
- 因为有 **backward** 指针，取最大值也很快。



### **3.7 批量按分数/排名删除**

- ZREMRANGEBYSCORE / ...BYRANK：先 O(logN) 定位范围端点，再按 level 0 迭代删除 **k** 个，整体 O(logN + k)。







# **4) 为什么选「跳表 + dict」，而不是“单棵红黑树”？**

- **实现简单 & 常数好**：跳表代码短、易维护；随机层带来良好的平均常数。
- **排名（rank）友好**：span 让“第 k 名是谁 / 这个 member 是第几名”在 O(logN) 实现，写树要额外维护子树大小也行，但复杂度更高。
- **两套结构分责明确**：
  - dict 做 **存在性 + O(1) 查 score**；
  - 跳表做 **排序 + 范围/排名**。
- **Redis 单线程**：无锁操作，跳表旋转/回溯少；链式指针修改在单线程里可控、低抖动。







# **5) 小集合的** **listpack**编码（节省内存）

当 zset **很小**（元素数与元素长度都未超过阈值）时，为极致省内存，Redis 用**紧凑连续内存** listpack 来存储（而不是 dict+skiplist）：

- 内部以 **有序对** [member, score, member, score, ...] 形式按 (score,member) 排序存放；
- 查找/插入需要 **线性扫描/移动**，但因元素少，常数仍然很小；
- 一旦规模/元素长度超过阈值，会**转换为「dict+skiplist」** 编码，后续操作走上面那套 O(logN)。

> 你可以把它理解为：**小用 array（紧凑、省内存），大用 index（跳表）**。这是 Redis 常见的“多编码切换”哲学。







# **6) 源码级直觉（伪代码）**

## **6.1 插入（简化）**

```
zadd(key, member, score):
  if dictFind(zs->dict, member, &oldscore):
     if score != oldscore:
        zslDelete(zs->zsl, oldscore, member)  // O(logN)
        zslInsert(zs->zsl, score, member)     // O(logN)
        dictSet(zs->dict, member, score)
     else:
        // 可能根据选项（XX/LT/GT/INCR）调整
  else:
     zslInsert(zs->zsl, score, member)        // O(logN)
     dictAdd(zs->dict, member, score)
```



## **6.2 跳表查找 rank（利用 span）**

```
zslGetRank(zsl, score, member):
  rank = 0
  x = zsl->header
  for level from (zsl->level-1) downto 0:
     while x->level[level].forward 
           and (x->level[level].forward->score < score
                or (==score and forward->ele < member)):
        rank += x->level[level].span
        x = x->level[level].forward
  // x 现在在目标之前的最后一个节点
  x = x->level[0].forward
  if x && x->score==score && x->ele==member:
     return rank + 1  // 1-based
  else return 0       // 不存在
```



## **6.3 插入节点的随机层（典型）**

```
int zslRandomLevel() {
  level = 1
  while (rand() < p * RAND_MAX && level < MAXLEVEL) level++
  return level
}
```







# **7) ASCII 直观图：多层前进 + span 求 rank**

```
Level 3:  [H] ----(span=8)----> [N9]
Level 2:  [H] --(3)-> [N3] --(5)-> [N9]
Level 1:  [H] -(1)-> [N1] -(2)-> [N3] -(3)-> [N6] -(3)-> [N9]
Level 0:  [H] -> N1 -> N2 -> N3 -> N4 -> N5 -> N6 -> N7 -> N8 -> N9 -> ...
            ^                                ^
         rank=1                           rank=6
```

- 想找 N6（第 6 个）：在高层跳跃累加 span：

  - L3：直接到 N9 超了，停
  - L2：H→N3（+3），N3→N9 超了，停
  - L1：N3→N6（+3） ⇒ 总计 3+3=6

  

- 到 L0 校验命中即得 rank。







# **8) 复杂度小结与常见问答**

- **ZADD/ZREM/ZSCORE/ZRANK**：O(logN)（ZSCORE 是 O(1)，因 dict）。
- **范围类（BYSCORE/LEX/RANK）**：O(logN + k)。
- **ZPOPMIN/ZPOPMAX**：O(logN)（删除需修各层指针）。
- **为什么 score 相等要看 member**：保证全序、可重复稳定；同时避免“同分数组合”操作时的二义性。
- **浮点分数比较**：内部使用 double，比较时按 IEEE 规则（还要考虑 ±inf、NaN 的处理；命令层会做边界语义解析，支持 (+|-)inf 与 (x/[x 的开闭区间）。







# **9) 与 Java 视角类比（记忆点）**

- **像 NavigableMap<Double, Set<String>>**，但用 **跳表** 代替红黑树，并把“集合的 membership”外接到 **dict** 做 O(1) 查。
- **ZRANK ≈ order statistic**，跳表用 span 自带“子树大小”的效果。
- **范围扫描**像 subMap/tailMap，复杂度 O(logN + k)。







# **10) 实战建议（面试&生产）**

- **热路径**：大量 ZRANGE(BYSCORE) / ZRANK / ZADD；理解 “O(logN + k)” 的 k 很关键。
- **键/成员长度**：小而多 → 尽量让其停留在 **listpack** 编码（省内存）；一旦超过阈值会自动转 **dict+skiplist**，性能转向 O(logN)。
- **大批量删除**：优先用 ...BYSCORE / ...BYRANK 一次删范围，避免逐个 ZREM。
- **去抖动**：单线程下，所有这些操作都“锁内”执行，无并发条件竞争；只需注意单次操作量（k）别太大。