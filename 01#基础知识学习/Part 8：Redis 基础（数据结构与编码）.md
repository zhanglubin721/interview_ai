### #8.1.1

#### redisObject

大纲

1. redisObject 在整个数据结构里的位置
2. 结构体长啥样
3. 四个核心字段：type / encoding / lru(LFU) / refcount
4. 命令执行过程中是怎么用它的
5. 面试/总结版怎么说



##### **1. redisObject 在哪儿？**

每个 DB 里本质就是一个 **dict**：

```
/* 简化理解 */
dict<key_sds, redisObject*>
```

- **key**：一个 sds（简单动态字符串）
- **value**：一个指针，指向 redisObject

所以：

- 你 SET k v，实际上是：
  - 把 "k" 变成一个 sds 作为哈希表的 key
  - value 是一个 redisObject*，里面封装了：类型、编码、淘汰信息、引用计数、以及真正数据结构的指针

也就是说：**redisObject 是 Redis 对“值对象”的统一包装头（header）**。





##### **2. redisObject 结构长什么样**

Redis 7 里大致是这样（删掉了注释和宏）： 

```
typedef struct redisObject {
    unsigned type:4;       // 类型：string/list/hash/set/zset/stream...
    unsigned encoding:4;   // 编码：raw/embstr/listpack/quicklist/intset/...
    unsigned lru:LRU_BITS; // LRU 时间 或 LFU 的计数+时间
    int refcount;          // 引用计数
    void *ptr;             // 指向底层真正数据结构
} robj;
```

- type + encoding + lru 组合占 4 字节

- refcount 4 字节

- ptr 在 64 位系统上一般 8 字节

  → **redisObject 头本身大约 16 字节**（不含真实数据）。 





##### **3. 四个关键字段逐个讲**

###### 3.1type：是什么数据类型

type 是一个 4bit 的枚举，比如： 

- OBJ_STRING
- OBJ_LIST
- OBJ_HASH
- OBJ_SET
- OBJ_ZSET
- OBJ_STREAM
- ……

作用：

1. **类型检查**：执行命令前会先看 type

   - GET 只能操作 OBJ_STRING

   - HSET 只能操作 OBJ_HASH

     不匹配就直接返回 WRONGTYPE 错误。

2. **选择一套逻辑分支**：

   比如内部有 lookupKeyReadOrReply → 拿到 redisObject → 根据 type 决定调用哪个模块的 API。





###### 3.2 encoding：底层怎么存

同一个 type，可以有多种 encoding。这是你前面问过的那一堆： 

- OBJ_STRING：

  - OBJ_ENCODING_INT：小整数，直接 long long
  - OBJ_ENCODING_EMBSTR：短字符串，一块内存里嵌着 redisObject + sdshdr + buf
  - OBJ_ENCODING_RAW：普通 SDS 指针

  

- OBJ_LIST：

  - 历史：ziplist / linkedlist
  - 现在：基本是 quicklist（内部再嵌 ziplist/listpack）

  

- OBJ_HASH：

  - 小：listpack（旧版本 ziplist）
  - 大：hashtable

  

- OBJ_SET：

  - 小且全是整数：intset
  - 否则：hashtable

  

- OBJ_ZSET：

  - 小：listpack
  - 大：skiplist + dict

  

- OBJ_STREAM：

  - stream：Radix tree + listpack

  

**type 决定“这是一个什么 Redis 类型”，encoding 决定“底下具体用什么数据结构”**。

而 ptr 指针就指向这个数据结构，例如： 

- type = OBJ_LIST, encoding = OBJ_ENCODING_QUICKLIST

  → ptr 指向一个 quicklist*

- type = OBJ_STRING, encoding = OBJ_ENCODING_RAW

  → ptr 指向一个 sds

- type = OBJ_ZSET, encoding = OBJ_ENCODING_SKIPLIST

  → ptr 通常指向一个 zset，里面再包含 dict + skiplist

很多命令实现里，会先根据 encoding 分发到不同路径，比如 list 的操作函数里会写类似：

```
if (o->encoding == OBJ_ENCODING_QUICKLIST) {
    // quicklist 操作
} else {
    // listpack 操作（小对象）
}
```



###### 3.3 lru：LRU / LFU 复用字段

这个字段的注释很关键： 

> /* LRU time (relative to global lru_clock) or LFU data (least significant 8 bits frequency and most significant 16 bits access time). */

也就是说：**同一个字段，在不同淘汰策略下语义不一样**：



① 当使用 LRU 相关策略时（allkeys-lru / volatile-lru）

- lru 存的是一个近似的访问时间戳：

  - 它用的是一个全局的 lru_clock（通常 24 bit，单位粗粒度秒级）。

