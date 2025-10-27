# **1) 申请内存：zmalloc → jemalloc**

## **路径**

```
你的命令 → 创建对象/结构（SDS/dict/quicklist/skiplist/rax/...）
       → 调用 zmalloc()/zcalloc()/zrealloc()
       → 底层调用 jemalloc（mallocx/rtree/bins/arenas）
       → 返回指针；zmalloc 同时把“真实分配字节数”计入 used_memory
```



## **jemalloc 怎么分**

- **多 arena**：降低多线程争用（Redis 主线程 + 后台线程/模块线程）。
- **按 size-class 分 bin**：8, 16, 32…512B（小对象），再到 1KB、2KB…（大对象）；小对象放进 **slab(run)** 里按 **region** 切块。
- **大对象**：直接以 **extent**（多页）分配。
- **tcache**：线程本地缓存，加速小块申请/释放。

这套设计让频繁的小对象（dictEntry、SDS 小字符串等）申请非常快，但也引入**尺寸对齐**与**slab 内空洞**等碎片来源（见 §3）。



# **2) 释放与回收：同步释放 + 惰性释放 + 回收给 OS**

## **同步释放（命令删除/覆盖）**

- Redis 对象生命周期结束 → 调用 **zfree()** → 交给 jemalloc free()。

- jemalloc 把这个块标记为空闲：

  - 小块：回到所在 slab 的空闲链；
  - 大块/extent：可能变“dirty/retained”，等待**清理/合并**。

  



## **惰性释放（lazyfree）**

为避免一次性 free 很多对象卡主线程，Redis 提供**异步释放**选项与命令：

- UNLINK key / FLUSHDB ASYNC / 过期淘汰的懒释放；
- 配置：lazyfree-lazy-eviction、lazyfree-lazy-expire、lazyfree-lazy-server-del；
- 实现：把“析构工作”扔到 **bio 后台线程**，主线程尽快返回。





## **把空闲内存还给操作系统**

- **jemalloc 后台清理线程**（可开）：周期 **purge** “dirty/retained” extents，用 madvise(MADV_DONTNEED/FREE) 归还物理页。
  - Redis 配置：jemalloc-bg-thread yes（如果编译的 jemalloc 支持）。
- 系统层：Linux 页回收、overcommit_memory 等也影响 RSS 波动。



# **3) 碎片：从哪来 & 怎么量**

## **主要来源**

1. **尺寸对齐**：要对齐到 size-class，申请 33B 实际占 48/64B。
2. **slab 内空洞**：同一 slab 里混杂生命周期不同的小对象，释放后留下“零散坑位”。
3. **大对象切分/合并不完全**：大 extent 反复分配/释放会产生不可立即合并的残片。
4. **复制/COW 放大**：BGSAVE/AOF 重写 fork() 后，父进程修改页导致 **写时复制**，短期内 RSS 膨胀；结束后才回落。
5. **编码切换/对象变大**：比如 listpack → hashtable/skiplist 后对象更“碎”。



很好，我就直接讲 **jemalloc 是如何在内部对“内存碎片”动手术**——从“怎么切分/回收”到“选择策略/后台整理/与 OS 交互”。先把“碎片”拆成两类：



- **内部碎片**：按 size-class 对齐后没用到的尾巴；以及 slab（run）里零散空洞。
- **外部碎片**：地址空间里有空闲，但无法满足某次“更大/更对齐”的分配。





------





# **1) 结构：arena / bin / slab(run) / extent（铺好路再开车）**

- **arena**：分配器的分区（多线程降低锁竞争）。Redis 也会有后台线程，所以 jemalloc 的多 arena 有意义。
- **size-class**：把大小离散化（8,16,24…；具体取决于版本与页大小）以**上取整**，控制内部碎片上界。
- **bin**（小对象类别）：每个 bin 对应一个 size-class。
- **slab(run)**：若干页构成的“大块”，被**等分**成同尺寸的小“region”；每个 slab 用 **bitmap** 记录哪些 region 空闲。
- **extent**：页粒度的大块（大/巨对象直接从 extent 来），可**split/merge** 和**对齐**，是对外部碎片的“砖块”。

> 关键思路：**小的走 slab 等分**（尽量把同类对象塞在一起，减少空洞），**大的走 extent**（靠分裂/合并控制外部碎片）。



# **2) 小对象：尽量把“未满的盘子”先吃完（减少内部碎片）**

**目标**：让“**部分填满的 slab 越少越好**”，这样 bitmap 中的空洞集中在少量 slab 上，其它 slab 能完全满或完全空，空的 slab 就能整体回收，减少内部碎片和 RSS 压力。

**做法（bin 的 slab 选择策略）**：

1. **优先用当前 slab（slabcur）**：局部性最好。
2. 若 slabcur 没空位，就从 **非满集合（slabs_nonfull）里选“最满的那块”**（即**剩余空位最少**的 slab）继续填——**先吃快吃完的盘子**。
3. 实在没有，就向 arena 申请**新的 slab**（按该 size-class 选定的 run 大小，使“最后一页的浪费最小”）。