- 每次访问这个对象，都会更新它的 lru 为当前 clock。

- 当需要淘汰 key 时：

  - Redis 会采样一批 key
  - 对比它们的 lru，谁“时间更久没被访问”（idle time 更长），谁就优先被淘汰。 

  



② 当使用 LFU 策略时（allkeys-lfu / volatile-lfu）

这时 lru 被当成 **LFU 复合字段** 使用： 

- 高 16 bit：大致访问时间（用于衰减频次）
- 低 8 bit：访问频率计数（LFU 计数器）



访问时会：

- 检查距离上次访问时间是否足够久，如果久 → 衰减频次
- 然后再对低 8 bit 的频率做随机递增（防止轻微访问就堆到极高）



淘汰时：

- 采样一批 key
- 选 lfu counter 最小的，优先淘汰

> 简单记：**lru 字段 = “谁最近用过 / 谁经常用” 的统计信息，是 LRU/LFU 淘汰算法的核心数据来源**。



Step 1：先做“时间衰减”

- 拿到当前的“LFU 时间刻度”：now
- 从高 16 bit 里取出 ldt（上次访问的时间片）
- 计算 delta = now - ldt
- 如果 delta 足够大（比如超过若干分钟/小时的阈值），就认为“这段时间没怎么用了”，要衰减 counter：
  - counter 会被按一定规则**往下减 / 右移（比如减半）**
  - 更新 ldt = now

这样：

- 很久没访问的 key，即便之前 counter 再高，也会被逐步衰减下来
- 频繁被访问的 key，因为访问很密集，delta 一直不大 → 衰减不明显

> 直观：这个高 16 bit 就像一个“上次打卡时间”，太久没打卡，就把“热度值”往下掉一档。



Step 2：再做“频次递增”

这时才轮到低 8 bit 的 counter：

- 并不是每次访问都 counter++

  否则很容易一下子冲到 255，然后大家都打满。

- 实际上是用了一个**概率性递增**：

  - counter 越大，再往上长一格的概率越低
  - 低频 key：几次访问就能明显涨
  - 高频 key：再怎么访问，counter 增长很慢，防“高频对象垄断池子”

 

淘汰的时候怎么用这个字段？

当触发内存淘汰（allkeys-lfu/volatile-lfu）时：

1. Redis 按配置采样一批 key（比如 5～10 个）
2. 对这批 key，看它们的 counter（低 8 bit）
3. **counter 越小、越冷门的 key 优先被淘汰**

衰减+采样组合的效果：

- 高频访问的 key 因为 counter 高，很难被选中
- 一段时间不用的 key，counter 被时间衰减 + 样本中偏低，很快被淘汰掉



###### 3.4 refcount：引用计数

C 没有 GC，所以 Redis 自己搞了一套引用计数： 

- 新创建对象：refcount = 1
- 这个对象被别的结构再引用一次 → refcount++
- 删除引用 / 用完：refcount--
- 当 refcount == 0：
  - 释放 redisObject 本身
  - 再释放 ptr 指向的真实数据结构（比如 sds/quicklist/skiplist 等）

**为什么要引用计数？**

1. **共享对象**

   Redis 会预先创建一些“共享对象”，比如小整数 0 ~ 9999、固定字符串 "OK"、"QUEUED" 等，然后所有用到这些值的地方都指向这一份对象，refcount++ 即可，省内存+省分配开销。 

2. **多地方持有同一个 value**

   某些场景可能一个对象被多个结构引用（例如复制、事务队列等），需要知道什么时候可以统一 free。

你可以用 OBJECT REFCOUNT key 看某个 key 对应对象当前 refcount，主要是调试用途。 



##### **4. 命令执行时，redisObject 是如何被使用的？**

典型流程可以想象成这样（简化）：

1. 根据 key，在 DB dict 里查到 redisObject*（找不到就返回 NULL）
2. 检查 o->type 是否符合命令要求（比如必须是 OBJ_LIST、OBJ_STRING）
3. 根据 o->encoding 选择对底层结构的操作函数：
   - encoding == QUICKLIST → 调 quicklist API
   - encoding == LISTPACK → 调 listpack API
4. 更新 lru / lfu 字段（访问时间或频率）
5. 根据需要对 refcount 做加减：
   - 比如把对象放进某个队列时会临时 incrRefCount
   - 用完后 decrRefCount，可能触发释放

> 也就是说：**redisObject 是所有值操作的入口：类型检查、底层分派、淘汰统计、生命周期都挂在这一个小结构上**。 





##### **5. 面试/复述用的一段话（你可以记这个）**

你可以对面试官这样说：

> 在 Redis 里，每个 value 实际上都是一个 redisObject，它是一个通用对象头，内部封装了：

- > type：值的逻辑类型，比如 string/list/hash/zset/stream；

- > encoding：该类型用哪种底层结构实现，比如 string 用 raw/int/embstr，list 用 quicklist，hash 用 listpack/hashtable 等；

- > lru：根据当前的内存淘汰策略，存 LRU 时间戳或者 LFU 的频率+时间，用来做 LRU/LFU 淘汰采样；

- > refcount：引用计数，用于管理共享对象和生命周期，计数为 0 时释放对象和底层数据。

> DB 里面其实是 dict<sds, redisObject*>，执行命令时先查到 redisObject，通过 type 做类型检查，再根据 encoding 调不同的数据结构实现，同时更新 lru/lfu 和 refcount，整个对象系统就靠这四个字段把“类型抽象、内存优化和淘汰策略”串起来了。



#### SDS（Simple Dynamic String）

**SDS（Simple Dynamic String）** 是 Redis 基于 C 语言实现的一套 **动态字符串抽象**，用于替代原生 C 字符串（char*）。

它在保持对大多数 C API 兼容的前提下，解决了 C 字符串在性能和安全上的诸多问题，更适合 Redis 这类高频字符串读写的场景。



##### **内存布局与结构**

典型的 SDS 头部结构（不同版本有 sdshdr8/16/32/64 等变体，但思路一致）：

```
struct sdshdr {
    uint32_t len;   // 当前已使用长度
    uint32_t alloc; // 已分配总容量（不含结尾 '\0'）
    char buf[];     // 数据区，末尾仍然预留一个 '\0' 以兼容 C API
};
```

内存布局示意：

```
[ sdshdr(len, alloc) ][ buf ... ][ '\0' ]
                        ^
                        |
                     返回给外部的 char*
```

特点：

- 对外看起来依然是 char*，可以直接传给很多 C 标准库函数。
- 对内通过 len/alloc 元数据管理字符串长度与容量。





##### **相比 C 字符串的主要改进**

###### **O(1) 获取长度**

- C 字符串：strlen(s) 需要从头扫描到 '\0'，复杂度 O(N)。
- SDS：长度保存在 len 字段，获取长度是 O(1)。

在 Redis 中，字符串长度被频繁使用（协议编码、命令处理、AOF 持久化等），O(1) 长度带来明显性能收益。



###### **二进制安全（Binary Safe）**

- C 字符串以 '\0' 作为结束标志，中间不能安全地出现 '\0'。
- SDS 只依赖 len 字段，不依赖内容中的任何字节：
  - 可以安全存储任意二进制数据（压缩结果、图片片段、序列化数据、protobuf 等）。
  - RESP 协议中的 Bulk String 也可以直接映射到 SDS。



###### **安全的动态扩容策略**

SDS 在追加数据时会自动检查是否有足够空间：

- 如果 alloc - len 不足，会自动执行扩容（sdsMakeRoomFor 等）。
- 扩容不是只扩到刚好够用，而是带有 **预分配策略**：
  - 小字符串：一般按 2 倍容量扩容，避免频繁 realloc。
  - 大字符串：按固定增量扩容，控制内存浪费。



优点：

- 减少 malloc/realloc 调用和内存拷贝次数。
- 字符串拼接、追加操作更高效，更不易出现性能抖动。





###### **减少缓冲区溢出风险**

- C 字符串追加/拷贝时，调用方需要自己确保目标缓冲区足够大，容易写出越界。

- SDS 将 **长度和容量管理封装在内部**：

  - 所有写入操作统一通过 SDS API，内部负责检查并扩容。
  - 显著降低缓冲区溢出、内存破坏等典型 C 语言问题。

  

###### **支持惰性缩容（lazy free）与空间重用**

- SDS 支持释放多余空间、或保留一定冗余空间供下次追加直接复用。
- 在频繁修改字符串（例如拼接日志、构造协议报文）场景下，可以减少反复分配/释放带来的开销。



##### **SDS 在 Redis 中的作用**

- Redis 中所有字符串相关的地方，**底层基本都用 SDS 表示**，包括：
  - key、string 类型的 value；
  - hash / list / zset / stream 等复合结构中的字符串成员；
  - 客户端输入/输出缓冲区；
  - AOF/复制/协议解析等内部缓冲。
- 在此基础上，Redis 又构建了更复杂的结构（如 embstr、quicklist、listpack 等），但**字符串的核心抽象依旧是 SDS**。



可以将 SDS 概括为：

> 一种基于 C 实现的、携带长度与容量元数据的二进制安全动态字符串抽象。

> 它提供 O(1) 获取长度、自动扩容、二进制安全和更好的内存安全性，是 Redis 内部几乎所有字符串数据的基础表示形式，相比原生 C 字符串更适合高性能、高可靠的服务端开发场景。