**释放**：free 时把对应 bit 清 0；slab 若变空 → 返回到 arena，可进一步回收页（见 §4）。

这套策略把**同 size-class 的对象尽量挤在少数 slab**，从分配器视角“内部碎片被压缩到少数载体”，方便统一回收。



# **3) 大对象：extent 的分裂/合并 + 最佳匹配，控制外部碎片**

**目标**：当你要一块 N KiB 的内存，尽量从现有空闲 extents 中**找最贴合**的，不够就**分裂**大块；释放时尽量**合并相邻**，把“碎砖”拼成大砖，为下次大请求腾路。

**做法**：

- **空闲 extent 的三层“缓存”**（不同语义，名字可能见到 *dirty/muzzy/retained*）：
  - **dirty**：最近释放，有脏内容；
  - **muzzy**：给 OS 标注“可丢弃”（MADV_FREE），被动回收；
  - **retained**：彻底从活动集剥离（或 MADV_DONTNEED），但地址映射关系可能保留以便快速再用。
- **映射结构**：用按地址/大小索引的树或哈希（jemalloc 里有 extent map/rtree），能判断**相邻**从而合并。
- **分配路径**：先在空闲集合里按大小**近似 best-fit**找块，必要时**split**；
- **释放路径**：把块放回 extent cache，并尝试**与前后相邻块 coalesce**，越大块越好（为后续请求减少外部碎片）。



# **4) 和操作系统“拔河”：purge/decay 背景回收，压低 RSS**

内部/外部碎片最终都会映射成**进程 RSS 偏高**。jemalloc 有两种回收动作（可由**后台线程**周期处理）：

1. **purge（主动清理）**：把空闲页 madvise(MADV_DONTNEED)，物理页归还给 OS（RSS 下降，地址保留）。
2. **decay（衰减时钟）**：空闲页不是马上 purge，而是放一段“冷却期”，按**decay 时间参数**逐步转为 muzzy/retained，再 purge。
   - 这样避免“抖动型负载”导致频繁释放/再分配。
   - 可调：dirty_decay_ms / muzzy_decay_ms（在支持的构建里），以及**后台线程**开关（如 jemalloc-bg-thread，由上层应用/构建决定）。

**效果**：即便 slab 内有空洞，只要某些 slab 完全空了，就能把对应页 purge；对于 extent，也能把大块回给 OS，**缓解“看起来碎”的 RSS**。



# **5) 现场“修补”：原位扩/缩（xallocx）与对齐策略**

- **原位扩展/收缩**：xallocx/rallocx 尝试在**不搬家**的情况下调整对象大小：

  - 若同 slab 里相邻 region 空闲 → 可以**原位扩**；

  - 对大对象，若相邻 extent 可合并 → 也能**原位扩**；否则再走“新块 + 拷贝”。

    这能减少“搬家引入的新碎片”和复制开销。

- **对齐**：大对象/特殊对齐需求时，jemalloc 会选择合适的 extent/分裂策略达成对齐，尽量不制造额外外部碎片（找最小满足对齐的块）。



# **6) tcache 与多 arena：性能与碎片的平衡**

- **tcache（线程缓存）**：缓存少量小对象的“最近释放”，**极大降低锁竞争**与系统调用，但短期看会让空闲块**留在本线程手里**，延迟回流到 slab/arena，从而**延迟**碎片被集中与回收。jemalloc 会在阈值/时间到达时**刷回**。
- **多 arena**：减少跨线程争用，也能“把碎片隔离在各自 arena 内”，避免全局污染；但 arena 太多时会让**全局可合并的空闲被分散**。jemalloc 给的是折中默认值；线程不多的应用（如 Redis 主线程）过多 arena 反而**不利于聚合空闲**。





# **7) 观测与调参（你真的能“摸”到的）**

- **观测**：mallctl/jemalloc stats 能看到 allocated/active/resident、各 bin 的“满/非满 slab”数、extents 的 dirty/muzzy/retained 等指标。

- **常用调参开关**（具体名随版本略异）：

  - 开启 **后台线程**：让 purge/decay 自动进行；
  - 设置 **dirty/muzzy decay**：平衡“回 RSS 速度”和“避免抖动”；
  - 控制 **tcache 大小/刷新**：抑制短期堆积；
  - **arena 数量**：线程少时不必太多；
  - 对上层（如 Redis）再配合 **active defrag**：应用级“搬家”把对象挤到紧凑 slab，jemalloc 更容易把整块页 purge 出去（这是 Redis 专属的额外“外科手术”）。

  





## **一口气总结（按“碎片类型 → jemalloc 手段”记忆）**

- **内部碎片（小对象）**：**等分 slab + 选“最满”的非满 slab优先** → 集中空洞，整 slab 释放；bitmap 精细管理。
- **外部碎片（大对象）**：**extent 分裂/合并 + best-fit**；空闲页**decay/purge** 回 OS。
- **运行中修补**：**原位扩/缩**，尽量不搬家。
- **系统级收口**：**后台线程 + decay 策略** 让 RSS 回落、避免抖动。
- **权衡项**：**tcache/arena** 带来吞吐与碎片的博弈，默认多为折中。