### 底层数据结构编码

#### 1）ziplist（老东西，已被淘汰）

**是什么：**

一块**连续内存**上的“压缩列表”，里面顺序存很多元素（string 或 int），早期用于：

- 小 hash
- 小 zset
- list 的老版本实现

**大致布局：**

```
| zlbytes | zltail | zllen | entry1 | entry2 | ... | zlend(0xFF) |
```

- zlbytes：整个 ziplist 长度

- zltail：最后一个 entry 的偏移

- zllen：元素个数（>65535 时不再精确）

- 每个 entry：prevlen + encoding+len + data

  - prevlen 是前一个 entry 的长度，用来支持从尾部往前遍历

  

**问题：**

- 因为每个 entry 里要存 prevlen，当前一个 entry 变长/变短时，会导致**连锁更新**（后面所有 prevlen 都要挪），最坏 O(N²)。
- 结构复杂，容易出 bug。

=> 后来被 **listpack** 替代。





#### 2）listpack（ziplist 的升级版）

**是什么：**

还是一块**连续内存**上的压缩列表，但设计更简单、避免连锁更新，用于：

- 小 hash
- 小 zset
- stream 内部的 entries
- quicklist 节点的内部存储（新版本）



**大致布局：**

```
| total_bytes | num_elements | entry1 | entry2 | ... | end(0xFF) |
```

- total_bytes：整个 listpack 大小

- num_elements：元素个数

- 每个 entry：encoding+len + data

  - 不再保留 prevlen 这种东西

  

**区别（相对 ziplist）：**

- 移除了 prevlen，不再支持高效从尾向前遍历，但：
  - 省空间
  - **没有连锁更新**：改一个 entry 不会影响后面一串
- encoding 更统一，可压缩 int / string，并用变长编码压缩长度和值。

**总结一句：**

都是“连续内存上的小数组”，**ziplist = 老的、容易连锁移动；listpack = 新的、更简洁、更稳定**。





#### **3）intset（小整数集合）**

**是什么：**

专门给 set 类型用的“小而全是整数”的编码。

**底层结构：**

```c
typedef struct intset {
    uint32_t encoding; // 元素使用何种整数宽度：INT16/INT32/INT64
    uint32_t length;   // 元素个数
    int8_t   contents[]; // 紧凑排列的一段有序整数数组
} intset;
```

特点：

- contents 里是**有序数组**（按数值排序），查找用**二分查找**。
- 元素全是相同位宽：
  - 初始可能是 int16
  - 插入一个更大的数时，整个 intset“升级”为 int32/int64，批量转换。

适用场景：

- set 里元素个数不多
- 且全是整数

一旦：

- 元素数超过阈值（set-max-intset-entries）
- 或插入了非整数

就会自动升级成 **hashtable** 编码。





#### 4）hashtable（dict）

**是什么：**

Redis 里最通用的哈希表实现，底层叫 dict，用于：

- 大 hash 的 field→value
- 一般的 set 的元素→NULL
- zset 的 member→score 映射的一部分
- 整个 DB 的 keyspace（db->dict）



**底层结构（简化）：**

```c
typedef struct dictEntry {
    void *key;
    void *val;           // 或者另外一个 union 指针
    struct dictEntry *next;
} dictEntry;

typedef struct dictht {
    dictEntry **table;   // 桶数组
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;  // 当前 entry 数量
} dictht;

typedef struct dict {
    dictht ht[2];        // 增量 rehash 用两个表
    long rehashidx;      // = -1 不在 rehash，>=0 则在挪桶
} dict;
```

特点：

- **开放地址 + 链地址**的混合：每个桶是一个链表（dictEntry*）。

- 通过 ht[0] 和 ht[1] 实现**渐进式 rehash**：

  - 需要扩容/缩容时，只分配新表，不一次性挪全部
  - 每次操作顺便挪一点，避免长时间卡顿

  



#### 5）skiplist + dict（ZSet 大数据量实现）

用于 **ZSet 的“大编码”**：

```
typedef struct zset {
    dict *dict;          // member -> score
    zskiplist *zsl;      // 按 score 排序的跳表
} zset;
```



##### skiplist（跳表）

跳表核心特点：

- 有很多“层”：
  - 最底层是全链表，所有节点
  - 越往上层节点越稀疏
- 查找时，从上层开始向右走，遇到大了就往下层跳 → 平均 O(logN)

在 Redis 里：

- 节点按 (score, member) 排序：
  - 主关键字：score
  - 次关键字：member（同分数情况下保证顺序）

好处：

- **范围查询**（按分数区间、按 rank 排名）很方便：
  - 可以从某个节点开始往后扫一段。
- 插入/删除复杂度 O(logN)，比单纯链表靠谱得多。



##### dict

- 只负责 member -> score 快速查：

  - 对应命令：ZSCORE、ZREM、ZINCRBY 等。

- dict + skiplist 组合起来：

  - 精确查找 O(1)
  - 范围遍历 O(logN + k)

  



#### 6）quicklist（list 的现代实现）

**是什么：**

list 类型现在基本都是 quicklist，设计目标：

- 保留 list 两端 LPUSH/RPUSH/LPOP/RPOP 的 O(1) 特性
- 同时比单链表更省内存、更 cache 友好

**底层结构：**

```
typedef struct quicklistNode {
    struct quicklistNode *prev, *next;
    unsigned char *entry;   // 指向一个压缩块：ziplist 或 listpack
    size_t sz;              // 该块的字节数
    unsigned int count;     // 该块包含的元素个数
    ...                     // 压缩标志、编码类型等
} quicklistNode;

typedef struct quicklist {
    quicklistNode *head, *tail;
    unsigned long count;    // 总元素数
    unsigned long len;      // 节点个数（quicklistNode 数）
    ...                     // 填充策略、压缩参数等
} quicklist;
```

理解为：

```
quicklist（双向链表）
  ├─ node1: [elem1, elem2, elem3 ...]   // 一个 ziplist/listpack
  ├─ node2: [elemX, ...]
  └─ node3: [...]
```

特点：

- **外层**是双向链表：方便两端 push/pop。

- **内层**的每个 node 是一个压缩数组（ziplist/listpack）：

  多个元素挤在一块连续内存里：

  - 减少 per-entry 的指针开销
  - 更省内存、更 cache-friendly

- 可以配置每个 node 的最大元素数/最大字节数（避免单块太大）。



**对比纯 linkedlist：**

- linkedlist：每个元素一个 listNode + pointer，内存碎片多、指针开销大。
- quicklist：多个元素打包到一个 node 的压缩块里，指针数量减很多，整体更紧凑。





7）整体区别小结（帮你记忆）

你可以这样在脑子里归类：

- **“一整块连续内存的压缩列表”**

  - ziplist：老、复杂、有连锁更新

  - listpack：新、简化版，不存 prevlen，避免连锁更新

    → 用于小 hash、小 zset、stream、quicklist 节点内部

- **“小整数专用结构”**

  - intset：有序整型数组，支持升级到更大位宽，set 的小整数集合优化

- **“通用哈希表”**

  - hashtable(dict)：key→val 映射，支持增量 rehash，hash、set、zset 内部都用它

- **“排序 + 范围查询结构”**

  - skiplist + dict：ZSet 大数据量使用
    - dict：member→score 精确查
    - skiplist：按 score 有序，做范围查/排名

- **“双向链表 + 小块压缩数组”**

  - quicklist：list 的现代实现，两端 O(1) + 内存紧凑



### Redis 主流程

> **Redis 的主流程本质上就是一个单线程事件循环：**启动时先完成配置加载、初始化全局状态（server、db、RDB/AOF、复制信息等），然后进入 for (;;) 的“主循环”。每一轮循环中，Redis 通过 I/O 多路复用（epoll/kqueue/select）等待就绪事件：有新连接就 accept 成为客户端、有可读事件就把 socket 中的数据读入输入缓冲，按 RESP 协议解析成“命令 + 参数”，在命令表中查到对应的实现函数；随后在 **单线程** 上顺序执行命令，对应地读写内存中的数据结构（dict、quicklist、listpack、skiplist 等），更新 redisObject、过期键、键空间通知等，同时将写操作追加到 AOF 缓冲、触发复制流量发送给从库；最后把命令的返回结果写入客户端输出缓冲，由事件循环在 socket 可写时一次性刷回给客户端。整个服务器就是这样：**I/O 多路复用把所有连接上的事件喂进来，单线程按命令表执行业务逻辑，周期性地在主循环里顺带处理定时任务（过期检查、RDB/AOF、复制心跳等），形成一个高效而可预期的主流程。**

我按刚才那段话里提到的**几个核心角色**给你拆开讲一圈（都围绕“单线程事件循环”这个主心骨）：



#### 1. I/O 多路复用（epoll/kqueue/select）

- Redis 主线程不会傻等某一个 socket 的读/写，而是把**所有客户端连接 + 监听 socket**都挂到内核的多路复用接口上（Linux 下主要是 epoll）。

- 主循环里调用一次 aeApiPoll() → 底层就是 epoll_wait()：

  - 哪些连接有**新数据可读** / **可以写** / **有新连接到达**，内核一次性告诉它。

  

- 好处：

  - 单线程就可以同时管理成千上万连接，因为**几乎所有时间都堵在 epoll 里等事件**，真正的处理逻辑是短促的。

你可以想成：**epoll 是“门卫 + 对讲机”，所有客户端排队按门铃，门卫把“哪个人在敲门”一次告诉 Redis 主线程。**



#### **2. 客户端对象 & 输入/输出缓冲**

每个 TCP 连接在 Redis 里对应一个 client 结构：

- 里面有：

  - **读缓冲**：把 socket 里读出的字节先攒起来，按 RESP 解析成命令。
  - **写缓冲**：命令执行完的返回结果先写到这里，不一定立刻写 socket。

  

- 一轮循环里，流程大概是：

  - epoll 告诉某个 fd 可读 → 调 read/recv 填充 client->querybuf → 按 RESP 协议解析出一条条命令。
  - 执行命令后，把响应写到 client->buf 或 client->reply 链表里。
  - 下次 epoll 告诉这个 fd 可写时，再把缓冲内容刷回客户端。

> 这就是“**读是事件驱动、写是延迟发送**”：保证主线程不会被慢客户端 block 住。



#### **3. 命令表（command table）**

Redis 有一张**全局命令表**（一个哈希表），key 是命令名字，value 是描述结构体：

```
struct redisCommand {
    char *name;                // "get", "set", "hset" ...
    redisCommandProc *proc;    // 实际执行函数
    int arity;                 // 参数个数要求
    int flags;                 // 读/写、是否 key 写操作、是否阻塞等
    ...
};
```

解析出命令后：

1. 在命令表里查 "GET" 或 "SET" → 拿到对应 redisCommand
2. 做参数个数、读写标记之类检查
3. 调用 cmd->proc(client *)

> 所以 Redis 整体像一个“**大号 switch-case**”：命令表就是这张总路由表。



#### **4. redisObject + 底层数据结构**

命令执行时，真正操作的是：

- db->dict：当前数据库的 key 空间（dict<sds, redisObject*>）
- 每个 redisObject 里带 **type / encoding / ptr**：
  - type：string/list/hash/set/zset/stream…
  - encoding：raw/embstr/intset/quicklist/listpack/skiplist…
  - ptr：指向真正的底层结构（SDS、quicklist、listpack、dict、skiplist…）



命令函数内部的典型步骤：

1. 通过 key 在 dict 里查到 redisObject*
2. 校验 type 是否匹配当前命令
3. 根据 encoding 选择对哪种底层结构做操作（quicklist/listpack/dict/intset/skiplist 等）
4. 修改结束后，可能更新过期时间、键空间通知、AOF/复制流等

> 所以：**命令表负责“调哪段代码”，redisObject + 各种数据结构负责“数据怎么存、怎么改”。**



#### **5. 持久化：AOF / RDB**

在主流程里，持久化主要以**两种方式挂进去**：

1. **AOF（Append Only File）**

   - 每条写命令执行完后，会把这条命令以文本协议形式 append 到 AOF 缓冲。
   - 主循环周期性（或按策略）把 AOF 缓冲刷到磁盘。
   - AOF 重写（rewrite）通常由子进程异步完成，主线程只负责往旧 AOF 写增量。

   

2. **RDB（快照）**

   - SAVE/BGSAVE 或定时任务触发时：
     - BGSAVE 会 fork 一个子进程，子进程把当前内存数据 dump 成 RDB 文件。
   - 主线程继续按原流程处理命令，只在 fork 的那一瞬间有短暂停顿。

> 主循环里就是：**执行写命令 → 记日志（AOF）→ 偶尔触发 fork 做 RDB 重写**。



#### **6. 复制：主从数据同步**

作为主库时，每条写命令执行完：

- 会被写入“复制缓冲”（replication buffer）
- 异步地通过网络发给从库
  - 新从库会先 PSYNC 拉一份 RDB（全量），然后再拉增量命令。
- 主循环里有定期的复制心跳：
  - PING、REPLCONF ACK，用来检测连接健康，推进复制偏移量。

> 复制对主线程来说：**写命令执行完，多做一步“往复制缓冲里 append”，剩下由异步 I/O 给各个从库推过去。**



#### **7. 定时任务（serverCron 等）**

主循环除了处理 socket 事件，还会周期性跑一些“后台活”：

- 检查 & 删除过期键（随机采样，不是一次扫完）
- 更新内存使用统计、采样做内存淘汰
- 检查是否该触发 RDB/AOF 重写
- 发送复制心跳、集群心跳
- 慢查询统计、延迟监控等

实现上就是一个定时事件 / 每 N ms 调一次 serverCron()，里面串了一堆“小任务”。

> 你可以理解成：**主循环 while(true)** 里，每转一圈除了“处理 I/O 事件”，顺手跑一批 house-keeping 的逻辑。



#### 一次循环处理多少请求？

> **Redis 一轮事件循环里，会把这次 epoll 返回的所有就绪“事件”都处理完**（新连接 + 可读 + 可写），

> 周期性任务（过期、渐进 rehash、RDB/AOF、复制等）是通过 serverCron 这个“定时事件”插在循环里的，并且这些任务内部都有“时间片/配额”控制，避免把一整秒都耗光。真正会把其它东西拖死的，是你自己发的**大 O(N) 命令**。



正常场景（命令很轻）

一般业务都是 GET/SET/INCR/HSET 这种小 O(1)/O(logN) 命令：

- 一轮 event loop，即使有几千个就绪 FD，每个命令执行时间都是几十纳秒量级。
- 这一圈跑完也就是几百微秒～几毫秒。
- 这样 aeProcessEvents 很快就结束，又会进入下一轮，processTimeEvents 能按 hz 非常规律地触发 serverCron。 

这时候“处理多少请求”对后台任务影响非常小。



异常场景：你自己发了巨大的 O(N) 命令

如果你在**单线程**的 Redis 里干这些事：

- KEYS *、FLUSHDB、SMEMBERS 超大集合、LRANGE 0 -1 超大列表、SORT 大 key，或者一次 DEL 百万成员的大 hash/zset……

那问题就来了：

- Redis 是 **单线程执行命令**，不会在命令中途“切走”去跑 serverCron。 
- 这一大条命令执行的整个时间段内，serverCron、响应其它客户端、处理复制 ACK 等都被**整体延后**。



官方 latency 文档也直接点名：**服务器端延迟的主要来源就是这种“长时间运行的命令”** 。

所以严格地讲：

> Redis 并没有一个“每轮最多只执行 X 条命令”的硬阈值，

> 真正的保护机制是：

- > 周期任务内部有 time limit（25% 周期等）；

- > 写 socket 有 NET_MAX_WRITES_PER_EVENT 限制；

- > 你自己不要发超大的 O(N) 命令。

> Redis 使用单线程 + I/O 多路复用的事件循环。每轮循环中，aeApiPoll（epoll/kqueue…）返回本轮所有就绪的 file events，Redis 会把这些事件全部处理完：接收新连接、从可读 socket 读命令、向可写 socket 写回复。不过单个客户端写数据时受 NET_MAX_WRITES_PER_EVENT（默认 64KB）限制，避免一个大响应长时间占用循环。

> 周期性任务（过期扫描、渐进 rehash、主从复制巡检、RDB/AOF 子进程状态检查等）通过 serverCron 以 time event 的形式插入事件循环，频率由 hz 控制（默认 10 次/秒）。serverCron 内部对每类任务都设置了严格的时间片上限（例如过期扫描最多占用 serverCron 周期 25% 的 CPU 时间），以防周期任务阻塞主循环。

> 因此，一轮循环中“要处理多少请求”并没有直接的计数上限，Redis 的策略是：**file events 全部处理、周期任务靠时间片限流**。真正能拖垮其它任务的，是少数开发者自己发送的长耗时 O(N) 命令，而不是普通的小请求并发。



### Bitmap：在一个字符串上按 bit 存标记

**本质**：

Bitmap 其实就是普通 String，只是我们把它当成“位数组”用——一个 bit 存一个开关状态。

典型命令：

```
SETBIT key offset value
GETBIT key offset
BITCOUNT key [start end
```

- SETBIT key offset value：把第 offset 位设成 0/1
  - offset 是 bit 下标，从 0 开始
  - Redis 会自动扩容这个 String（中间缺的字节补 0）
- GETBIT key offset：读这一位是 0 还是 1
- BITCOUNT key [start end]：统计某个范围 byte 里的 bit=1 个数（常用来算活跃人数）



**典型用法：**

- **每日打卡 / 每日活跃**：

  - 用用户 ID 当 offset：SETBIT active:2025-11-13 <uid> 1
  - 日活：BITCOUNT active:2025-11-13

  

- **布尔标记集合**：

  - 某功能是否开启、某人是否完成任务等

**好处：**

- 极其省内存：1 bit 一个用户，1 亿用户也就十几 MB。
- 操作 O(1) 或 O(N/8)，适合极大规模的“布尔集合”。





### Bitfield：在一个值里读写多段 bit 域

Bitmap 是“一位一位管理”；**Bitfield** 是在一个 String 里**定义多个小字段**，每段字段可以看成有符号/无符号整数，然后做读写/自增等操作。

命令结构比较特殊：

```
BITFIELD key
  [GET type offset]
  [SET type offset value]
  [INCRBY type offset increment]
  [OVERFLOW WRAP|SAT|FAIL]
```

例子：

```
# 把 key 这个字符串看成一块 bit 空间
# 在 offset=0 处存一个无符号 8 位整数（0~255）
BITFIELD user:stat BFSET u8 0 5
BITFIELD user:stat BFGET u8 0           # -> 5

# 在 offset=8 处再存一个有符号 8 位整数
BITFIELD user:stat BFSET i8 8 -3
BITFIELD user:stat BFGET i8 8           # -> -3
```

（新版写法是 BFSET/BFGET/BFINCRBY，老版是 SET/GET/INCRBY，语义类似。）

**适用场景：**

- 希望用**一个 key 压缩存多种小指标**：如：
  - 某用户 8 种开关配置（每个占 1 bit）
  - “等级(0~255)、经验值、小标志位”打包进一个 String
- 想使用 bit 级自增、读取，而不是只做布尔标记。

**和 Bitmap 的区别：**

- Bitmap：主要按 bit 操作，**每位只表示 0/1**；
- Bitfield：把一段 bit 当作“小整数”，支持**多位组成字段 + 自增 + 溢出策略**。





### HyperLogLog（HLL）：基数估算（去重计数）

**解决的问题**：

“这个集合里大概有多少不重复的元素？”——类似：

- 网站一共有多少 UV？
- 某活动有多少独立用户参与？

如果用 set 存，每个人一个元素，数据量大了非常占内存；

HyperLogLog 用 **极少内存（固定 12 KB 左右）** 做**近似去重计数**，误差率约 1%。

命令：

```
PFADD key element [element ...]   # 添加元素（只用于影响计数）
PFCOUNT key [key ...]             # 估算基数（可以合并多个 HLL）
PFMERGE destkey sourcekey ...     # 合并多个 HLL
```

例子：

```
PFADD uv:2025-11-13 user1 user2 user3
PFADD uv:2025-11-13 user2 user4
PFCOUNT uv:2025-11-13             # -> 4（大概率）
```

内部原理（一句话版）：

> 把元素 hash 成一个很长的比特串，根据前缀决定落到哪个“桶”，根据后缀的前导零数估计这个桶里的“最大前导零”，再根据多个桶的统计量推导整体基数。

**适用场景：**

- 非常在乎内存，允许 1% 左右的误差：
  - UV、去重用户数、去重 IP 数、去重设备数等。
- 不关心具体有哪些元素，只关心“有多少个”。





### GEO：存地理坐标 & 查询附近点位

**本质**：

GEO 在 Redis 里也是基于 ZSet 实现的：

- member：地点名称（比如用户 ID、门店 ID）
- score：一个通过 lng/lat 编码出来的 52-bit geohash 值

命令（新版推荐用 GEOSEARCH 系列）：

```
GEOADD key longitude latitude member [longitude latitude member ...]
GEOSEARCH key FROMMEMBER member BYRADIUS radius unit [ASC|DESC] [COUNT n]
GEOSEARCH key FROMLONLAT lon lat BYBOX width height unit ...
GEOPOS key member [member ...]        # 取坐标
GEODIST key member1 member2 [unit]    # 计算两点距离
```

简单例子：

```
GEOADD shop 116.3974 39.9092 "天安门"
GEOADD shop 116.3846 39.9255 "景山"
GEOSEARCH shop FROMMEMBER "天安门" BYRADIUS 2 km ASC
# 返回 2km 以内的店，按距离排序
```

内部大致玩法：

1. lng/lat → 编码成一个 geohash（支持范围查询的数值）
2. 存到 ZSet 的 score 里
3. 邻近查询时：
   - 先根据圆/矩形边界推一批 geohash 范围
   - 在这些范围上做 ZRANGE/ZRANGEBYSCORE 类扫描
   - 再精确计算距离，过滤掉边界误差

**适用场景：**

- “附近的人 / 店 / 车”；
- LBS 场景：找某点半径/矩形内的所有对象；
- 需要按距离排序＋分页。





可以怎么一句话总结给自己/面试官

你可以这样“打包”这四个东西：

> Redis 在 String 的基础上扩展了几类特殊玩法：

- > **Bitmap**：把 String 当 bit 数组用，SETBIT/GETBIT/BITCOUNT，可以极省内存做打卡、活跃标记等布尔集合；

- > **Bitfield**：在同一个字符串里按位定义多个整数字段，用 BITFIELD 做精细的 bit 级读写和自增；

- > **HyperLogLog**：用固定小内存做基数估算，通过 PFADD/PFCOUNT 统计去重数量，适合 UV 等不追求精确值的场景；

- > **GEO**：基于 ZSet+geohash 存经纬度，用 GEOADD/GEOSEARCH 做附近查询和距离排序，适合 LBS “附近的人/店/车”。